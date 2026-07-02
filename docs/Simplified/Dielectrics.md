---
sidebar_position: 11
---

# 11. 介電質

像水、玻璃和钻石这样的透明材料都是介电质。当一条光线打到它们时，光线会分成一条反射光线和一条折射（透射）光线。我们会通过在反射和折射之间随机选择来处理这件事，也就是每次相互作用只生成一条散射光线。

先快速复习一下术语：反射光线会打到表面，然后“弹开”到新的方向。

折射光线则会在从材料周围环境进入材料本身时发生弯折，例如进入玻璃或水中。这就是为什么铅笔部分插入水中时，看起来会像是弯掉了一样。

折射光线弯折的程度由材料的折射率决定。一般来说，折射率是一个单一数值，用来描述光从真空进入某种材料时会弯折多少。玻璃的折射率大约是 1.5 到 1.7，钻石大约是 2.4，而空气的折射率很小，约为 1.000293。

当一个透明材料被放在另一个不同的透明材料中时，可以用相对折射率来描述折射：也就是物体材料的折射率除以周围材料的折射率。例如，如果你想渲染水中的玻璃球，那么玻璃球的有效折射率会是 1.125。这是由玻璃的折射率 1.5 除以水的折射率 1.333 得到的。

你可以通过快速的网络搜索，找到大多数常见材料的折射率。

# 11.1 折射

最难调试的部分是折射光线。我通常会先让所有光线都发生折射，只要存在折射光线就这么做。对于这个项目，我试着在场景中放入两个玻璃球，然后得到了下面这样的结果（我还没告诉你这样做是对还是错，不过很快就会讲到！）：

<img
  src="https://raytracing.github.io/images/img-1.15-glass-first.png"
  width="600"
/>

这样对吗？玻璃球在现实生活中看起来确实有点怪。但不，这并不对。整个世界应该上下颠倒，而且不应该有那些奇怪的黑色东西。我只是把穿过图像正中央的光线打印出来看了一下，结果明显是错的。这招通常就很管用。

# 11.2 斯涅尔定律

折射可以用斯涅尔定律来描述：

$$
\eta \cdot \sin\theta = \eta' \cdot \sin\theta'
$$

其中，$\theta$ 和 $\theta'$ 是相对于法线的角度，而 $\eta$ 和 $\eta'$ 读作 “eta” 和 “eta prime”，表示折射率。几何关系如下：

<img
  src="https://raytracing.github.io/images/fig-1.17-refraction.jpg"
  width="600"
/>

为了确定折射光线的方向，我们需要解出 $\sin\theta'$：

$$
\sin\theta' = \frac{\eta}{\eta'} \cdot \sin\theta
$$

在表面的折射侧，有一条折射光线 $\mathbf{R}'$ 和一条法线 $\mathbf{n}'$，它们之间存在一个角度 $\theta'$。我们可以把 $\mathbf{R}'$ 分解成垂直于 $\mathbf{n}'$ 和平行于 $\mathbf{n}'$ 的两个部分：

$$
\mathbf{R}' = \mathbf{R}'*\perp + \mathbf{R}'*\parallel
$$

如果我们解出 $\mathbf{R}'*\perp$ 和 $\mathbf{R}'*\parallel$，就会得到：

$$
\mathbf{R}'_\perp = \frac{\eta}{\eta'}(\mathbf{R} + |\mathbf{R}|\cos(\theta)\mathbf{n})
$$

$$
\mathbf{R}'*\parallel = -\sqrt{1 - |\mathbf{R}'*\perp|^2}\mathbf{n}
$$

如果你愿意，可以自己继续证明这个式子，但我们会先把它当成事实继续往下走。本书后面的内容不要求你理解这个证明。

我们知道右边每一项的值，除了 $\cos\theta$ 之外。众所周知，两个向量的点积可以用它们之间夹角的余弦来解释：

$$
\mathbf{a} \cdot \mathbf{b} = |\mathbf{a}||\mathbf{b}|\cos\theta
$$

如果我们限制 $\mathbf{a}$ 和 $\mathbf{b}$ 都是单位向量：

$$
\mathbf{a} \cdot \mathbf{b} = \cos\theta
$$

现在我们可以用已知量重写 $\mathbf{R}'_\perp$：

$$
\mathbf{R}'_\perp = \frac{\eta}{\eta'}(\mathbf{R} + (-\mathbf{R} \cdot \mathbf{n})\mathbf{n})
$$

当我们把这些部分重新组合起来，就可以写一个函数来计算 $\mathbf{R}'$：

```cpp
...

inline vec3 reflect(const vec3& v, const vec3& n) {
    return v - 2*dot(v,n)*n;
}

// diff-add-start
inline vec3 refract(const vec3& uv, const vec3& n, double etai_over_etat) {
    auto cos_theta = std::fmin(dot(-uv, n), 1.0);
    vec3 r_out_perp =  etai_over_etat * (uv + cos_theta*n);
    vec3 r_out_parallel = -std::sqrt(std::fabs(1.0 - r_out_perp.length_squared())) * n;
    return r_out_perp + r_out_parallel;
}
// diff-add-end
```

而总是折射的介电材质如下：

```cpp
...

class metal : public material {
    ...
};

// diff-add-start
class dielectric : public material {
  public:
    dielectric(double refraction_index) : refraction_index(refraction_index) {}

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
    const override {
        attenuation = color(1.0, 1.0, 1.0);
        double ri = rec.front_face ? (1.0/refraction_index) : refraction_index;

        vec3 unit_direction = unit_vector(r_in.direction());
        vec3 refracted = refract(unit_direction, rec.normal, ri);

        scattered = ray(rec.p, refracted);
        return true;
    }

  private:
    // 真空或空气中的折射率，或者材料折射率与包围介质折射率的比值
    double refraction_index;
};
// diff-add-end
```

现在，我们将更新场景来展示折射效果：把左边的球改成玻璃材质，玻璃的折射率大约是 1.5。

```cpp
auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
// diff-add-start
auto material_left   = make_shared<dielectric>(1.50);
// diff-add-end
auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);
```

其給出以下結果：

<img
  src="https://raytracing.github.io/images/img-1.16-glass-always-refract.png"
  width="600"
/>

# 11.3 全内反射

折射中有一个比较麻烦的实际问题：对于某些光线角度，用斯涅尔定律是得不到解的。当一条光线以足够贴近表面的掠射角进入折射率较低的介质时，它可能会以大于 $90^\circ$ 的角度发生折射。如果我们回到斯涅尔定律以及 $\sin\theta'$ 的推导：

$$
\sin\theta' = \frac{\eta}{\eta'} \cdot \sin\theta
$$

如果光线在玻璃内部，而外部是空气，也就是 $\eta = 1.5$ 且 $\eta' = 1.0$：

$$
\sin\theta' = \frac{1.5}{1.0} \cdot \sin\theta
$$

$\sin\theta'$ 的值不可能大于 1。所以，如果

$$
\frac{1.5}{1.0} \cdot \sin\theta > 1.0
$$

方程两边的相等关系就被破坏了，也就不存在解。如果不存在解，玻璃就不能发生折射，因此必须反射这条光线：

```cpp
if (ri * sin_theta > 1.0) {
    // 必須折射
    ...
} else {
    // 可以折射
    ...
}
```

这里所有光线都会被反射，而且因为在实际中这种情况通常发生在实体物体内部，所以它被称为全内反射 (total internal relfection)。这就是为什么有时候，当你潜在水下时，水和空气的交界面会像一面完美的镜子：如果你在水下向上看，可以看到水面上的东西；但当你靠近水面并侧向看时，水面看起来就像镜子一样。

我们可以用三角恒等式求出 `sin_theta`：

$$
\sin\theta = \sqrt{1 - \cos^2\theta}
$$

以及：

$$
\cos\theta = \mathbf{R} \cdot \mathbf{n}
$$

```cpp
double cos_theta = std::fmin(dot(-unit_direction, rec.normal), 1.0);
double sin_theta = std::sqrt(1.0 - cos_theta*cos_theta);

if (ri * sin_theta > 1.0) {
    // 必須折射
    ...
} else {
    // 可以折射
    ...
}
```

且这个总是会发生折射的介电材质（在可能折射的情况下）為：

```cpp
class dielectric : public material {
  public:
    dielectric(double refraction_index) : refraction_index(refraction_index) {}

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
    const override {
        attenuation = color(1.0, 1.0, 1.0);
        double ri = rec.front_face ? (1.0/refraction_index) : refraction_index;

        vec3 unit_direction = unit_vector(r_in.direction());
        // diff-add-start
        double cos_theta = std::fmin(dot(-unit_direction, rec.normal), 1.0);
        double sin_theta = std::sqrt(1.0 - cos_theta*cos_theta);

        bool cannot_refract = ri * sin_theta > 1.0;
        vec3 direction;

        if (cannot_refract)
            direction = reflect(unit_direction, rec.normal);
        else
            direction = refract(unit_direction, rec.normal, ri);

        scattered = ray(rec.p, direction);
        // diff-add-end
        return true;
    }

  private:
    // 真空或空气中的折射率，或者材料折射率与包围介质折射率的比值
    double refraction_index;
};
```

衰减始终为 1 —— 玻璃表面不会吸收任何东西。

如果我们用新的 `dielectric::scatter()` 函数渲染之前的场景，会发现……没有变化。嗯？

其实原因是：对于一个折射率大于空气的球体材料来说，不存在任何入射角会产生全内反射——无论是在光线进入球体的位置，还是在光线离开球体的位置都不会。这是由球体的几何形状导致的，因为一条掠射入射的光线总会先被弯折到一个更小的角度，然后在离开时又被弯回原本的角度。

那么我们要怎样展示全内反射呢？如果球体的折射率小于它所在介质的折射率，那么我们就可以用较浅的掠射角打到它，从而得到全外反射。这样应该足够观察到这个效果了。

我们会建模一个充满水的世界，水的折射率大约是 1.33，然后把球体材质改成空气，空气的折射率是 1.00 —— 也就是一个气泡！为此，需要把左边球体材质的折射率改成：

$$
\frac{\text{空气的折射率}}{\text{水的折射率}}
$$

```cpp
auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
// diff-add-start
auto material_left   = make_shared<dielectric>(1.00 / 1.33);
// diff-add-end
auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);
```

這個改變給出以下渲染圖：

<img
  src="https://raytracing.github.io/images/img-1.17-air-bubble-total-reflection.png"
  width="600"
/>

在这里可以看到，或多或少接近直射的光线会折射，而擦着表面掠过的光线会反射。

# 11.4 Schlick 近似

现实中的玻璃具有随角度变化的反射率——以很陡的角度看窗户时，它就会变得像一面镜子。描述这个现象有一个很大、很丑的方程，但几乎所有人都会使用 Christophe Schlick 提出的一个便宜而且出奇准确的多项式近似。这样我们就得到了完整的玻璃材质：

```cpp
class dielectric : public material {
  public:
    dielectric(double refraction_index) : refraction_index(refraction_index) {}

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
    const override {
        attenuation = color(1.0, 1.0, 1.0);
        double ri = rec.front_face ? (1.0/refraction_index) : refraction_index;

        vec3 unit_direction = unit_vector(r_in.direction());
        double cos_theta = std::fmin(dot(-unit_direction, rec.normal), 1.0);
        double sin_theta = std::sqrt(1.0 - cos_theta*cos_theta);

        bool cannot_refract = ri * sin_theta > 1.0;
        vec3 direction;

        // diff-add-start
        if (cannot_refract || reflectance(cos_theta, ri) > random_double())
        // diff-add-end
            direction = reflect(unit_direction, rec.normal);
        else
            direction = refract(unit_direction, rec.normal, ri);

        scattered = ray(rec.p, direction);
        return true;
    }

  private:
    // 真空或空气中的折射率，或者材料折射率与包围介质折射率的比值
    double refraction_index;

    // diff-add-start
    static double reflectance(double cosine, double refraction_index) {
        // 使用 Schlick 近似來計算折射率
        auto r0 = (1 - refraction_index) / (1 + refraction_index);
        r0 = r0*r0;
        return r0 + (1-r0)*std::pow((1 - cosine),5);
    }
    // diff-add-end
};
```

# 11.5 建模空心玻璃球

我们来建模一个空心玻璃球。它是一个具有一定厚度的球体，内部还有另一个由空气组成的球。如果你思考一条光线穿过这种物体的路径，它会先打到外层球体，发生折射；然后打到内层球体（假设确实打到了），再次折射，并在内部空气中传播。接着它会继续前进，打到内层球体的内侧表面，折射回去，然后打到外层球体的内侧表面，最后再次折射，退出并回到场景的大气中。

外层球体可以直接用标准的玻璃球建模，折射率大约为 1.50，也就是模拟光线从外部空气进入玻璃。内层球体稍微有点不同，因为它的折射率应该相对于周围外层球体的材料来定义，也就是模拟从玻璃进入内部空气的过渡。

这其实很容易指定，因为介电材质中的 `refraction_index` 参数可以被解释为：物体材料的折射率除以包围介质的折射率。在这个例子中，内层球体的折射率就是空气的折射率（内层球体材料）除以玻璃的折射率（包围介质），也就是：$1.00 / 1.50 = 0.67$

代码如下：

```cpp
...
auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
// diff-add-start
auto material_left   = make_shared<dielectric>(1.50);
auto material_bubble = make_shared<dielectric>(1.00 / 1.50);
// diff-add-end
auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 0.0);

world.add(make_shared<sphere>(point3( 0.0, -100.5, -1.0), 100.0, material_ground));
world.add(make_shared<sphere>(point3( 0.0,    0.0, -1.2),   0.5, material_center));
world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.5, material_left));
// diff-add-start
world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.4, material_bubble));
// diff-add-end
world.add(make_shared<sphere>(point3( 1.0,    0.0, -1.0),   0.5, material_right));
...
```

以下是結果：

<img
  src="https://raytracing.github.io/images/img-1.18-glass-hollow.png"
  width="600"
/>