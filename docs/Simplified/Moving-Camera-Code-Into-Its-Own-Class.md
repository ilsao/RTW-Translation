---
sidebar_position: 7
---

# 7. 将相机代码移至它自己的类

在继续之前，现在是将我们的相机和场景渲染代码整合到一个单一新类的好时机：即 `camera` 类。`camera` 类将负责两项重要工作： 

1. 构造光线并将其派遣到世界中。 
2. 使用这些光线的结果来构造渲染出的图像。 

在这次重构中，我们将收集 `ray_color()` 函数，以及我们主程序中的图像、相机和渲染部分。新的 `camera` 类将包含两个共有方法 `initialize()` 和 `render()`，再加上两个私有辅助方法 `get_ray()` 和 `ray_color()`。 

最终，相机将遵循我们所能想到的最简单的使用模式：它将被默认构造无参数，然后拥有它的代码将通过简单的赋值来修改相机的共有变量，最后通过调用 `initialize()` 函数来初始化一切。选择这种模式，而非让拥有者调用带有大量参数的构造函数，或者通过定义和调用一堆 setter 方法。相反，拥有它的代码只需要设置它明确关心的内容。最后，我们既可以让拥有它的代码调用 `initialize()`，也可以直接让相机在 `render()` 开始时自动调用该函数。我们将使用后者。

在 main 创建一个相机并设置默认值之后，它将调用 `render()` 方法。`render()` 方法将为相机做好渲染准备，然后执行渲染循环。 这是我们新 `camera` 类的骨架：

```cpp
#ifndef CAMERA_H
#define CAMERA_H

#include "hittable.h"

class camera {
  public:
    /* 公有相机参数如下 */

    void render(const hittable& world) {
        ...
    }

  private:
    /* 私有相机变量如下 */

    void initialize() {
        ...
    }

    color ray_color(const ray& r, const hittable& world) const {
        ...
    }
};

#endif
```

首先，让我们填入来自 `main.cc` 的 `ray_color()` 函数：

```cpp
class camera {
  ...

  private:
    ...


    color ray_color(const ray& r, const hittable& world) const {
	    // diff-add-start
        hit_record rec;

        if (world.hit(r, interval(0, infinity), rec)) {
            return 0.5 * (rec.normal + color(1,1,1));
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
        // diff-add-end
    }
};

#endif
```

现在我们将几乎所有的内容从 `main()` 函数移动到我们新的相机类中。`main()` 函数中唯一剩下的东西就是世界的构造。带有新迁移代码的相机类如下：

```cpp
class camera {
  public:
  // diff-add-start
    double aspect_ratio = 1.0;  // 图像的宽高比
    int    image_width  = 100;  // 渲染图像的宽度（以像素为单位）

    void render(const hittable& world) {
        initialize();

        std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

        for (int j = 0; j < image_height; j++) {
            std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
            for (int i = 0; i < image_width; i++) {
                auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
                auto ray_direction = pixel_center - center;
                ray r(center, ray_direction);

                color pixel_color = ray_color(r, world);
                write_color(std::cout, pixel_color);
            }
        }

        std::clog << "\rDone.                 \n";
    }
    // diff-add-end

  private:
  // diff-add-start
    int    image_height;   // 渲染图像的高
    point3 center;         // 相机中心
    point3 pixel00_loc;    // 像素 0, 0 的位置
    vec3   pixel_delta_u;  // 向右移动的像素偏移量
    vec3   pixel_delta_v;  // 向下移动的像素偏移量

    void initialize() {
        image_height = int(image_width / aspect_ratio);
        image_height = (image_height < 1) ? 1 : image_height;

        center = point3(0, 0, 0);

        // 确定视口维度。
        auto focal_length = 1.0;
        auto viewport_height = 2.0;
        auto viewport_width = viewport_height * (double(image_width)/image_height);

        // 计算视口水平边缘和垂直边缘的向量
        auto viewport_u = vec3(viewport_width, 0, 0);
        auto viewport_v = vec3(0, -viewport_height, 0);

        // 计算相邻像素间水平和垂直间距向量
        pixel_delta_u = viewport_u / image_width;
        pixel_delta_v = viewport_v / image_height;

        // 计算左上角像素的位置
        auto viewport_upper_left =
            center - vec3(0, 0, focal_length) - viewport_u/2 - viewport_v/2;
        pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);
    }
    // diff-add-end

    color ray_color(const ray& r, const hittable& world) const {
        ...
    }
};

#endif
```

然后这是大幅简化后的 `main`：

```cpp
#include "rtweekend.h"

#include "camera.h"
#include "hittable.h"
#include "hittable_list.h"
#include "sphere.h"

color ray_color(const ray& r, const hittable& world) {
    ...
}

int main() {
    hittable_list world;

    world.add(make_shared<sphere>(point3(0,0,-1), 0.5));
    world.add(make_shared<sphere>(point3(0,-100.5,-1), 100));

    camera cam;

    cam.aspect_ratio = 16.0 / 9.0;
    cam.image_width  = 400;

    cam.render(world);
}
```

运行这个重构的程序会得到跟之前相同的图像。
