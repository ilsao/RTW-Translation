---
sidebar_position: 14
---

# 14. 接下来做什么？

# 14.1 最终的渲染器

让我们来生成这本书封面上的图像——由大量随机球体组成的场景。

```cpp
int main() {
    hittable_list world;

    // diff-add-start
    auto ground_material = make_shared<lambertian>(color(0.5, 0.5, 0.5));
    world.add(make_shared<sphere>(point3(0,-1000,0), 1000, ground_material));

    for (int a = -11; a < 11; a++) {
        for (int b = -11; b < 11; b++) {
            auto choose_mat = random_double();
            point3 center(a + 0.9*random_double(), 0.2, b + 0.9*random_double());

            if ((center - point3(4, 0.2, 0)).length() > 0.9) {
                shared_ptr<material> sphere_material;

                if (choose_mat < 0.8) {
                    // diffuse
                    auto albedo = color::random() * color::random();
                    sphere_material = make_shared<lambertian>(albedo);
                    world.add(make_shared<sphere>(center, 0.2, sphere_material));
                } else if (choose_mat < 0.95) {
                    // metal
                    auto albedo = color::random(0.5, 1);
                    auto fuzz = random_double(0, 0.5);
                    sphere_material = make_shared<metal>(albedo, fuzz);
                    world.add(make_shared<sphere>(center, 0.2, sphere_material));
                } else {
                    // glass
                    sphere_material = make_shared<dielectric>(1.5);
                    world.add(make_shared<sphere>(center, 0.2, sphere_material));
                }
            }
        }
    }

    auto material1 = make_shared<dielectric>(1.5);
    world.add(make_shared<sphere>(point3(0, 1, 0), 1.0, material1));

    auto material2 = make_shared<lambertian>(color(0.4, 0.2, 0.1));
    world.add(make_shared<sphere>(point3(-4, 1, 0), 1.0, material2));

    auto material3 = make_shared<metal>(color(0.7, 0.6, 0.5), 0.0);
    world.add(make_shared<sphere>(point3(4, 1, 0), 1.0, material3));
    // diff-add-end

    camera cam;

    // diff-add-start
    cam.aspect_ratio      = 16.0 / 9.0;
    cam.image_width       = 1200;
    cam.samples_per_pixel = 500;
    cam.max_depth         = 50;

    cam.vfov     = 20;
    cam.lookfrom = point3(13,2,3);
    cam.lookat   = point3(0,0,0);
    cam.vup      = vec3(0,1,0);

    cam.defocus_angle = 0.6;
    cam.focus_dist    = 10.0;
    // diff-add-end

    cam.render(world);
}
```

（请注意，上面的代码与项目示例代码略有不同：为了生成高质量图像，上面将 `samples_per_pixel` 设置为 500，因此渲染会花费相当长的时间。为了在开发和验证过程中保持合理的运行时间，项目源代码中使用的值是 10。）

最终会得到：

<img
  src="https://raytracing.github.io/images/img-1.23-book1-final.jpg"
  width="600"
/>

一个很有意思的现象是：你可能会注意到，这些玻璃球几乎没有阴影，因此看起来像是漂浮在空中。

这并不是程序错误。现实生活中我们并不经常看到玻璃球，而真实的玻璃球看起来同样会有些奇怪；尤其在阴天时，它们确实可能显得像漂浮起来一样。

玻璃球下方的大球表面仍然会接收到大量光线，因为玻璃球并没有直接遮挡天空的光线，而是通过折射改变了这些光线的传播方向。

# 14.2 后续步骤

现在你已经拥有一个很酷的光线追踪器了！接下来该做什么呢？

## 14.2.1 第二本书：《Ray Tracing: The Next Week》

本系列的第二本书建立在你刚刚开发的光线追踪器之上，并加入了以下新功能：

* **运动模糊**：更加真实地渲染运动中的物体。
* **包围体层次结构（BVH）**：加速复杂场景的渲染。
* **纹理映射**：将图像贴到物体表面。
* **Perlin 噪声**：一种随机噪声生成器，可用于许多不同的技术。
* **四边形**：除了球体之外，终于可以渲染其他东西了！它也是实现圆盘、三角形、圆环以及几乎所有其他二维图元的基础。
* **光源**：为场景添加光源。
* **变换**：用于放置和旋转物体。
* **体积渲染**：渲染烟雾、云朵以及其他气体体积。

## 14.2.2 第三本书：《Ray Tracing: The Rest of Your Life》

这本书会在第二本书的基础上进一步扩展。书中的大量内容都围绕着同时提高渲染图像的质量与渲染器的性能，并重点讨论如何生成正确的光线，以及如何恰当地累积这些光线的贡献。

这本书适合那些认真希望编写专业级光线追踪器的读者，也适合对实现次表面散射、嵌套介质等高级效果所需的基础知识感兴趣的读者。

## 14.2.3 其他方向

从这里开始，你还可以探索许多其他方向，其中也包括本系列尚未介绍的技术，例如：

**三角形**——大多数很酷的三维模型都以三角形网格的形式存在。模型的输入与输出处理是最麻烦的部分，几乎所有人都会设法找别人的代码来完成它。此外，你还需要高效处理由大量三角形组成的大型网格，而这本身也会带来不少挑战。

**并行化**——在 $N$ 个核心上运行 $N$ 份程序，并为它们使用不同的随机种子，然后对这 $N$ 次运行结果取平均。这个平均过程还可以分层完成：先对 $N/2$ 对图像分别求平均，得到 $N/4$ 张图像，然后继续对结果两两求平均。只需很少的代码，这种并行方式便可以很好地扩展到数千个核心。

**阴影光线**——通过向光源发射光线，可以准确判断某个特定点受到遮挡的程度。利用这种方法，你可以渲染清晰的硬阴影或柔和的软阴影，从而进一步增强场景的真实感。

祝你玩得开心，也请把你创作出的酷炫图像发给我看看！
