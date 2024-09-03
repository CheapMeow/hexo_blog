---
title: 'OpenGL 和 Vulkan 的投影矩阵'
date: 2024-09-02 20:32:09
tags:
  - Grphics
  - Vulkan
  - OpenGL
---

Vulkan 只是屏幕坐标系和别人不一样。世界空间，view 空间的手性是用户定义的，想怎么做都行，只要最终保证传入 Vulkan 的 NDC 坐标的手性是对的。也就是说，你可以在计算投影矩阵的时候，注意要转换手性。

如果没有推导过投影矩阵，就可能不会理解到世界空间，view 空间的手性和 NDC 的手性可以毫无关系——毕竟你不知道有矩阵可以转换手性。

## 推导透视矩阵时可能遇到的困难

### 物理过程

首先要知道变换过程是怎么样的

<div style="overflow-x: scroll">
<div style="width: 1800px">
<img src="/images/perspective_matrix_for_opengl_and_vulkan/opengl_pipeline.drawio.svg"></img>
</div>
</div>

### 物理量的定义

不同的文章对某一个物理量的定义可能不同，但是他们又使用了相同的符号，结果就导致最终的表达式可能差了一个负号

比如 games101 的推导中，视图空间是右手系，近平面的坐标为 n，远平面的坐标为 f。这说明 n,f 都是负数

但是在 LearnOpenGL 推荐的贴子 [https://www.songho.ca/opengl/gl_projectionmatrix.html](https://www.songho.ca/opengl/gl_projectionmatrix.html) 中，他视图空间是右手系，但定义 n,f 为正，那么近平面的坐标为 -n，远平面的坐标为 -f

### 深度的约定

创建 `vk::PipelineDepthStencilStateCreateInfo` 时，我们一般会传入 `vk::CompareOp::eLessOrEqual` 到 `depthCompareOp`，表示传入的深度比存储的深度更小时，就通过了测试，写入新的深度。这就表明近处的物体会遮挡远处的物体，符合了 z 的正值越大表示深度越深的习惯。

用户可以定义用 z 的正值越大表示深度越深，或者是负值越大表示深度越深，也就是说，用户可以定义用 z 轴垂于屏幕朝内或者朝外来表示深度的正半轴

理论上来说，你不知道 NDC 的 z 轴朝向，你单单讨论屏幕空间的 x 和 y 之间的关系，你是没办法说 NDC 空间的手性的手性如何

于是当我们看到某些帖子在说 Vulkan 与 OpenGL 的区别在于 y 轴反了，那是默认 z 轴垂于屏幕朝内表示深度的正半轴

### NDC 深度范围与 z 轴反转

某些细节会影响结果，但是一般的教程不会强调这些细节，因为他们默认你都知道。比如假设近平面的坐标为 n，远平面的坐标为 f 都是负数，那么他要转换到 [-1, 1]，是 n 对应 -1 还是 f 对应 -1？或者是转换到 [0, 1]？都有可能。

### 变换到 clip 空间中的坐标的齐次坐标 w 的符号

在推导挤压平截头体的矩阵的时候，齐次坐标的位置可以用 z 或者 -z，都不影响结果，但是结果矩阵会所有元素差一个负号。最终因为要做透视除法，所以每个元素多出来的一个负号和 w 的负号相抵，所以不会导致变换到裁剪空间的结果不同。

但是这会导致公式中的符号不同，所以可能令人困惑

### 为什么 OpenGL 中相机在 eye 空间中看向 -z 轴

OpenGL 推导透视矩阵时，相机在 eye 空间中是看向 -z 轴，所以视锥体的近平面坐标和远平面坐标都是负数，这是推导透视矩阵公式的基础。那么为什么是看向 -z 轴而不是 +z 轴呢？

glm 默认的 `lookAt` 调用的是 `glm::lookAtRH`

OpenGL 中的构建 view 矩阵的堆栈（来自 [https://learnopengl.com/](https://learnopengl.com/)）

```cpp
glm::mat4 view = camera.GetViewMatrix();
```

```cpp
glm::mat4 GetViewMatrix()
{
    return glm::lookAt(Position, Position + Front, Up);
}
```

```cpp
void updateCameraVectors()
{
    // calculate the new Front vector
    glm::vec3 front;
    front.x = cos(glm::radians(Yaw)) * cos(glm::radians(Pitch));
    front.y = sin(glm::radians(Pitch));
    front.z = sin(glm::radians(Yaw)) * cos(glm::radians(Pitch));
    Front = glm::normalize(front);
    // also re-calculate the Right and Up vector
    Right = glm::normalize(glm::cross(Front, WorldUp));  // normalize the vectors, because their length gets closer to 0 the more you look up or down which results in slower movement.
    Up    = glm::normalize(glm::cross(Right, Front));
}
```

于是我们知道了，传入 `lookAtRH` 的期望是摄像机看向 `Front` 的指向

但是为什么反而在 eye 空间中却看向 -z 轴呢？按道理来说，乘以 view 矩阵之后，整个世界都被转到相机面向物体的坐标系中了呀？现在你推导透视矩阵时反而认为相机背向物体？

所以这个事情还是要看 `lookAtRH` 是怎么做的

```cpp
template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER mat<4, 4, T, Q> lookAtRH(vec<3, T, Q> const& eye, vec<3, T, Q> const& center, vec<3, T, Q> const& up)
{
  vec<3, T, Q> const f(normalize(center - eye));
  vec<3, T, Q> const s(normalize(cross(f, up)));
  vec<3, T, Q> const u(cross(s, f));

  mat<4, 4, T, Q> Result(1);
  Result[0][0] = s.x;
  Result[1][0] = s.y;
  Result[2][0] = s.z;
  Result[0][1] = u.x;
  Result[1][1] = u.y;
  Result[2][1] = u.z;
  Result[0][2] =-f.x;
  Result[1][2] =-f.y;
  Result[2][2] =-f.z;
  Result[3][0] =-dot(s, eye);
  Result[3][1] =-dot(u, eye);
  Result[3][2] = dot(f, eye);
  return Result;
}
```

可以看出，其实返回的 view 空间的 z 轴是 -front

但是明明 front 是指向物体的，所以 view 变换之后，摄像机就背向了物体，摄像机的 -z 指向物体

所以这就说明了 lookAt 矩阵表示的 z 轴他不一定是传入的 front

### lookAt 怎么与透视矩阵对应

我不止一次看到别人推荐这个网站 [https://www.songho.ca/opengl/gl_projectionmatrix.html](https://www.songho.ca/opengl/gl_projectionmatrix.html)，然后引用这句话

> Note that the eye coordinates are defined in the right-handed coordinate system, but NDC uses the left-handed coordinate system. That is, the camera at the origin is looking along -Z axis in eye space, but it is looking along +Z axis in NDC.

他的意思似乎是，因为 NDC 是左手系，view 空间（eye 空间）是右手系，所以 x 和 y 轴不变的话，就可以认为两者的 z 轴是反的。那么假设视锥体都在一个固定的 NDC 正 z 的地方，那么我在 NDC 中需要看向这个视锥体，所以我在 view 空间中才需要让我的摄像机看向 -z 方向而不是 +z

当然这是一个倒因为果……并不是因为透视矩阵是这样，视图矩阵才是这样

而是因为首先你知道了，经过 lookAt 之后，摄像机看向了 -z，所以透视矩阵中注意对 z 反转，使得 NDC 中摄像机看向 +z

网上大部分教程以及评论都默认你右手系并且用的是 `glm::lookAt`，这确保了摄像机在 eye 空间中却看向 -z 轴

原则上你用什么 lookAt 都可以，但是你对应的投影矩阵的公式也要跟着变

现在大家用的都是 glm 的 lookAt，所以我觉得还是入乡随俗

## 从 Games101 的公式开始

先假设世界空间，视图空间是右手系

视图空间中有一个正交长方体的左平面的坐标为 l，右平面的坐标为 r，上平面的坐标为 t，下平面的坐标为 b，近平面的坐标为 n，远平面的坐标为 f

现在希望转成一个 NDC 标准坐标

那么 games101 是

$M_{ortho}=\left(\begin{array}{cccc}
\frac{2}{r-l} & 0 & 0 & 0\\
0 & \frac{2}{t-b} & 0 & 0\\
0 & 0 & \frac{2}{n-f} & 0\\
0 & 0 & 0 & 1
\end{array}\right)\left(\begin{array}{cccc}
1 & 0 & 0 & -\frac{r+l}{2}\\
0 & 1 & 0 & -\frac{t+b}{2}\\
0 & 0 & 1 & -\frac{n+f}{2}\\
0 & 0 & 0 & 1
\end{array}\right)$

注意，这个公式把 [f, n] 转成 [-1, 1] 的，那么原来是 f 比 n 小，现在也是 -1 比 1 小，所以没有改变手性

frustum 的关系式是

$x' = n/z*x$

$y' = n/z*y$

因此设计一个 frustum 挤压成正交长方体的矩阵，使得变换出来的 $x,y,w$ 的部分与这个关系式对应。对应的方法就是使得齐次坐标 $w$ 的位置放关系式的分母，也就是 $z$

$M_{persp2ortho} = \left(\begin{array}{cccc}
n & 0 & 0 & 0\\
0 & n & 0 & 0\\
a_1 & a_2 & a_3 & a_4\\
0 & 0 & 1 & 0
\end{array}\right)$

$M_{persp2ortho}\left(\begin{array}{c}
x\\
y\\
z\\
1
\end{array}\right)=\left(\begin{array}{c}
nx\\
ny\\
?\\
z
\end{array}\right)$

为了求解未知数，有两个关系，一个是近平面上的点在压缩时不变，另一个是远平面的中心点在压缩时不变

单独看“近平面上的点在压缩时不变”，这使得

$M_{persp2ortho}\left(\begin{array}{c}
x\\
y\\
n\\
1
\end{array}\right)=\left(\begin{array}{c}
nx\\
ny\\
n^2\\
n
\end{array}\right)$

对任意 x,y 成立

那么可以证出 $a_1,a_2$ 都是 0，因为 $n^2$ 与 $x,y$ 无关

那么取近平面的中心点不变和远平面的中心点不变，得到两个式子

$M_{persp2ortho}\left(\begin{array}{c}
0\\
0\\
n\\
1
\end{array}\right)=\left(\begin{array}{c}
0\\
0\\
n^2\\
n
\end{array}\right)$

$M_{persp2ortho}\left(\begin{array}{c}
0\\
0\\
f\\
1
\end{array}\right)=\left(\begin{array}{c}
0\\
0\\
f^2\\
f
\end{array}\right)$

即

$a_3 n + a_4 = n^2$
$a_3 f + a_4 = f^2$

解得

$a_3 = n+f, a_4 = -nf$

最终结果

$M_{persp2ortho} = \left(\begin{array}{cccc}
n & 0 & 0 & 0\\
0 & n & 0 & 0\\
0 & 0 & f+n & -f\,n\\
0 & 0 & 1 & 0
\end{array}\right)$

两者相乘可以得到

$M_{proj} = M_{ortho} * M_{persp2ortho} = \left(\begin{array}{cccc}
\frac{2\,n}{r-l} & 0 & \frac{l+r}{l-r} & 0\\
0 & \frac{2\,n}{t-b} & \frac{b+t}{b-t} & 0\\
0 & 0 & -\frac{f+n}{f-n} & \frac{2\,f\,n}{f-n}\\
0 & 0 & 1 & 0
\end{array}\right)$

进行这个投影变换之后，原来是右手坐标的视图空间变为右手坐标的裁剪空间

## 考虑 OpenGL

OpenGL 的公式中要求 n,f 都是距离，所以都是正值

那么近平面是 -n，远平面是 -f

又要求变换之后是左手系，也就是 [-n, -f] 变换到 [-1, 1]

那么正交投影矩阵

$M_{ortho}=\left(\begin{array}{cccc}
\frac{2}{r-l} & 0 & 0 & 0\\
0 & \frac{2}{t-b} & 0 & 0\\
0 & 0 & \frac{2}{n-f} & 0\\
0 & 0 & 0 & 1
\end{array}\right)\left(\begin{array}{cccc}
1 & 0 & 0 & -\frac{r+l}{2}\\
0 & 1 & 0 & -\frac{t+b}{2}\\
0 & 0 & 1 & \frac{f+n}{2}\\
0 & 0 & 0 & 1
\end{array}\right)$

frustum 的关系式是

$x' = -n/z*x$
$y' = -n/z*y$

因为这里的 z 是负值，所以要加负号表示距离

设计压缩矩阵

$M_{persp2ortho} = \left(\begin{array}{cccc}
-n & 0 & 0 & 0\\
0 & -n & 0 & 0\\
0 & 0 & a_3 & a_4\\
0 & 0 & 1 & 0
\end{array}\right)$

$M_{persp2ortho}\left(\begin{array}{c}
x\\
y\\
z\\
1
\end{array}\right)=\left(\begin{array}{c}
-nx\\
-ny\\
?\\
z
\end{array}\right)$

那么取近平面的中心点不变和远平面的中心点不变，得到两个式子

$M_{persp2ortho}\left(\begin{array}{c}
0\\
0\\
-n\\
1
\end{array}\right)=\left(\begin{array}{c}
0\\
0\\
n^2\\
-n
\end{array}\right)$

$M_{persp2ortho}\left(\begin{array}{c}
0\\
0\\
-f\\
1
\end{array}\right)=\left(\begin{array}{c}
0\\
0\\
f^2\\
-f
\end{array}\right)$

即

$-a_3 n + a_4 = n^2$
$-a_3 f + a_4 = f^2$

解得

$a_3 = -(n+f), a_4 = -nf$

最终结果

$M_{persp2ortho} = \left(\begin{array}{cccc}
-n & 0 & 0 & 0\\
0 & -n & 0 & 0\\
0 & 0 & -(f+n) & -f\,n\\
0 & 0 & 1 & 0
\end{array}\right)$

两者相乘可以得到

$M_{proj} = M_{ortho} * M_{persp2ortho} = \left(\begin{array}{cccc}
\frac{2\,n}{l-r} & 0 & \frac{l+r}{l-r} & 0\\
0 & \frac{2\,n}{b-t} & \frac{b+t}{b-t} & 0\\
0 & 0 & \frac{f+n}{f-n} & \frac{2\,f\,n}{f-n}\\
0 & 0 & 1 & 0
\end{array}\right)$

但是还是和 OpenGL 的公式搭不上

$M_{proj} = M_{ortho} * M_{persp2ortho} = \left(\begin{array}{cccc}
\frac{2\,n}{r-l} & 0 & \frac{l+r}{r-l} & 0\\
0 & \frac{2\,n}{t-b} & \frac{b+t}{t-b} & 0\\
0 & 0 & -\frac{f+n}{f-n} & -\frac{2\,f\,n}{f-n}\\
0 & 0 & -1 & 0
\end{array}\right)$

观察发现我自己推出来的矩阵乘以 -1 就是 OpenGL 的公式

于是说……这两个矩阵的结果会是一样的吗

之后看了 [https://www.zhyingkun.com/perspective/perspective/](https://www.zhyingkun.com/perspective/perspective/)

才确认了别人也遇到了这个问题，并且他们会是一样的

重新推一下，frustum 的关系式是

$x' = n/(-z)*x$
$y' = n/(-z)*y$

设计压缩矩阵

$M_{persp2ortho} = \left(\begin{array}{cccc}
n & 0 & 0 & 0\\
0 & n & 0 & 0\\
0 & 0 & a_3 & a_4\\
0 & 0 & -1 & 0
\end{array}\right)$

$M_{persp2ortho}\left(\begin{array}{c}
x\\
y\\
z\\
1
\end{array}\right)=\left(\begin{array}{c}
nx\\
ny\\
?\\
-z
\end{array}\right)$

那么取近平面的中心点不变和远平面的中心点不变，得到两个式子

$M_{persp2ortho}\left(\begin{array}{c}
0\\
0\\
-n\\
1
\end{array}\right)=\left(\begin{array}{c}
0\\
0\\
-n^2\\
n
\end{array}\right)$

$M_{persp2ortho}\left(\begin{array}{c}
0\\
0\\
-f\\
1
\end{array}\right)=\left(\begin{array}{c}
0\\
0\\
-f^2\\
f
\end{array}\right)$

即

$-a_3 n + a_4 = -n^2$
$-a_3 f + a_4 = -f^2$

解得

$a_3 = n+f, a_4 = nf$

最终结果

$M_{persp2ortho} = \left(\begin{array}{cccc}
n & 0 & 0 & 0\\
0 & n & 0 & 0\\
0 & 0 & f+n & f\,n\\
0 & 0 & -1 & 0
\end{array}\right)$

That's all. 可以看到投影矩阵和之前的差在乘以一个负号，最终算出来的就是 OpenGL 的公式了。

glm 的函数

```cpp
template<typename T>
GLM_FUNC_QUALIFIER mat<4, 4, T, defaultp> perspectiveRH_NO(T fovy, T aspect, T zNear, T zFar)
{
  assert(abs(aspect - std::numeric_limits<T>::epsilon()) > static_cast<T>(0));

  T const tanHalfFovy = tan(fovy / static_cast<T>(2));

  mat<4, 4, T, defaultp> Result(static_cast<T>(0));
  Result[0][0] = static_cast<T>(1) / (aspect * tanHalfFovy);
  Result[1][1] = static_cast<T>(1) / (tanHalfFovy);
  Result[2][2] = - (zFar + zNear) / (zFar - zNear);
  Result[2][3] = - static_cast<T>(1);
  Result[3][2] = - (static_cast<T>(2) * zFar * zNear) / (zFar - zNear);
  return Result;
}
```

Matlab 代入我推的公式

```matlab
syms zNear zFar width height fovy aspect;  

% world space is right hand
% zNear > 0, zFar > 0
n = zNear;  
f = zFar;  

tanHalfFovy = tan(fovy/2);  
height = 2 * n * tanHalfFovy;
width = aspect * height;  
  
r = width/2;  
l = -width/2;
t = height/2;  
b = -height/2;  
  
Mortho = [2/(r-l) 0 0 0; 0 2/(t-b) 0 0; 0 0 2/(n-f) 0; 0 0 0 1] * [1 0 0 -(r+l)/2; 0 1 0 -(t+b)/2; 0 0 1 (n+f)/2; 0 0 0 1];
Mortho = simplify(Mortho);
Mpersp2ortho = [n 0 0 0; 0 n 0 0; 0 0 (n+f) n*f; 0 0 -1 0];
Mproj = Mortho * Mpersp2ortho;
Mproj = simplify(Mproj)

```

得到的结果一样

$M_{proj} = \left(\begin{array}{cccc}
\frac{1}{\mathrm{aspect}\,\tan \left(\frac{\mathrm{fovy}}{2}\right)} & 0 & 0 & 0\\
0 & \frac{1}{\tan \left(\frac{\mathrm{fovy}}{2}\right)} & 0 & 0\\
0 & 0 & -\frac{\mathrm{zFar}+\mathrm{zNear}}{\mathrm{zFar}-\mathrm{zNear}} & -\frac{2\,\mathrm{zFar}\,\mathrm{zNear}}{\mathrm{zFar}-\mathrm{zNear}}\\
0 & 0 & -1 & 0
\end{array}\right)$

这就是所谓的齐次坐标里面放 z 还是 -z 会导致的公式的不同

这一点不会导致结果不同

## 用于 Vulkan 的透视矩阵

视图矩阵是右手系，正交长方体的左平面的坐标为 l，右平面的坐标为 r，上平面的坐标为 t，下平面的坐标为 b，近平面的坐标为 -n，远平面的坐标为 -f。转换之后 NDC 坐标还是右手系。

n, f 为正

[b, t] 映射到 [1, -1], [-n, -f] 映射到 [0, 1]

$
M_{ortho}=\left(\begin{array}{cccc}
\frac{2}{r-l} & 0 & 0 & 0\\
0 & \frac{2}{b-t} & 0 & 0\\
0 & 0 & \frac{1}{n-f} & 0\\
0 & 0 & 0 & 1
\end{array}\right)\left(\begin{array}{cccc}
1 & 0 & 0 & -\frac{r+l}{2}\\
0 & 1 & 0 & -\frac{t+b}{2}\\
0 & 0 & 1 & n\\
0 & 0 & 0 & 1
\end{array}\right)
$

沿用之前推 OpenGL 推出来的压缩矩阵

$M_{persp2ortho} = \left(\begin{array}{cccc}
n & 0 & 0 & 0\\
0 & n & 0 & 0\\
0 & 0 & f+n & f\,n\\
0 & 0 & -1 & 0
\end{array}\right)$

Matlab 代入

```matlab
syms zNear zFar width height fovy aspect;  

% world space is right hand
% zNear > 0, zFar > 0
n = zNear;  
f = zFar;  

tanHalfFovy = tan(fovy/2);  
height = 2 * n * tanHalfFovy;
width = aspect * height;  
  
r = width/2;  
l = -width/2;
t = height/2;  
b = -height/2;  
  
Mortho = [2/(r-l) 0 0 0; 0 2/(b-t) 0 0; 0 0 1/(n-f) 0; 0 0 0 1] * [1 0 0 -(r+l)/2; 0 1 0 -(t+b)/2; 0 0 1 n; 0 0 0 1];
Mortho = simplify(Mortho);
Mpersp2ortho = [n 0 0 0; 0 n 0 0; 0 0 (n+f) n*f; 0 0 -1 0];
Mproj = Mortho * Mpersp2ortho;
Mproj = simplify(Mproj)
```

Matlab 结果

$M_{proj} = \left(\begin{array}{cccc}
\frac{1}{\mathrm{aspect}\,\tan \left(\frac{\mathrm{fovy}}{2}\right)} & 0 & 0 & 0\\
0 & -\frac{1}{\tan \left(\frac{\mathrm{fovy}}{2}\right)} & 0 & 0\\
0 & 0 & -\frac{\mathrm{zFar}}{\mathrm{zFar}-\mathrm{zNear}} & -\frac{\mathrm{zFar}\,\mathrm{zNear}}{\mathrm{zFar}-\mathrm{zNear}}\\
0 & 0 & -1 & 0
\end{array}\right)$

得到的结果与 `perspectiveRH_ZO` 确实仅仅在 [1][1] 差一个负号

```cpp
template<typename T>
GLM_FUNC_QUALIFIER mat<4, 4, T, defaultp> perspectiveRH_ZO(T fovy, T aspect, T zNear, T zFar)
{
  assert(abs(aspect - std::numeric_limits<T>::epsilon()) > static_cast<T>(0));

  T const tanHalfFovy = tan(fovy / static_cast<T>(2));

  mat<4, 4, T, defaultp> Result(static_cast<T>(0));
  Result[0][0] = static_cast<T>(1) / (aspect * tanHalfFovy);
  Result[1][1] = static_cast<T>(1) / (tanHalfFovy);
  Result[2][2] = zFar / (zNear - zFar);
  Result[2][3] = - static_cast<T>(1);
  Result[3][2] = -(zFar * zNear) / (zFar - zNear);
  return Result;
}
```

但是如果仅仅就这么用了

```cpp
glm::vec3 forward = transfrom_comp_ptr->rotation * glm::vec3(0.0f, 0.0f, 1.0f);
glm::mat4 view    = glm::lookAt(
    transfrom_comp_ptr->position, transfrom_comp_ptr->position + forward, glm::vec3(0.0f, 1.0f, 0.0f));

ubo_data.view       = view;
ubo_data.projection = glm::perspectiveRH_ZO(camera_comp_ptr->field_of_view,
                                            (float)window_size[0] / (float)window_size[1],
                                            camera_comp_ptr->near_plane,
                                            camera_comp_ptr->far_plane);
ubo_data.projection[1][1] *= -1.f;
```

还会有 x 轴反转的问题

这个确实……有点难以思考原因。我觉得可能还是因为反转了 z 轴的问题。

于是最终还是自己抄了一个透视矩阵，其中与 `glm::perspectiveRH_ZO` 的区别就是反转了 x 轴，然后用 viewport 负高度，front 设置为 `vk::FrontFace::eClockwise`

```cpp
static glm::mat4 perspective_vk(float fovy, float aspect, float zNear, float zFar)
{
    assert(abs(aspect - std::numeric_limits<float>::epsilon()) > static_cast<float>(0));

    float const tanHalfFovy = tan(fovy / 2.0f);

    glm::mat4 Result(0.0f);
    Result[0][0] = -1.0f / (aspect * tanHalfFovy);
    Result[1][1] = 1.0f / (tanHalfFovy);
    Result[2][2] = zFar / (zNear - zFar);
    Result[2][3] = -1.0f;
    Result[3][2] = -(zFar * zNear) / (zFar - zNear);
    return Result;
}
```

这样是可以 work，也可以保证用的是 glm 的 view 空间，也是基于 glm 的透视矩阵改的，我觉得还 ok

别人也会有类似的 x 轴翻转的问题

[https://stackoverflow.com/questions/65049297/perspective-projection-inverting-x-axis-glmperspective](https://stackoverflow.com/questions/65049297/perspective-projection-inverting-x-axis-glmperspective)

[https://stackoverflow.com/questions/78557339/glmlookat-image-is-visually-flipped-both-x-and-y-axis](https://stackoverflow.com/questions/78557339/glmlookat-image-is-visually-flipped-both-x-and-y-axis)

但是我脑子有限不知道怎么办

<script src="https://utteranc.es/client.js"
        repo="CheapMeow/cheapmeow.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>