---
sidebar_position: 5
---

# 5. 加入一个球体

让我们加入一个物件到光线追踪器。人们很常在光线追踪器中使用球体，因为计算一道光线是否击中一个球体相对简单。

# 5.1. 光线与球体求交

一个球心为原点，半径为 $r$ 的球体方程，是一个很重要的数学方程式：
$$x^2+y^2+z^2=r^2$$
你也可以想成：如果给定一个球面上的点 $(x, y, z)$ ，则 $x^2+y^2+z^2=r^2$ ；如果给定一个球体内部的点 $(x, y, z)$ ，则 $x^2+y^2+z^2<r^2$ ；如果给定一个球体外部的点 $(x, y, z)$ ，则 $x^2+y^2+z^2>r^2$ 。

如果我们想允许球心是任意的点 $(C_x,C_y,C_z)$ ，则方程式就会没那么漂亮：
$$(C_x-x)^2+(C_y-y)^2+(C_z-z)^2=r^2$$
在图形学中，希望尽可能用向量表达公式，这样所有的 $x/y/z$ 就可以简单用一个 `vec3` 类来表示。
你可能已经注意到，从点 $P=(x,y,z)$ 指向球心 $C=(C_x,C_y,C_z)$ 的向量是 $(C-P)$ 。

如果我们使用点积的定义：
$$(C-P)\cdot(C-P)=(C_x-x)^2+(C_y-y)^2+(C_z-z)^2$$
于是我们将球体公式改写成向量形式：
$$(C-P)\cdot(C-P)=r^2$$
我们可以将其理解为”任何满足该方程的点 $P$ 都在球面上“。我们想知道，光线 $P(t)=Q+td$ 是否击中了球体。如果它确实击中了球体，那么必然存在某个 $t$ 值，使得 $P(t)$ 能够满足该方程。因此，我们寻找的就是能让以下等式成立的 $t$ ：
$$(C-P(t))\cdot(C-P(t))=r^2$$
可以将 $P(t)$ 代换成它的展开式来解：
$$(C-(Q+td))\cdot(C-(Q+td))=r^2$$
左边有三个向量，右边也有三个向量，而他们都要进行点积运算。如果我们将整个点积完全展开，将会得到九个项。你当然可以一步一步把它写出来，但没必要那么辛苦。如果你还记得的话，我们的目标是求解 $t$ ，因此我们可以根据一项中是否包含 $t$ 来将这些项分组归类。
$$(-td+(C-Q))\cdot(-td+(C-Q))=r^2$$
接下来我们利用向量代数的分配律展开这个点积：
$$t^2d\cdot d-2td\cdot (C-Q)+(C-Q)\cdot(C-Q)=r^2$$
将 $r$ 平方项移项至左侧：
$$t^2d\cdot d-2td\cdot (C-Q)+(C-Q)\cdot(C-Q)-r^2=0$$
儘管很难一眼看清这个方程到底是什么，但其中的向量和半径 $r$ 其实都是已知常数。此外，方程中仅有的几个向量也透过点积转换成了纯量。唯一的未知数只有 $t$ ，并且项中包含一个 $t^2$ ，这意味这它是一个一元二次方程。你可以透过求根方式来求解标准式为 $ax^2+bx+c=0$ 的一元二次方程：
$$\frac{-b\pm\sqrt{b^2-4ac}}{2a}$$
所以，求解光线与球体求交方程中的 $t$ ，可以得到如下的 $a, b, c$ ：
$$ a=d\cdot d$$
$$b=-2d\cdot (C-Q)$$
$$c=(C-Q)\cdot(C-Q)-r^2$$
利用上述所有公式，你可以解出 $t$ 。但在公式中包含一个平方根，它的值可以是正数（意味着有两个实数解）、负数（意味着没有实数解）或零（意味着有一个实数解）。在图形学中，代数关係几乎总是和几何图形学有非常直接的对应关係。我们面对的具体情况如下：
[![表 5：光线与球体求交结果](https://raytracing.github.io/images/fig-1.05-ray-sphere.jpg)](https://raytracing.github.io/images/fig-1.05-ray-sphere.jpg)
# 5.2. 创造我们的第一个光线追踪图像

 如果我们直接把这些数学公式写死到程序中，我们就可以进行代码测试了：在 $z$ 轴上 -1 的位置放置一个小球然后将任何与该球体相交的像素涂成红色。

```cpp
// heighlight-start
bool hit_sphere(const point3& center, double radius, const ray& r) {
    vec3 oc = center - r.origin();
    auto a = dot(r.direction(), r.direction());
    auto b = -2.0 * dot(r.direction(), oc);
    auto c = dot(oc, oc) - radius*radius;
    auto discriminant = b*b - 4*a*c;
    return (discriminant >= 0);
}
// highlight-end

color ray_color(const ray& r) {
// highlight-start
    if (hit_sphere(point3(0,0,-1), 0.5, r))
        return color(1, 0, 0);
// highlight-end

    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5*(unit_direction.y() + 1.0);
    return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
}
```

我们将会得到：

[![图 3：一个简单红色球体](https://raytracing.github.io/images/img-1.03-red-sphere.png)](https://raytracing.github.io/images/img-1.03-red-sphere.png)

目前这个渲染器还缺少各式各样的功能——比如着色（shading）、反射光线（reflection rays）以及多个物体的渲染——但我们现在的进度离大功告成已经比一开始近多了！需要注意的一点是，我们目前是通过求解一元二次方程并判断是否存在解，来检测光线是否与球体相交的。然而，即使 $t$ 的值为负数，这个方程解依然能成立。如果你把球新位置改到 $z=+1$ ，你会得到完全相同的画面，因为目前的求解方法根本不能区分物体是在相机前方还是相机后方。这可不是一个 feature！我们接下来会解决这个问题。