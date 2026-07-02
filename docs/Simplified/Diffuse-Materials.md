---
sidebar_position: 9
---

# 9. 漫反射材質

現在我們已經有了物體，以及每個像素發射多條射線，因此可以開始製作一些看起來更真實的材質。我們先從漫反射材質 (Diffuse Materials，也稱霧面材質 ) 開始。其中一個設計上的問題是：我們是否要將幾何與材質分離，使一個材質可以指定給多個球體，反之亦然；還是要讓幾何與材質緊密綁定在一起。後者對於程序化生成的物體特別有用，因為它們的幾何形狀與材質通常是一起產生、彼此關聯的。本書將採用分離式設計，也就是將幾何與材質分開管理。這是大多數渲染器常見的做法，不過也請注意，實際上還存在其他不同的設計方式。

# 9.1 一個簡易漫反射材質

不自行發光的漫反射物體只會呈現周遭環境的顏色，但也會用自身的固有顏色對其進行調變。從漫反射表面反射出的光，其方向會被隨機化；因此，如果我們向兩個漫反射表面之間的縫隙射入三條射線，它們各自都會有不同的隨機行為：

<img
  src="https://raytracing.github.io/images/fig-1.09-light-bounce.jpg"
  width="600"
/>

它們可能被吸收，而不是被反射。表面越暗，光線就越有可能被吸收（這就是它看起來很暗的原因！）。其實任何可隨機方向的算法，都能產生看起來像霧面材質的表面。我們先從最直觀的方法開始：讓表面把光線隨機、均勻地反彈到所有方向。對於這種材質，一條擊中表面的射線，會以相同的機率朝著遠離表面的任意方向反彈。

<img
  src="https://raytracing.github.io/images/fig-1.10-random-vec-horizon.jpg"
  width="600"
/>

這種非常直觀的材質是最簡單的一種漫反射材質；事實上，許多早期的光線追蹤論文都使用了這種漫反射方法，後來才改採用更準確的方法，也就是我們稍後會實現的那種方法。我們目前還沒有辦法隨機反射一條射線，所以需要在向量工具頭文件中加入幾個函數。首先，我們需要能夠產生任意的隨機向量：

```cpp
class vec3 {
  public:
    ...

    double length_squared() const {
        return e[0]*e[0] + e[1]*e[1] + e[2]*e[2];
    }

    // highlight-start
    static vec3 random() {
        return vec3(random_double(), random_double(), random_double());
    }

    static vec3 random(double min, double max) {
        return vec3(random_double(min,max), random_double(min,max), random_double(min,max));
    }
    // highlight-end
};
```

然後，我們要想辦法操作一個隨機向量，讓最終結果只會落在半球的表面上。這件事的確有解析方法可以做到，但那些方法意外地難懂，而且實現起來也相當麻煩。相比之下，我們通常會使用最簡單的算法：拒絕採樣法 (A rejection method) 。拒絕採樣法的做法是：反覆產生隨機樣本，直到產生一個符合我們需求條件的樣本為止。換句話說，就是一直拒絕不好的樣本，直到找到一個好的樣本。

使用拒絕採樣法在半球上產生隨機向量，有很多種同樣合理的方法；不過就我們的目的而言，我們會採用最簡單的一種，也就是：

1. 在單位球內產生一個隨機向量
2. 將這個向量正則化，使其延伸到球面上
3. 若正則化後的向量落在錯誤的半球上，則將其反轉

首先，我們會使用拒絕採樣法，在單位球內產生隨機向量，也就是半徑為 1 的球。做法是：在包住單位球的立方體內隨機選一個點，也就是讓 $x$、$y$ 和 $z$ 都落在 $[-1,+1]$ 的範圍內。如果這個點位於單位球外面，那就重新產生一個新的點，直到找到一個位於單位球內部或球面上的點為止。

<img
  src="https://raytracing.github.io/images/fig-1.11-sphere-vec.jpg"
  width="600"
/>

<img
  src="https://raytracing.github.io/images/fig-1.12-sphere-unit-vec.jpg"
  width="600"
/>

下面是我們第一版的代碼：

```cpp
...

inline vec3 unit_vector(const vec3& v) {
    return v / v.length();
}

// highlight-start
inline vec3 random_unit_vector() {
    while (true) {
        auto p = vec3::random(-1,1);
        auto lensq = p.length_squared();
        if (lensq <= 1)
            return p / sqrt(lensq);
    }
}
// highlight-end
```

遗憾的是，我们还需要处理一个小小的浮点数抽象泄漏问题。由于浮点数的精度是有限的，一个非常小的数在平方之后可能会下溢为零。因此，如果三个坐标都足够小，也就是非常靠近球心，那么这个向量的范数就会变成零，而对它进行归一化就会得到一个错误的向量 $[\pm\infty, \pm\infty, \pm\infty]$。

为了解决这个问题，我们也会拒绝掉那些落在球心附近这个“黑洞”区域内的点。使用双精度浮点数，也就是 64 位浮点数时，我们可以安全地支持大于 $10^{-160}$ 的值。

下面是我们更健壮的函数：

```cpp
inline vec3 random_unit_vector() {
    while (true) {
        auto p = vec3::random(-1,1);
        auto lensq = p.length_squared();
        // highlight-start
        if (1e-160 < lensq && lensq <= 1)
        // highlight-end
            return p / sqrt(lensq);
    }
}
```

現在我們有了一個隨機向量，我們可以通過與表面法線比對，來判斷該隨機向量是否落在正確的半球：

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

// highlight-start
inline vec3 random_on_hemisphere(const vec3& normal) {
    vec3 on_unit_sphere = random_unit_vector();
    if (dot(on_unit_sphere, normal) > 0.0) // In the same hemisphere as the normal
        return on_unit_sphere;
    else
        return -on_unit_sphere;
}
// highlight-end
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
            // highlight-start
            vec3 direction = random_on_hemisphere(rec.normal);
            return 0.5 * ray_color(ray(rec.p, direction), world);
            // highlight-end
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

... 的確，我們得到了一個相對不錯的灰色球體：

<img
  src="https://raytracing.github.io/images/img-1.07-first-diffuse.png"
  width="600"
/>

# 9.2 限制子射線的數量

这里潜藏着一个问题。注意，`ray_color` 函数是递归的。那它什么时候会停止递归呢？当射线没有击中任何物体的时候。然而在某些情况下，这可能会持续很久，久到足以导致栈溢出。为了避免这种情况，我们来限制最大递归深度：当达到最大深度时，就不再返回任何光照贡献。

```cpp
class camera {
  public:
    double aspect_ratio      = 1.0;  // Ratio of image width over height
    int    image_width       = 100;  // Rendered image width in pixel count
    int    samples_per_pixel = 10;   // Count of random samples for each pixel
    // highlight-start
    int    max_depth         = 10;   // Maximum number of ray bounces into scene
    // highlight-end

    void render(const hittable& world) {
        initialize();

        std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

        for (int j = 0; j < image_height; j++) {
            std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
            for (int i = 0; i < image_width; i++) {
                color pixel_color(0,0,0);
                for (int sample = 0; sample < samples_per_pixel; sample++) {
                    ray r = get_ray(i, j);
                    // highlight-start
                    pixel_color += ray_color(r, max_depth, world);
                    // highlight-end
                }
                write_color(std::cout, pixel_samples_scale * pixel_color);
            }
        }

        std::clog << "\rDone.                 \n";
    }
    ...
  private:
    ...
    // highlight-start
    color ray_color(const ray& r, int depth, const hittable& world) const {
        // If we've exceeded the ray bounce limit, no more light is gathered.
        if (depth <= 0)
            return color(0,0,0);
    // highlight-end

        hit_record rec;

        if (world.hit(r, interval(0, infinity), rec)) {
            vec3 direction = random_on_hemisphere(rec.normal);
            // highlight-start
            return 0.5 * ray_color(ray(rec.p, direction), depth-1, world);
            // highlight-end
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

更新 `main()` 函數以採用新的深度限制：

```cpp
int main() {
    ...

    camera cam;

    cam.aspect_ratio      = 16.0 / 9.0;
    cam.image_width       = 400;
    cam.samples_per_pixel = 100;
    // highlight-start
    cam.max_depth         = 50;
    // highlight-end

    cam.render(world);
}
```

對於這個非常簡單的場景，我們應該得到一個幾乎一致的結果：

<img
  src="https://raytracing.github.io/images/img-1.08-second-diffuse.png"
  width="600"
/>

# 解決阴影痤疮问题

还有一个需要我们处理的微妙 bug。当一条光线与某个表面相交时，它会尝试精确地计算交点。不幸的是，这个计算很容易受到浮点数舍入误差的影响，导致算出来的交点出现极其微小的偏差。这意味着下一条光线的起点，也就是从表面随机散射出去的那条光线的起点，不太可能和表面完全贴合。它可能刚好在表面上方，也可能刚好在表面下方。如果这条光线的起点刚好在表面下方，那么它就可能再次与这个表面相交。也就是说，它会在 $t = 0.00000001$，或者 hit 函数给出的某个浮点近似值处，找到最近的表面。解决这个问题最简单的 hack，就是直接忽略那些离计算出的交点非常近的命中结果：

```cpp
class camera {
  ...
  private:
    ...
    color ray_color(const ray& r, int depth, const hittable& world) const {
        // If we've exceeded the ray bounce limit, no more light is gathered.
        if (depth <= 0)
            return color(0,0,0);

        hit_record rec;

        // highlight-start
        if (world.hit(r, interval(0.001, infinity), rec)) {
            // highlight-end
            vec3 direction = random_on_hemisphere(rec.normal);
            return 0.5 * ray_color(ray(rec.p, direction), depth-1, world);
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

這解決了暗影痔瘡 (Shadow Acne) 的問題。對，就是這麼叫。以下是結果：

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

這個修改其實非常小：

```cpp
class camera {
    ...
    color ray_color(const ray& r, int depth, const hittable& world) const {
        // If we've exceeded the ray bounce limit, no more light is gathered.
        if (depth <= 0)
            return color(0,0,0);

        hit_record rec;

        if (world.hit(r, interval(0.001, infinity), rec)) {
            // highlight-start
            vec3 direction = rec.normal + random_unit_vector();
            // highlight-end
            return 0.5 * ray_color(ray(rec.p, direction), depth-1, world);
        }

        vec3 unit_direction = unit_vector(r.direction());
        auto a = 0.5*(unit_direction.y() + 1.0);
        return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
    }
};
```

渲染後我們會得到一個類似的圖像：

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
        // If we've exceeded the ray bounce limit, no more light is gathered.
        if (depth <= 0)
            return color(0,0,0);

        hit_record rec;

        if (world.hit(r, interval(0.001, infinity), rec)) {
            vec3 direction = rec.normal + random_unit_vector();
            // highlight-start
            return 0.1 * ray_color(ray(rec.p, direction), depth-1, world);
            // highlight-end
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
// highlight-start
inline double linear_to_gamma(double linear_component)
{
    if (linear_component > 0)
        return std::sqrt(linear_component);

    return 0;
}
// highlight-end

void write_color(std::ostream& out, const color& pixel_color) {
    auto r = pixel_color.x();
    auto g = pixel_color.y();
    auto b = pixel_color.z();

    // Apply a linear to gamma transform for gamma 2
    r = linear_to_gamma(r);
    g = linear_to_gamma(g);
    b = linear_to_gamma(b);

    // highlight-start
    // Translate the [0,1] component values to the byte range [0,255].
    static const interval intensity(0.000, 0.999);
    int rbyte = int(256 * intensity.clamp(r));
    int gbyte = int(256 * intensity.clamp(g));
    int bbyte = int(256 * intensity.clamp(b));
    // highlight-end

    // Write out the pixel color components.
    out << rbyte << ' ' << gbyte << ' ' << bbyte << '\n';
}
```

使用 gamma 矯正，我们现在会得到一个从暗到亮更加一致的渐变效果：

<img
  src="https://raytracing.github.io/images/img-1.12-gamma-gamut.png"
  width="600"
/>