---
sidebar_position: 13
---

# 13. 散焦模糊

现在来介绍我们的最后一个功能：散焦模糊（defocus blur）。需要注意的是，摄影师通常把这种现象称为景深，所以在和你学光线追踪的朋友交流时，记得只使用 “散焦模糊” 这个说法。

真实相机之所以会产生散焦模糊，是因为它们需要通过一个较大的孔洞来收集光线，而不是只使用针孔。单纯使用一个大孔会让所有物体都失焦，但如果我们在胶片或传感器前放置一个透镜，那么就会存在某个特定距离，使得位于该距离上的所有物体都能够清晰成像。物体距离这个位置越远，看起来就会越模糊，并且模糊程度会随距离近似线性增加。

你可以这样理解透镜：所有从焦点距离处某一个特定点发出的光线，只要它们击中透镜，都会被透镜折射，并重新汇聚到图像传感器上的同一个点。

我们把相机中心到 “所有物体都能够完全对焦的平面” 之间的距离称为对焦距离。

需要注意，对焦距离通常并不等于焦距。焦距指的是相机中心到成像平面之间的距离。不过，在我们的模型中，这两个距离会取相同的值，因为我们会直接把像素网格放置在对焦平面上，而这个平面距离相机中心正好为对焦距离。

在真实相机中，对焦距离是通过调节透镜与胶片或传感器之间的距离来控制的。这也就是为什么当你改变对焦位置时，会看到镜头相对于相机机身发生移动。在手机相机中也可能发生类似的现象，不过有时移动的是传感器。

所谓的光圈（aperture），本质上是一个孔洞，用来控制透镜的有效大小。对于真实相机来说，如果需要接收更多光线，就必须增大光圈，但这样也会让偏离对焦距离的物体产生更明显的模糊。

对于我们的虚拟相机来说，我们假设传感器是完美的，所以永远不需要为了增加进光量而扩大光圈。我们只会在希望产生散焦模糊效果时使用光圈。

# 13.1 薄透镜近似

真实相机通常使用结构复杂的复合镜头。对于我们的代码，我们当然可以按照真实相机的顺序进行模拟：先放置传感器，然后是镜头，最后是光圈。接着，我们可以计算每条光线应该射向哪里，并在图像计算完成后将其翻转，因为图像实际上会以倒置的形式投影到胶片上。

不过，计算机图形学领域通常会使用一种更简单的模型，称为薄透镜近似（thin lens approximation）：

<img
  src="https://raytracing.github.io/images/fig-1.21-cam-lens.jpg"
  width="600"
/>

我们不需要模拟相机内部的任何结构——对于渲染相机外部的图像来说，这只会带来不必要的复杂度。

相反，我通常会让光线从一个无限薄的圆形“透镜”上的随机位置出发，并将它们射向对焦平面上我们感兴趣的像素位置。对焦平面距离透镜为 `focal_length`，三维世界中位于该平面上的所有物体都会清晰对焦。

在实际实现中，我们通过将视口放置在这个平面上来达到这一效果。综合起来，模型具有以下结构：

1. 对焦平面与相机的观察方向正交。
2. 对焦距离是相机中心与对焦平面之间的距离。
3. 视口位于对焦平面上，其中心处于相机观察方向向量所指的位置。
4. 像素位置组成的网格位于视口内部，而这个视口本身位于三维世界中。
5. 在当前像素位置周围的区域内，随机选择图像采样位置。
6. 相机从透镜上的随机位置发射光线，使光线穿过当前的图像采样位置。

<img
  src="https://raytracing.github.io/images/fig-1.22-cam-film-plane.jpg"
  width="600"
/>

# 13.2 生成采样光线

在没有散焦模糊时，场景中的所有光线都从相机中心（也就是 `lookfrom`）出发。

为了实现散焦模糊，我们在相机中心处构造一个圆盘。圆盘的半径越大，散焦模糊就越明显。你也可以把原本的相机理解为具有一个半径为零的散焦圆盘，此时完全没有模糊，因此所有光线都从圆盘中心，也就是 `lookfrom` 出发。

那么，散焦圆盘应该有多大呢？

由于这个圆盘的大小决定了散焦模糊的程度，因此它应该作为相机类的一个参数。我们可以直接把圆盘半径设为相机参数，但这样一来，模糊程度会随着投影距离的变化而改变。

一种稍微更方便的参数化方式，是指定一个圆锥的夹角。这个圆锥的顶点位于视口中心，底面则是位于相机中心的散焦圆盘。对于同一个拍摄场景，当你改变对焦距离时，这种方式通常可以得到更加一致的模糊效果。

由于我们需要从散焦圆盘中随机选取点，因此还需要一个函数来完成这件事：`random_in_unit_disk()`。这个函数采用的方法与 `random_unit_vector()` 类似，只不过它是在二维空间中进行随机采样。

```cpp
...

inline vec3 unit_vector(const vec3& u) {
    return v / v.length();
}

// diff-add-start
inline vec3 random_in_unit_disk() {
    while (true) {
        auto p = vec3(random_double(-1,1), random_double(-1,1), 0);
        if (p.length_squared() < 1)
            return p;
    }
}
// diff-add-end

...
```

现在，让我们更新相机，使射线从散焦圆盘上的采样点出发：

```cpp
class camera {
  public:
    double aspect_ratio      = 1.0;  // 图像宽度与高度的比值
    int    image_width       = 100;  // 渲染图像宽 (以像素为单位)
    int    samples_per_pixel = 10;   // 每个像素的随机采样数
    int    max_depth         = 10;   // 光线在场景中最多可反弹的次数

    double vfov     = 90;              // 垂直视角（视野角，Field of View）
    point3 lookfrom = point3(0,0,0);   // 相机所在的位置
    point3 lookat   = point3(0,0,-1);  // 相机所注视的目标点
    vec3   vup      = vec3(0,1,0);     // 相机的「向上」方向（相对于相机的 up 向量）

    // diff-add-start
    double defocus_angle = 0;  // Variation angle of rays through each pixel
    double focus_dist = 10;    // Distance from camera lookfrom point to plane of perfect focus
    // diff-add-end

    ...

  private:
    int    image_height;         // 渲染图像高
    double pixel_samples_scale;  // 像素样本总和的颜色缩放因子
    point3 center;               // 相机中心
    point3 pixel00_loc;          // 像素 0, 0 的位置
    vec3   pixel_delta_u;        // 像素到其右侧的偏移量
    vec3   pixel_delta_v;        // 像素到其下的偏移量
    vec3   u, v, w;              // 相机坐标系的基底向量
    // diff-add-start
    vec3   defocus_disk_u;       // Defocus disk horizontal radius
    vec3   defocus_disk_v;       // Defocus disk vertical radius
    // diff-add-end

    void initialize() {
        image_height = int(image_width / aspect_ratio);
        image_height = (image_height < 1) ? 1 : image_height;

        pixel_samples_scale = 1.0 / samples_per_pixel;

        center = lookfrom;

        // 确定视口维度。
        // diff-remove-start
        auto focal_length = (lookfrom - lookat).length();
        // diff-remove-end
        auto theta = degrees_to_radians(vfov);
        auto h = std::tan(theta/2);
        // diff-add-start
        auto viewport_height = 2 * h * focus_dist;
        // diff-add-end
        auto viewport_width = viewport_height * (double(image_width)/image_height);

        // 计算相机坐标系的 u、v、w 单位基底向量。
        w = unit_vector(lookfrom - lookat);
        u = unit_vector(cross(vup, w));
        v = cross(w, u);

        // 计算视口水平边缘和垂直边缘的向量
        vec3 viewport_u = viewport_width * u;    // 视口水平边缘向量
        vec3 viewport_v = viewport_height * -v;  // 视口垂直边缘向量（向下）

        // 计算相邻像素间水平和垂直间距向量
        pixel_delta_u = viewport_u / image_width;
        pixel_delta_v = viewport_v / image_height;

        // 计算左上角像素的位置
        // diff-add-start
        auto viewport_upper_left = center - (focus_dist * w) - viewport_u/2 - viewport_v/2;
        // diff-add-end
        pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);

        // diff-add-start
        // Calculate the camera defocus disk basis vectors.
        auto defocus_radius = focus_dist * std::tan(degrees_to_radians(defocus_angle / 2));
        defocus_disk_u = u * defocus_radius;
        defocus_disk_v = v * defocus_radius;
        // diff-add-end
    }

    ray get_ray(int i, int j) const {
        // diff-add-start
        // Construct a camera ray originating from the defocus disk and directed at a randomly
        // sampled point around the pixel location i, j.
        // diff-add-end

        auto offset = sample_square();
        auto pixel_sample = pixel00_loc
                          + ((i + offset.x()) * pixel_delta_u)
                          + ((j + offset.y()) * pixel_delta_v);

        // diff-add-start
        auto ray_origin = (defocus_angle <= 0) ? center : defocus_disk_sample();
        // diff-add-end
        auto ray_direction = pixel_sample - ray_origin;

        return ray(ray_origin, ray_direction);
    }

    vec3 sample_square() const {
        ...
    }

    // diff-add-start
    point3 defocus_disk_sample() const {
        // Returns a random point in the camera defocus disk.
        auto p = random_in_unit_disk();
        return center + (p[0] * defocus_disk_u) + (p[1] * defocus_disk_v);
    }
    // diff-add-end

    color ray_color(const ray& r, int depth, const hittable& world) const {
        ...
    }
};
```

使用较大的光圈：

```cpp
int main() {
    ...

    camera cam;

    cam.aspect_ratio      = 16.0 / 9.0;
    cam.image_width       = 400;
    cam.samples_per_pixel = 100;
    cam.max_depth         = 50;

    cam.vfov     = 20;
    cam.lookfrom = point3(-2,2,1);
    cam.lookat   = point3(0,0,-1);
    cam.vup      = vec3(0,1,0);

    // diff-add-start
    cam.defocus_angle = 10.0;
    cam.focus_dist    = 3.4;
    // diff-add-end

    cam.render(world);
}
```

可得：

<img
  src="https://raytracing.github.io/images/img-1.22-depth-of-field.png"
  width="600"
/>
