---
sidebar_position: 9
---

# 9. 漫反射材质

现在我们已经有了物体，以及每个像素发射多条射线，因此可以开始制作一些看起来更真实的材质。我们先从漫反射材质 (Diffuse Materials，也称雾面材质 ) 开始。其中一个设计上的问题是：我们是否要将几何与材质分离，使一个材质可以指定给多个球体，反之亦然；还是要让几何与材质紧密绑定在一起。后者对于程序化生成的物体特别有用，因为它们的几何形状与材质通常是一起产生、彼此关联的。本文将采用分离式设计，也就是将几何与材质分开管理。这是大多数渲染器常见的做法，不过也请注意，实际上还存在其他不同的设计方式。

# 9.1 一个简易漫反射材质

不自行发光的漫反射物体只会呈现周遭环境的颜色，但也会用自身的固有颜色对其进行调变。从漫反射表面反射出的光，其方向会被随机化；因此，如果我们向两个漫反射表面之间的缝隙射入三条射线，它们各自都会有不同的随机行为：

<img
  src="https://raytracing.github.io/images/fig-1.09-light-bounce.jpg"
  width="600"
/>

它们可能被吸收，而不是被反射。表面越暗，光线就越有可能被吸收（这就是它看起来很暗的原因！）。其实任何可随机方向的算法，都能产生看起来像雾面材质的表面。我们先从最直观的方法开始：让表面把光线随机、均匀地反弹到所有方向。对于这种材质，一条击中表面的射线，会以相同的机率朝着远离表面的任意方向反弹。

<img
  src="https://raytracing.github.io/images/fig-1.10-random-vec-horizon.jpg"
  width="600"
/>

这种非常直观的材质是最简单的一种漫反射材质；事实上，许多早期的光线追踪论文都使用了这种漫反射方法，后来才改采用更准确的方法，也就是我们稍后会实现的那种方法。我们目前还没有办法随机反射一条射线，所以需要在向量工具头文件中加入几个函数。首先，我们需要能够产生任意的随机向量：

```cpp
class vec3 {
  public:
    ...

    double length_squared() const {
        return e[0]*e[0] + e[1]*e[1] + e[2]*e[2];
    }

    // diff-add-start
    static vec3 random() {
        return vec3(random_double(), random_double(), random_double());
    }

    static vec3 random(double min, double max) {
        return vec3(random_double(min,max), random_double(min,max), random_double(min,max));
    }
    // diff-add-end
};
```

然后，我们要想办法操作一个随机向量，让最终结果只会落在半球的表面上。这件事的确有解析方法可以做到，但那些方法意外地难懂，而且实现起来也相当麻烦。相比之下，我们通常会使用最简单的算法：拒绝采样法 (A rejection method) 。拒绝采样法的做法是：反覆产生随机样本，直到产生一个符合我们需求条件的样本为止。换句话说，就是一直拒绝不好的样本，直到找到一个好的样本。

使用拒绝采样法在半球上产生随机向量，有很多种同样合理的方法；不过就我们的目的而言，我们会采用最简单的一种，也就是：

1. 在单位球内产生一个随机向量
2. 将这个向量正则化，使其延伸到球面上
3. 若正则化后的向量落在错误的半球上，则将其反转

首先，我们会使用拒绝采样法，在单位球内产生随机向量，也就是半径为 1 的球。做法是：在包住单位球的立方体内随机选一个点，也就是让 $x$、$y$ 和 $z$ 都落在 $[-1,+1]$ 的范围内。如果这个点位于单位球外面，那就重新产生一个新的点，直到找到一个位于单位球内部或球面上的点为止。

<img
  src="https://raytracing.github.io/images/fig-1.11-sphere-vec.jpg"
  width="600"
/>

<img
  src="https://raytracing.github.io/images/fig-1.12-sphere-unit-vec.jpg"
  width="600"
/>

下面是我们第一版的代码：

```cpp
...

inline vec3 unit_vector(const vec3& v) {
    return v / v.length();
}

// diff-add-start
inline vec3 random_unit_vector() {
    while (true) {
        auto p = vec3::random(-1,1);
        auto lensq = p.length_squared();
        if (lensq <= 1)
            return p / sqrt(lensq);
    }
}
// diff-add-end
```

遗憾的是，我们还需要处理一个小小的浮点数抽象泄漏问题。由于浮点数的精度是有限的，一个非常小的数在平方之后可能会下溢为零。因此，如果三个坐标都足够小，也就是非常靠近球心，那么这个向量的范数就会变成零，而对它进行归一化就会得到一个错误的向量 $[\pm\infty, \pm\infty, \pm\infty]$。

为了解决这个问题，我们也会拒绝掉那些落在球心附近这个“黑洞”区域内的点。使用双精度浮点数，也就是 64 位浮点数时，我们可以安全地支持大于 $10^{-160}$ 的值。

下面是我们更健壮的函数：

```cpp
inline vec3 random_unit_vector() {
    while (true) {
        auto p = vec3::random(-1,1);
        auto lensq = p.length_squared();
        // diff-add-start
        if (1e-160 < lensq && lensq <= 1)
        // diff-add-end
            return p / sqrt(lensq);
    }
}
```

现在我们有了一个随机向量，我们可以通过与表面法线比对，来判断该随机向量是否落在正确的半球：

<img
  src="https://raytracing.github.io/images/fig-1.13-surface-normal.jpg"
  width="600"
/>

我们可以计算表面法线和随机向量的点积，来判断这个随机向量是否位于正确的半球内。如果点积为正，那么这个向量就在正确的半球内；如果点积为负，那么我们就需要把这个向量反转。

```cpp
...

inline vec3 random_unit_vector() {
    while (true) {
        auto p = vec3::random(-1,1);
        auto lensq = p.length_squared();
        if (1e-160 < lensq && lensq <= 1)
            return p / sqrt(lensq);
    }
}

// diff-add-start
inline vec3 random_on_hemisphere(const vec3& normal) {
    vec3 on_unit_sphere = random_unit_vector();
    if (dot(on_unit_sphere, normal) > 0.0) // 与法线在相同半球
        return on_unit_sphere;
    else
        return -on_unit_sphere;
}
// diff-add-end
```

如果一条光线从某种材质上反弹之后，保留了它 $100%$ 的颜色，那么我们称这种材质为白色。如果一条光线从某种材质上反弹之后，保留了它 $0%$ 的颜色，那么我们称这种材质为黑色。

作为我们新的漫反射材质的第一个演示，我们会让 `ray_color` 函数返回一次反弹后颜色的 $50%$。这样我们应该会得到一个漂亮的灰色。

```cpp
class camera {
  ...
  private:
    ...
    color ray_color(const ray& r, const hittable& world) const {
        hit_record rec;

        if (world.hit(r, interval(0, infinity), rec)) {
            // diff-add-start
            vec3 direction = random_on_hemisphere(rec.normal);
            return 0.5 * ray_color(ray(rec.p, direction), world);
            // diff-add-end
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

... 的确，我们得到了一个相对不错的灰色球体：

<img
  src="https://raytracing.github.io/images/img-1.07-first-diffuse.png"
  width="600"
/>

# 9.2 限制子射线的数量

这里潜藏着一个问题。注意，`ray_color` 函数是递归的。那它什么时候会停止递归呢？当射线没有击中任何物体的时候。然而在某些情况下，这可能会持续很久，久到足以导致栈溢出。为了避免这种情况，我们来限制最大递归深度：当达到最大深度时，就不再返回任何光照贡献。

```cpp
class camera {
  public:
    double aspect_ratio      = 1.0;  // 图像宽度与高度的比值
    int    image_width       = 100;  // 渲染图像宽 (以像素为单位)
    int    samples_per_pixel = 10;   // 每个像素的随机采样数
    // diff-add-start
    int    max_depth         = 10;   // 射线在场景中弹射次数上限
    // diff-add-end

    void render(const hittable& world) {
        initialize();

        std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

        for (int j = 0; j < image_height; j++) {
            std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
            for (int i = 0; i < image_width; i++) {
                color pixel_color(0,0,0);
                for (int sample = 0; sample < samples_per_pixel; sample++) {
                    ray r = get_ray(i, j);
                    // diff-add-start
                    pixel_color += ray_color(r, max_depth, world);
                    // diff-add-end
                }
                write_color(std::cout, pixel_samples_scale * pixel_color);
            }
        }

        std::clog << "\rDone.                 \n";
    }
    ...
  private:
    ...
    // diff-add-start
    color ray_color(const ray& r, int depth, const hittable& world) const {
        // 若超出射线弹射上限，不再收集更多光线
        if (depth <= 0)
            return color(0,0,0);
    // diff-add-end

        hit_record rec;

        if (world.hit(r, interval(0, infinity), rec)) {
            vec3 direction = random_on_hemisphere(rec.normal);
            // diff-add-start
            return 0.5 * ray_color(ray(rec.p, direction), depth-1, world);
            // diff-add-end
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

更新 `main()` 函数以采用新的深度限制：

```cpp
int main() {
    ...

    camera cam;

    cam.aspect_ratio      = 16.0 / 9.0;
    cam.image_width       = 400;
    cam.samples_per_pixel = 100;
    // diff-add-start
    cam.max_depth         = 50;
    // diff-add-end

    cam.render(world);
}
```

对于这个非常简单的场景，我们应该得到一个几乎一致的结果：

<img
  src="https://raytracing.github.io/images/img-1.08-second-diffuse.png"
  width="600"
/>

# 解决阴影痤疮问题

还有一个需要我们处理的微妙 bug。当一条光线与某个表面相交时，它会尝试精确地计算交点。不幸的是，这个计算很容易受到浮点数舍入误差的影响，导致算出来的交点出现极其微小的偏差。这意味着下一条光线的起点，也就是从表面随机散射出去的那条光线的起点，不太可能和表面完全贴合。它可能刚好在表面上方，也可能刚好在表面下方。如果这条光线的起点刚好在表面下方，那么它就可能再次与这个表面相交。也就是说，它会在 $t = 0.00000001$，或者 hit 函数给出的某个浮点近似值处，找到最近的表面。解决这个问题最简单的 hack，就是直接忽略那些离计算出的交点非常近的命中结果：

```cpp
class camera {
  ...
  private:
    ...
    color ray_color(const ray& r, int depth, const hittable& world) const {
        // 若超出射线弹射上限，不再收集更多光线
        if (depth <= 0)
            return color(0,0,0);

        hit_record rec;

        // diff-add-start
        if (world.hit(r, interval(0.001, infinity), rec)) {
            // diff-add-end
            vec3 direction = random_on_hemisphere(rec.normal);
            return 0.5 * ray_color(ray(rec.p, direction), depth-1, world);
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

这解决了暗影痔疮 (Shadow Acne) 的问题。对，就是这么叫。以下是结果：

<img
  src="https://raytracing.github.io/images/img-1.09-no-acne.png"
  width="600"
/>

将反射光线均匀地散射到整个半球上，可以产生一种不错的柔和漫反射模型，但我们当然还能做得更好。对真实漫反射物体来说，一个更准确的表示是 Lambertian 分布。这个分布会按照与 $\cos(\phi)$ 成正比的方式散射反射光线，其中 $\phi$ 是反射光线和表面法线之间的夹角。这意味着，反射光线最有可能朝接近表面法线的方向散射，而朝远离法线方向散射的概率较低。相比之前的均匀散射，这种非均匀的 Lambertian 分布能更好地模拟现实世界中的材质反射。

我们可以通过把一个随机单位向量加到法线向量上，来生成这种分布。在表面的交点处，有命中点 $\mathbf{p}$，也有表面的法线 $\mathbf{n}$。在这个交点处，表面恰好有两侧，所以对于任意交点来说，只可能有两个唯一的单位球与其相切，也就是表面的每一侧各有一个唯一的单位球。这两个单位球会沿着半径长度从表面偏移；对于单位球来说，这个半径长度正好是 1。

其中一个球会沿着表面法线 $(\mathbf{n})$ 的方向偏移，另一个球会沿着相反方向 $(-\mathbf{n})$ 偏移。这样我们就得到两个单位大小的球，它们都只是在交点处刚好接触表面。由此可知，一个球的中心在 $(\mathbf{P} + \mathbf{n})$，另一个球的中心在 $(\mathbf{P} - \mathbf{n})$。中心位于 $(\mathbf{P} - \mathbf{n})$ 的球被认为在表面内部，而中心位于 $(\mathbf{P} + \mathbf{n})$ 的球被认为在表面外部。

我们想选择那个与光线起点位于表面同一侧的相切单位球。在这个单位半径球上随机选取一个点 $$\mathbf{S}$$，然后从命中点 $\mathbf{P}$ 向随机点 $\mathbf{S}$ 发射一条光线，也就是向量 $(\mathbf{S} - \mathbf{P})$：

<img
  src="https://raytracing.github.io/images/fig-1.14-rand-unitvec.jpg"
  width="600"
/>

这个修改其实非常小：

```cpp
class camera {
    ...
    color ray_color(const ray& r, int depth, const hittable& world) const {
        // 若超出射线弹射上限，不再收集更多光线
        if (depth <= 0)
            return color(0,0,0);

        hit_record rec;

        if (world.hit(r, interval(0.001, infinity), rec)) {
            // diff-add-start
            vec3 direction = rec.normal + random_unit_vector();
            // diff-add-end
            return 0.5 * ray_color(ray(rec.p, direction), depth-1, world);
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

渲染后我们会得到一个类似的图像：

<img
  src="https://raytracing.github.io/images/img-1.10-correct-lambertian.png"
  width="600"
/>

在我们的场景中只有两个球，实在太简单了，所以很难看出这两种漫反射方法之间的差异。不过，你应该能够注意到两个重要的视觉差别：

1. 修改之后，阴影更加明显了。
2. 修改之后，两个球都受天空影响而带上了蓝色调。

这两个变化都是因为光线散射变得不那么均匀了——更多光线会朝法线方向散射。这意味着对于漫反射物体来说，它们看起来会更暗，因为反弹到相机方向的光线变少了。对于阴影来说，更多光线会直接向上反弹，所以球体下方的区域会更暗。

日常生活中并没有太多物体是完全漫反射的，所以我们对于这些物体在光照下会如何表现，视觉直觉可能并不太可靠。随着本书中的场景逐渐变得更加复杂，建议你在这里介绍的不同漫反射渲染器之间切换比较。大多数值得关注的场景都会包含大量漫反射材质。通过理解不同漫反射方法会如何影响场景中的光照，你可以获得很有价值的洞察。

# 9.5 使用 Gamma 校正来获得准确的颜色强度

注意球体下方的阴影。图片非常暗，但我们的球体在每次反弹时只吸收一半的能量，所以它们是 50% 的反射体。球体本应该看起来相当明亮（在现实生活中，会是浅灰色），但它们现在看起来却相当暗。如果我们完整地遍历一下漫反射材质的亮度范围，就能更清楚地看到这一点。我们先把 `ray_color` 函数中的反射率从 `0.5`（50%）改成 `0.1`（10%）：

```cpp
class camera {
    ...
    color ray_color(const ray& r, int depth, const hittable& world) const {
        // 若超出射线弹射上限，不再收集更多光线
        if (depth <= 0)
            return color(0,0,0);

        hit_record rec;

        if (world.hit(r, interval(0.001, infinity), rec)) {
            vec3 direction = rec.normal + random_unit_vector();
            // diff-add-start
            return 0.1 * ray_color(ray(rec.p, direction), depth-1, world);
            // diff-add-end
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

我们用新的 10% 反射率渲染出图像。接着把反射率设为 30%，再渲染一次。然后对 50%、70%，最后 90% 重复同样的操作。你可以在自己喜欢的图片编辑器中，把这些图像从左到右叠放起来，这样应该会得到一幅非常清晰的视觉表示，展示你所选择的亮度范围是如何逐渐变亮的。下面这张就是我们目前一直在使用的结果：

<img
  src="https://raytracing.github.io/images/img-1.11-linear-gamut.png"
  width="600"
/>

如果你仔细观察，或者使用取色器，就会注意到 50% 反射率的渲染结果，也就是中间那张图，远远没有达到白色和黑色之间一半的位置，也就是中灰色。事实上，70% 反射率的结果反而更接近中灰色。原因在于，几乎所有计算机程序都会假设一张图像在写入图像文件之前已经经过了 “gamma 校正” 。这意味着，0 到 1 之间的数值在被存储为字节之前，会先经过某种变换。没有经过变换就写入数据的图像，被称为处于线性空间 (linear space)；而经过变换后的图像，则被称为处于 gamma 空间。你正在使用的图像查看器很可能期望接收到的是一张 gamma 空间中的图像，但我们给它的却是一张线性空间中的图像。这就是为什么我们的图像看起来会不准确地偏暗。

图像应该存储在 gamma 空间中有很多充分理由，不过对我们来说，只需要意识到这一点就够了。我们将把数据转换到 gamma 空间，这样图像查看器就能更准确地显示我们的图像。作为一个简单近似，我们可以使用 “gamma 2” 作为变换。这个 gamma 值是在从 gamma 空间转换到线性空间时使用的幂次。现在我们需要从线性空间转换到 gamma 空间，也就是取 “gamma 2” 的倒数；这意味着指数为 $1 / gamma$，也就是开平方。我们还需要确保能够稳健地处理负输入。

```cpp
// diff-add-start
inline double linear_to_gamma(double linear_component)
{
    if (linear_component > 0)
        return std::sqrt(linear_component);

    return 0;
}
// diff-add-end

void write_color(std::ostream& out, const color& pixel_color) {
    auto r = pixel_color.x();
    auto g = pixel_color.y();
    auto b = pixel_color.z();

    // 为 gamma 2 应用一个线性变换到 gamma
    r = linear_to_gamma(r);
    g = linear_to_gamma(g);
    b = linear_to_gamma(b);

    // diff-add-start
    // 将 [0, 1] 分量转换到一个字节的范围 [0, 255]
    static const interval intensity(0.000, 0.999);
    int rbyte = int(256 * intensity.clamp(r));
    int gbyte = int(256 * intensity.clamp(g));
    int bbyte = int(256 * intensity.clamp(b));
    // diff-add-end

    // 写出像素颜色分量
    out << rbyte << ' ' << gbyte << ' ' << bbyte << '\n';
}
```

使用 gamma 矫正，我们现在会得到一个从暗到亮更加一致的渐变效果：

<img
  src="https://raytracing.github.io/images/img-1.12-gamma-gamut.png"
  width="600"
/>
