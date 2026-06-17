---
sidebar_position: 4
---

# 4. 光线、简单相机与背景

# 4.1 `ray` 类

所有光线追踪器都具备的一个共同点，就是拥有一个 `ray`（射线）类，以及一个计算沿着射线能看到什么颜色的算法。我们可以把射线看作一个函数：$P(t)=a+tb$ 。在这里，$P$ 是三维空间中沿着一条直线的三维位置（坐标）。$A$ 是射线起点（ray origin），$B$ 是射线方向（ray direction）。射线参数 t 是一个实数（在代码中为 `double` 类型）。代入不同的 t 值，P(t) 就会沿着射线移动该点。如果加上负的 t 值，你可以到达这条三维直线上的任意位置。而对于正的 t 值，你只能得到 $A$ 前方的部分，这也就是人们常说的射线（half-line，或直接叫 ray）。

[![表 2：线性插值](https://raytracing.github.io/images/fig-1.02-lerp.jpg)](https://raytracing.github.io/images/fig-1.02-lerp.jpg)

我们可以用一个类来表示射线的概念，并用一个我们取名为 `ray::at(t)` 的函数来表示 $P(t)$ 函数：

```cpp
#ifndef RAY_H
#define RAY_H

#include "vec3.h"

class ray {
  public:
    ray() {}

    ray(const point3& origin, const vec3& direction) : orig(origin), dir(direction) {}

    const point3& origin() const  { return orig; }
    const vec3& direction() const { return dir; }

    point3 at(double t) const {
        return orig + t*dir;
    }

  private:
    point3 orig;
    vec3 dir;
};

#endif
```

（对于不熟悉 C++ 的读者，函数 `ray::origin()` 和 `ray::direction()` 均会返回其成员的不可变引用。调用者可以直接使用该引用，也可以根据需要创建一个可变副本。）

# 4.2 将光线送入场景中

现在我们准备好迈入下一阶段，製作一个光线追踪器了。光线追踪器的核心原理，就是向像素投射射线，并计算在这些射线方向上所看到的颜色。涉及的步骤包括：
1. 计算从 " 视点（eye）" 穿过像素的射线
2. 确定射线与哪些物体相交
3. 计算最近交点处的颜色
在最初开发光线追踪器时，我总会搭建一个简单相机，以便让代码先运行起来。
在 debug 时，我经常在使用正方形图像上遇到麻烦，因爲太容易将 x 和 y 座标转置了，所以我们将使用一个非正方形的图像。正方形图像的宽高比是 1：1 ，因为它的宽和高相同。既然我们使用非正方形图像，我们选择常见的 16：9。16：9 的宽高比意味着图像宽度与高度的比值是 16：9。换句话说，对于一个宽高比 16：9 的图像，
$$宽/高=16/9=1.7778$$
举一个实际的例子，一个 800 像素宽，400 像素高的图像，宽高比是 2：1。

图像的宽高比由宽和高的长度比例决定。然而，既然我们脑中已经有一个固定的宽高比，那么更方便的做法是先设定图像的宽高比，再依此计算它的高度。通过这种方式，我们可以改变图像宽度来缩放图像，并且不会破坏我们想要的宽高比。不过我们必须确保最终计算出来的高度至少爲不过我们必须确保最终计算出来的高度至少爲 1。

除了为渲染图像设置像素尺寸，我们还需要设置一个虚拟的视口（viewpoint），以便让场景光线穿过它。视口是三维世界中的一个虚拟矩形，它包含了图像像素位置的网格。如果像素在水平和垂直方向上的间距相同，那么包裹他们的视口将具有与渲染图像相同的宽高比。两个相邻像素之间的距离称为像素间距（pixel spcing），而正方形像素（square pixels）是标准。

作为开始，我们任意选择一个为 2.0 的视口高度，并根据它缩放视口宽度，以满足我们期望的宽高比。以下是这部分代码大致的样子：

```cpp
auto aspect_ratio = 16.0 / 9.0;
int image_width = 400;

// 计算图像的高，并确保它至少为 1。
int image_height = int(image_width / aspect_ratio);
image_height = (image_height < 1) ? 1 : image_height;

// 由于视口宽度是实际值，它可以小于 1。
auto viewport_height = 2.0;
auto viewport_width = viewport_height * (double(image_width)/image_height);
```

你想知道为什么计算 `viewport_width` 的时候不直接使用 `aspect_ratio` ，这是因为设成 `aspsct_ratio` 的值是理想比例，它可能不是 `image_width` 和 `image-height` 的真实比例。如果 `image_height` 被允许是一个实际值——而不只是一个整数——那么使用 `aspect_ratio` 是可行的。但 `image_width` 和 `image-height` 的实际值可能会受到两个代码中两个地方的影响而变化。首先， `image_height` 是透过向下取整来计算的，这可能会使实际比例增大。其次，我们不允许 `image_height` 小于 1，这同样可能会改变实际的宽高比。

注意到 `aspect_ratio` 是一个理想比例，我们通过基于整数的图像宽度除以图像宽度来尽可能接近它。为了让我们的视口比例和图像比例完全匹配，我们使用计算出来的图像实际宽高比，以决定最终的是口宽度。

接下来，我们将定义一个相机中心：一个三维空间中的点，所有场景射线都将从该点发出（此点也常被称为视点（eye point））。相机中心到视口中心的向量和视口正交。我们会先将视口到相机中心点的距离设为一个单位。这个距离通常被称为焦距（focal length）。

为了简化，我们先从相机中心为（0, 0, 0）开始。我们还设定 y 轴向上， x 轴向右， z 轴负极指向视线方向。这通常被称为右手坐标系。

[![表 3：相机几何结构](https://raytracing.github.io/images/fig-1.03-cam-geom.jpg)](https://raytracing.github.io/images/fig-1.03-cam-geom.jpg)

现在来到了不可避免的棘手部分。我们的三维空间遵循上述规定，但这与我们的图像坐标系产生了冲突——在图像中，我们希望第 0 个像素位于左上角，然后一次延伸至右下角最后一个元素。这意味着我们在图像坐标系的 y 轴是反转的：在图像中， y 值随下向移动而增大。

在我们扫描图像时，我们会由左上角的像素（像素 0, 0）开始，由左到右扫描每一列，然后一列一列由上到下扫描直到底部。为了在像素网格中定位，我们将使用一个从左边缘指向右边缘的向量（$V_u$），和一个从上边缘指到下边缘的向量（$V_v$）。

我们的像素网格将从视口边缘往内缩进半个像素间距。通过这个方式，我们的视口区域会被均匀划分为 $宽\times高$ 个完全相同的区域。以下是我们的视口与像素网格看起来的样子：

[![表 4：视口和像素网格](https://raytracing.github.io/images/fig-1.04-pixel-grid.jpg)](https://raytracing.github.io/images/fig-1.04-pixel-grid.jpg)

在这张图中，我们有视口，一个 $7\times5$ 分辨率的图像的像素网格，视口的左上角 $Q$ ，像素$P_{0,0}$ 位置，视口像素 $V_u$ （`viewport_u`），视口像素 $V_v$ （`viewport_v`），像素间距向量 $\Delta u\Delta v$ 。

基于以上所有几何关係，以下是实作相机的代码。我们将存根一个函数 `ray_color(const ray& r)`，他负责返回一条给定场景射线的颜色——目前我们将它设定成永远回传黑色。

```cpp
#include "color.h"
// highlight-start
#include "ray.h"
// highlight-end
#include "vec3.h"

#include <iostream>

// highlight-start
color ray_color(const ray& r) {
    return color(0,0,0);
}
// highlight-end

int main() {

    // 图像

	// highlight-start
    auto aspect_ratio = 16.0 / 9.0;
    int image_width = 400;

    // 计算图像的高，并确保它至少为 1。
    int image_height = int(image_width / aspect_ratio);
    image_height = (image_height < 1) ? 1 : image_height;

    // 相机

    auto focal_length = 1.0;
    auto viewport_height = 2.0;
    auto viewport_width = viewport_height * (double(image_width)/image_height);
    auto camera_center = point3(0, 0, 0);

    // 计算视口水平边缘和垂直边缘的向量
    auto viewport_u = vec3(viewport_width, 0, 0);
    auto viewport_v = vec3(0, -viewport_height, 0);

    // 计算相邻像素间水平和垂直间距向量
    auto pixel_delta_u = viewport_u / image_width;
    auto pixel_delta_v = viewport_v / image_height;

    // 计算左上角像素的位置
    auto viewport_upper_left = camera_center
                             - vec3(0, 0, focal_length) - viewport_u/2 - viewport_v/2;
    auto pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);
	// highlight-end

    // 渲染

    std::cout << "P3\n" << image_width << " " << image_height << "\n255\n";

    for (int j = 0; j < image_height; j++) {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; i++) {
	        // highlight-start
            auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
            auto ray_direction = pixel_center - camera_center;
            ray r(camera_center, ray_direction);

            color pixel_color = ray_color(r);
            // highlight-end
            write_color(std::cout, pixel_color);
        }
    }

    std::clog << "\rDone.                 \n";
}
```

注意，在上面的代码中，我并没有将 `ray_direction` 转换为单位向量，因为我认为不这样做代码会更简洁、运行速度更快。

现在我们来填充 `ray_color(ray)` 函数，以实作一个简单的渐变。这个函数会在将光线方向缩放至单位长度后（此时 $-1.0<y<1.0$ ），根据 y 轴高度，以线性方式混合白色和蓝色。因为我们观察的是向量归一化后的 y 轴高度，你会注意到，除了垂直渐变以外，颜色还会出现水平方向的渐变。

我将使用一个标准的图形学技巧，将其缩放到 $-0.0\le a\le 1.0$ 的区间。当 $a=1.0$ 时，我想要蓝色；当 $a=0.0$ 时，我想要白色。而在两者之间，我想要一种混色效果。这就构成了一个"线性混合"或"线性插值"。这通常被简称为两者之间的 lerp 。 lerp 的形式如下：
$$blendedValue=(1-a)\cdot startValue+a\cdot endValue,$$
其中 $a$ 由 0 走到 1。
将这些全部合在一起，我们会得到：

```cpp
#include "color.h"
#include "ray.h"
#include "vec3.h"

#include <iostream>


color ray_color(const ray& r) {
	// highlight-start
    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5*(unit_direction.y() + 1.0);
    return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    // highlight-end
}

...
```

在这个例子中，会产出：

[![图 2：一个取决于 y 轴的蓝白渐变](https://raytracing.github.io/images/img-1.02-blue-to-white.png)](https://raytracing.github.io/images/img-1.02-blue-to-white.png)