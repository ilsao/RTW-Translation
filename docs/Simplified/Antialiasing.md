---
sidebar_position: 8
---

# 8. 抗鋸齒 (Antialiasing)

若你放大目前渲染出来的图像，可能会注意到边缘呈现出很明显的 “阶梯状” 粗糙感。这种阶梯状通常称为 “走样”（aliasing），或 “锯齿”（jaggies）。真实相机拍照时，边缘通常不会有锯齿，因为边缘处的像素往往是前景與背景的混合。注意到，與我们渲染出来的图像不同，真实世界中的图像是连续的。即，世界本身（以及它的任何真实图像）实际上都有无限分辨率。我们可以藉由对每个像素取一组样本并求平均，来取得类似的效果。

当每个像素只从中心发出一条設限时，我们使用的是点采样 （point sampling）。点采样的问题可藉由渲染一个距离很远的小棋盘来说明。假设这个棋盘由一个 $8 \times 8$ 的黑白方格组成，但只有四条光线打到它上面，那么这四条光线可能全部只与白色方格相交，也可能全部只与黑色方格相交，或者出现某种奇怪的组合。在真实世界中，当我们用眼睛看远处的棋盘时，感知到的会是种灰色，而非尖锐分明的黑白点。这是因为我们的眼睛會自然地做我们希望光线追踪器去做的事：对落在渲染图像中某个特定区域 (离散) 上的光 (连续函数) 做整合。

显然，如果只是对穿过像素中心的同一条光线重复采样，并不会带来任何收益，每次得到的结果都会一样。相反，我们希望对落在像素周围的光进行采样，然后对这些样本进行整合，以近似真实的连续结果。那么，我们该如何对落在像素周围的光进行整合呢？

我們採用最簡單的模型：對以像素點為中心，延伸到與鄰近像素一半長度為邊的正方形採樣。(看看 [A Pixel Is Not A Little Square](https://www.researchgate.net/publication/244986797) 來更深入的探索這個主題)

# 8.1 一些隨機數工具

## 一些随机数工具

我们将需要一个随机数生成器，用来返回随机實數。这个函数应该返回一个规范化随机数（canonical random number），按照惯例，其取值范围為：$0 \le n < 1$。这里 1 前面的 “小于” 很重要，因为后面我们有时会利用这一点。

一个简单的方法是使用 `<cstdlib>` 中的 `std::rand()` 函数，它会返回一个范围在 `0` 到 `RAND_MAX` 之间的随机整数。因此，我们可以用下面这段代码得到想要的隨機实数，并將它添加到 `rtweekend.h` 中：

```cpp {}
#include <cmath>
// highlight-start
#include <cstdlib>
// highlight-end
#include <iostream>
#include <limits>
#include <memory>
...

// 工具函數

inline double degrees_to_radians(double degrees) {
    return degrees * pi / 180.0;
}

// highlight-start
inline double random_double() {
    // 返回一個在 [0,1) 上的隨機實數
    return std::rand() / (RAND_MAX + 1.0);
}

inline double random_double(double min, double max) {
    // 返回一個在 [min,max) 上的隨機實數
    return min + (max-min)*random_double();
}
// highlight-end
```

傳統上 C++ 并没有标准的随机数生成器，不过较新版本的 C++ 已经通过 `<random>` 头文件解决了这个问题（尽管一些专家认为这个方案并不完美）。如果你想用它，可以按照下面的方式生成一个满足我们需求的随机数：

``` cpp
...

// highlight-start
#include <random>
// highlight-end

...

// highlight-start
inline double random_double() {
    static std::uniform_real_distribution<double> distribution(0.0, 1.0);
    static std::mt19937 generator;
    return distribution(generator);
}
// highlight-end

inline double random_double(double min, double max) {
    // 返回一個範圍在 [min,max) 上的隨機實數
    return min + (max-min)*random_double();
}

...
```

# 8.2 使用多个样本生成像素

对于一个由多个样本组成的像素，我们会从像素周围的区域中选取样本，并将得到的光照（颜色）值取平均。

首先，我们会更新 write_color() 函数，使其能够考虑所使用的样本数量：我们需要求出所有采样结果的平均值。为此，我们会在每次迭代中累加完整的颜色值，然后在最后写出颜色之前，只进行一次除法运算（除以样本数量）。为了确保最终结果的颜色分量仍然保持在合适的 $[0,1]$ 范围，我们会添加并使用一个辅助函数：interval::clamp(x)。

``` cpp
class interval {
  public:
    ...

    bool surrounds(double x) const {
        return min < x && x < max;
    }

    // highlight-start
    double clamp(double x) const {
        if (x < min) return min;
        if (x > max) return max;
        return x;
    }
    // highlight-end
    ...
};
```

下面是更新后的 `write_color()` 函数，它引入了区间钳制函数：

``` cpp
// highlight-start
#include "interval.h"
// highlight-end
#include "vec3.h"

using color = vec3;

void write_color(std::ostream& out, const color& pixel_color) {
    auto r = pixel_color.x();
    auto g = pixel_color.y();
    auto b = pixel_color.z();

    // 將 [0, 1] 分量變換到自結範圍 [0, 255]
    // highlight-start
    static const interval intensity(0.000, 0.999);
    int rbyte = int(256 * intensity.clamp(r));
    int gbyte = int(256 * intensity.clamp(g));
    int bbyte = int(256 * intensity.clamp(b));
    // highlight-end

    // 寫出像素顏色分量
    out << rbyte << ' ' << gbyte << ' ' << bbyte << '\n';
}
```

现在，让我们更新 `camera` 类，定义并使用一个新的 `camera::get_ray(i,j)` 函数，用来为每个像素生成不同的样本。这个函数使用一个新的辅助函数 `sample_square()`，它会在以原点为中心的单位正方形内生成一个随机采样点。然后，我们再把这个随机样本，变换回当前正在采样的具体像素上。

``` cpp
class camera {
  public:
    double aspect_ratio      = 1.0;  // 圖像寬度與高度的比值
    int    image_width       = 100;  // 渲染圖像寬 (以像素為單位)
    // highlight-start
    int    samples_per_pixel = 10;   // 每個像素的隨機採樣數
    // highlight-end

    void render(const hittable& world) {
        initialize();

        std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

        for (int j = 0; j < image_height; j++) {
            std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
            for (int i = 0; i < image_width; i++) {
                // highlight-start
                color pixel_color(0,0,0);
                for (int sample = 0; sample < samples_per_pixel; sample++) {
                    ray r = get_ray(i, j);
                    pixel_color += ray_color(r, world);
                }
                write_color(std::cout, pixel_samples_scale * pixel_color);
                // highlight-end
            }
        }

        std::clog << "\rDone.                 \n";
    }
    ...
  private:
    int    image_height;         // 渲染圖像高
    // highlight-start
    double pixel_samples_scale;  // 像素样本总和的颜色缩放因子
    // highlight-end
    point3 center;               // 攝像機中心
    point3 pixel00_loc;          // 像素 0, 0 的位置
    vec3   pixel_delta_u;        // 像素到其右側的偏移量
    vec3   pixel_delta_v;        // 像素到其下的偏移量

    void initialize() {
        image_height = int(image_width / aspect_ratio);
        image_height = (image_height < 1) ? 1 : image_height;

        // highlight-start
        pixel_samples_scale = 1.0 / samples_per_pixel;
        // highlight-end

        center = point3(0, 0, 0);
        ...
    }

    // highlight-start
    ray get_ray(int i, int j) const {
        // 從 origin 構造出一個攝像機設限，其指向像素位置 i, j 附近的隨機採樣點

        auto offset = sample_square();
        auto pixel_sample = pixel00_loc
                          + ((i + offset.x()) * pixel_delta_u)
                          + ((j + offset.y()) * pixel_delta_v);

        auto ray_origin = center;
        auto ray_direction = pixel_sample - ray_origin;

        return ray(ray_origin, ray_direction);
    }

    vec3 sample_square() const {
        // 返回單位正方形 [-.5, -.5]-[+.5, +.5] 內的隨機點向量
        return vec3(random_double() - 0.5, random_double() - 0.5, 0);
    }
    // highlight-end

    color ray_color(const ray& r, const hittable& world) const {
        ...
    }
};

#endif
```

（除了上面新的 `sample_square()` 函数之外，你还可以在 GitHub 源代码中找到 `sample_disk()` 函数。它被包含进来为了方便你尝试非正方形像素，不过本书中不会使用該函數。`sample_disk()` 依赖于稍后才会定义的 `random_in_unit_disk()` 函数。）

`main` 函数也会被更新，用来设置新的相机参数。

```cpp
int main() {
    ...

    camera cam;

    cam.aspect_ratio      = 16.0 / 9.0;
    cam.image_width       = 400;
    // highlight-start
    cam.samples_per_pixel = 100;
    // highlight-end

    cam.render(world);
}
```

放大產出的圖像，我們可以看出邊緣像素的差別：

![](https://raytracing.github.io/images/img-1.06-antialias-before-after.png)