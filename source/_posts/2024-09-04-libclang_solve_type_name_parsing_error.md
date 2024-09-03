---
title: 'C++ libclang 解决变量类型解析失败为 int 的问题'
date: 2024-09-04 01:39:37
tags:
  - Cpp
  - Reflection
  - Libclang
---

对于简单的类型声明，libclang 是可以成功解析的。但是对于一些复杂的类型声明，libclang 可能会解析失败。

比如下面的代码中，`m_image_paths` 在 `TestRefl` 中可以被解析出类型为 `std::vector<std::string>`，但是在 `ModelComponent` 中就被解析为 `int`

```cpp
#pragma once

#include "core/reflect/macros.h"

#include <string>
#include <vector>

class [[reflectable_class()]] TestRefl
{
public:
    [[reflectable_field()]]
    std::string test_str;

    [[reflectable_field()]]
    std::vector<std::string> m_image_paths;
};
```

```cpp
#pragma once

#include "core/base/bitmask.hpp"
#include "function/object/game_object.h"
#include "function/render/structs/model.h"

#include <string>
#include <vector>

class [[reflectable_class()]] ModelComponent : public Component
{
public:
    UUIDv4::UUID         uuid;
    std::weak_ptr<Model> model_ptr;

    [[reflectable_field()]]
    std::string test_str;

    [[reflectable_field()]]
    std::vector<std::string> m_image_paths;

    ModelComponent(std::vector<float>&&        vertices,
                    std::vector<uint32_t>&&     indices,
                    BitMask<VertexAttributeBit> attributes);

    ModelComponent(const std::string& file_path, BitMask<VertexAttributeBit> attributes);
};
```

主要原因并不是被解析的类型本身有多么复杂，而是复杂的类型会 include 很多头文件，这些头文件的根目录可能并没有传到 `clang_parseTranslationUnit` 里面，即使是传入了，也可能因为缺少一些宏定义之类的问题导致报错

我的方法是，用于生成代码的 exe target 直接 include 所有主程序 target include 的第三方库，然后在 cmake 中获取这些依赖，存成字符串，例如下面代码中的 `${INCLUDE_PATH_COLLECTION}`

```cmake
function(get_target_include_directories TARGET VAR_NAME)  
    set(INCLUDE_DIRS "")  
    get_target_property(TMP_DIRS ${TARGET} INCLUDE_DIRECTORIES)    
    foreach(DIR ${TMP_DIRS})  
        # If DIR is a generator expression, there will be no expansion here
        # Here we assume they are direct paths 
        list(APPEND INCLUDE_DIRS "-I${DIR}")  
    endforeach()   
    set(${VAR_NAME} "${INCLUDE_DIRS}" PARENT_SCOPE)  
endfunction()  

get_target_include_directories(${CODE_GENERATOR_NAME} INCLUDE_PATH_COLLECTION)  
```

然后再把这个获取到的字符串传入用于生成代码的 exe

```cmake
add_custom_command(
    OUTPUT ${SRC_ROOT_DIR}/runtime/generated/register_all.cpp
    COMMAND ${CODE_GENERATOR_NAME} ${INCLUDE_PATH_COLLECTION} "-S${SRC_ROOT_DIR}/runtime" "-O${SRC_ROOT_DIR}/runtime/generated"
    DEPENDS ${HEADER_FILES_DEPEND}
    COMMENT "Generating register_all.cpp"
)
```

exe 接受了命令行参数，存成 vector，这个比较简单可以跳过

最终解析的时候要传入这些参数

```cpp
void Parser::ParseFile(const fs::path& path, const std::vector<std::string>& include_paths)
{
    // traverse AST to find class

    CXIndex index = clang_createIndex(0, 0);

    std::vector<const char*> all_args(3 + include_paths.size());
    all_args[0] = "-xc++";
    all_args[1] = "-std=c++20";
    all_args[2] = "-DGLM_ENABLE_EXPERIMENTAL";
    for (int i = 0; i < include_paths.size(); i++)
    {
        all_args[i + 3] = include_paths[i].c_str();
    }
    CXTranslationUnit unit = clang_parseTranslationUnit(
        index, path.string().c_str(), all_args.data(), all_args.size(), nullptr, 0, CXTranslationUnit_None);

    ...
}
```

其中要注意传入的参数的数量要正确，我就犯过这样的错误

不过用 `vector` 来存参数就更省心，直接取 `.size()`

当然，这可能并不足以消除错误

为了查看 libclang 解析单元时发生了什么错误，在 cpp 中就可以捕捉到

```cpp
void LibclangUtils::print_diagnostics(CXTranslationUnit TU)
{
    unsigned numDiagnostics = clang_getNumDiagnostics(TU);
    for (unsigned i = 0; i < numDiagnostics; ++i)
    {
        CXDiagnostic diag    = clang_getDiagnostic(TU, i);
        CXString     diagStr = clang_formatDiagnostic(diag, clang_defaultDiagnosticDisplayOptions());
        printf("Diagnostic %u: %s\n", i, clang_getCString(diagStr));
        clang_disposeString(diagStr);
        clang_disposeDiagnostic(diag);
    }
}
```

在解析单元之后输出即可

```cpp
CXTranslationUnit unit = clang_parseTranslationUnit(
    index, path.string().c_str(), all_args.data(), all_args.size(), nullptr, 0, CXTranslationUnit_None);
if (unit == nullptr)
{
    std::cerr << "Unable to parse translation unit. Quitting." << std::endl;
    exit(-1);
}

LibclangUtils::print_diagnostics(unit);
```

比如根据这个 diagnostics，报错是找不到 assimp 的 `config.h`

```
fatal error: 'assimp/config.h' file not found
```

原因是这个文件是 assimp 构建时生成的。所以我简单地 include assimp 的 include 文件夹时没有用的

要做的是 include 编译目标中的 assimp 源码的编译结果

```cmake
target_include_directories(${CODE_GENERATOR_NAME} PUBLIC ${CMAKE_BINARY_DIR}/src/3rdparty/assimp/include)
```

于是解决了我这份头文件的问题