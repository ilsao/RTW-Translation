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