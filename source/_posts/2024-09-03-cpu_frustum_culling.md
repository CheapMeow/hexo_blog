---
title: 'CPU 视锥体剔除'
date: 2024-09-03 20:50:00
tags:
  - Grphics
  - Vulkan
  - Culling
---

抄 [https://github.com/Angelo1211/SoftwareRenderer](https://github.com/Angelo1211/SoftwareRenderer) 的视锥体剔除时遇到的一些问题

## 代码实现

这是我转成 glm 的版本

plane.h

```cpp
struct Plane
{
    glm::vec3 normal;
    float     D;

    float distance(const glm::vec3& points);
    void  setNormalAndPoint(const glm::vec3& normal, const glm::vec3& point);
};
```

plane.cpp

```cpp
float Plane::distance(const glm::vec3& points) { return glm::dot(normal, points) + D; }

void Plane::setNormalAndPoint(const glm::vec3& n, const glm::vec3& p0)
{
    normal = n;
    D      = -glm::dot(n, p0);
}
```

frustum.h

```cpp
class Frustum
{
private:
    enum
    {
        TOP = 0,
        BOTTOM,
        LEFT,
        RIGHT,
        NEARP,
        FARP
    };

public:
    void
    updatePlanes(const glm::vec3 cameraPos, const glm::quat rotation, float fovy, float AR, float near, float far);
    bool checkIfInside(BoundingBox* bounds);

private:
    Plane pl[6];
};
```

frustum.cpp

```cpp
// Calculates frustum planes in world space
void Frustum::updatePlanes(const glm::vec3 cameraPos,
                            const glm::quat rotation,
                            float           fovy,
                            float           AR,
                            float           near,
                            float           far)
{
    float tanHalfFOVy = tan(fovy / 2.0f);
    float near_height = near * tanHalfFOVy; // Half of the frustum near plane height
    float near_width  = near_height * AR;

    glm::vec3 right   = rotation * glm::vec3(1.0f, 0.0f, 0.0f);
    glm::vec3 forward = rotation * glm::vec3(0.0f, 0.0f, 1.0f);
    glm::vec3 up      = glm::vec3(0.0f, 1.0f, 0.0f);

    // Gets worlds space position of the center points of the near and far planes
    // The forward vector Z points towards the viewer so you need to negate it and scale it
    // by the distance (near or far) to the plane to get the center positions
    glm::vec3 nearCenter = cameraPos + forward * near;
    glm::vec3 farCenter  = cameraPos + forward * far;

    glm::vec3 point;
    glm::vec3 normal;

    // We build the planes using a normal and a point (in this case the center)
    // Z is negative here because we want the normal vectors we choose to point towards
    // the inside of the view frustum that way we can cehck in or out with a simple
    // Dot product
    pl[NEARP].setNormalAndPoint(forward, nearCenter);

    // Far plane
    pl[FARP].setNormalAndPoint(-forward, farCenter);

    // Again, want to get the plane from a normal and point
    // You scale the up vector by the near plane height and added to the nearcenter to
    // optain a point on the edge of both near and top plane.
    // Subtracting the cameraposition from this point generates a vector that goes along the
    // surface of the plane, if you take the cross product with the direction vector equal
    // to the shared edge of the planes you get the normal
    point  = nearCenter + up * near_height;
    normal = glm::normalize(point - cameraPos);
    normal = glm::cross(right, normal);
    pl[TOP].setNormalAndPoint(normal, point);

    // Bottom plane
    point  = nearCenter - up * near_height;
    normal = glm::normalize(point - cameraPos);
    normal = glm::cross(normal, right);
    pl[BOTTOM].setNormalAndPoint(normal, point);

    // Left plane
    point  = nearCenter - right * near_width;
    normal = glm::normalize(point - cameraPos);
    normal = glm::cross(up, normal);
    pl[LEFT].setNormalAndPoint(normal, point);

    // Right plane
    point  = nearCenter + right * near_width;
    normal = glm::normalize(point - cameraPos);
    normal = glm::cross(normal, up);
    pl[RIGHT].setNormalAndPoint(normal, point);
}

// False is fully outside, true if inside or intersects
// based on iquilez method
bool Frustum::checkIfInside(BoundingBox* box)
{
    // Check box outside or inside of frustum
    for (int i = 0; i < 6; ++i)
    {
        int out = 0;
        out += ((pl[i].distance(glm::vec3(box->min.x, box->min.y, box->min.z)) < 0.0) ? 1 : 0);
        out += ((pl[i].distance(glm::vec3(box->max.x, box->min.y, box->min.z)) < 0.0) ? 1 : 0);
        out += ((pl[i].distance(glm::vec3(box->min.x, box->max.y, box->min.z)) < 0.0) ? 1 : 0);
        out += ((pl[i].distance(glm::vec3(box->max.x, box->max.y, box->min.z)) < 0.0) ? 1 : 0);
        out += ((pl[i].distance(glm::vec3(box->min.x, box->min.y, box->max.z)) < 0.0) ? 1 : 0);
        out += ((pl[i].distance(glm::vec3(box->max.x, box->min.y, box->max.z)) < 0.0) ? 1 : 0);
        out += ((pl[i].distance(glm::vec3(box->min.x, box->max.y, box->max.z)) < 0.0) ? 1 : 0);
        out += ((pl[i].distance(glm::vec3(box->max.x, box->max.y, box->max.z)) < 0.0) ? 1 : 0);

        if (out == 8)
            return false;
    }
    return true;
}
```

他这个非常好理解

如果某个包围盒的八个顶点都在某个平面的外侧，那么就确定这个包围盒要被剔除

如果都不在视锥体 6 个平面的外侧，那么这个不剔除

我感觉巧妙的是你不用强求获得斜面的中点。而是根据近平面的中点直接沿着 up 或者 right 平移就能得到斜面上的点

我看到 plane 这个类的时候，我就想到一定要拿到中点，可能是我数学直觉没转过来

## 坑

第一个坑就是传入的 `fovy` 需要注意是角度还是弧度

这个是可以不用 debug 直接看出来怎么错了，因为如果你仅仅是在正对着物体的时候才不剔除，稍微偏一点头，物体都没有走出视口时就被剔除了

并且剔除的时机很稳定，不会一闪一闪，也就是说不会是公式错了，单纯是你的视锥体比你的视口小

那么就是某些地方算小了

第二个坑是获得 up right front 的方式

之前的分析里面也可以看到了，传入 `glm::lookAt` 的 front 方向会和 view 中拆出来的 front 方向相反的

所以如果认为 front 是 +z 的方向……还是自己手动算吧

第三个坑是算斜面的 normal 的时候，可能会因为手性的问题，导致 normal 算反

normal 算反在渲染时的表现就是，转动摄像机时，视口内的物体会不断交替剔除，就像在闪烁一样

正常来说，一个物体在视口内，那么他就是已经在视锥体里面了，所以他不应该闪的

<script src="https://utteranc.es/client.js"
        repo="CheapMeow/cheapmeow.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
