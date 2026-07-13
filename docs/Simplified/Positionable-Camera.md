---
sidebar_position: 12
---

# 12. 可定位相机

相机和电介质一样，都很难调试，所以我总是用循序渐进的方式来开发相机。首先，让我们允许视场角（fov）可调。视场角是渲染图像从一侧边缘到另一侧边缘所对应的视觉角度。由于我们的图像不是正方形，水平视场角和垂直视场角并不相同，我总是用垂直视场角。我也习惯用角度来指定它，并在构造函数内部将其转换为弧度——只是个人偏好。

# 12.1 相机观察几何

首先，我们仍然让射线从原点出发，并射向 $z=−1$ 平面。我们也可以把它设为 $z=−2$ 平面，或任何其他位置，只要将 $h$ 设置为相对于该距离的比例即可。下面是我们的设置：

<img
  src="https://raytracing.github.io/images/fig-1.18-cam-view-geom.jpg"
  width="600"
/>

这意味着 $h=\tan(\frac{\theta}{2})$。我们的相机现在变为：

``` cpp
class camera {
  public:
    double aspect_ratio      = 1.0;  // 影像宽高比（影像宽度与高度的比例）
    int    image_width       = 100;  // 渲染影像的宽度（以像素数表示）
    int    samples_per_pixel = 10;   // 每个像素的随机采样次数
    int    max_depth         = 10;   // 光线在场景中最多可反弹的次数

    double vfov = 90;  // 垂直视角

    void render(const hittable& world) {
    ...

  private:
    ...

    void initialize() {
        image_height = int(image_width / aspect_ratio);
        image_height = (image_height < 1) ? 1 : image_height;

        pixel_samples_scale = 1.0 / samples_per_pixel;

        center = point3(0, 0, 0);

        // 计算视口的尺寸。
        auto focal_length = 1.0;
        auto theta = degrees_to_radians(vfov);
        auto h = std::tan(theta/2);
        auto viewport_height = 2 * h * focal_length;
        auto viewport_width = viewport_height * (double(image_width)/image_height);

        // 计算视口水平与垂直边缘所对应的向量。
        auto viewport_u = vec3(viewport_width, 0, 0);
        auto viewport_v = vec3(0, -viewport_height, 0);

        // 计算相邻像素之间在水平与垂直方向上的位移向量。
        pixel_delta_u = viewport_u / image_width;
        pixel_delta_v = viewport_v / image_height;

        // 计算左上角第一个像素中心的位置。
        auto viewport_upper_left =
            center - vec3(0, 0, focal_length) - viewport_u/2 - viewport_v/2;
        pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);
    }

    ...
};
```

我们将使用一个由两个相互接触的球体组成的简单场景来测试这些修改，并将视野角设置为 $90^{\circ}$。

``` cpp
int main() {
    hittable_list world;

    // diff-add-start
    auto R = std::cos(pi/4);

    auto material_left  = make_shared<lambertian>(color(0,0,1));
    auto material_right = make_shared<lambertian>(color(1,0,0));

    world.add(make_shared<sphere>(point3(-R, 0, -1), R, material_left));
    world.add(make_shared<sphere>(point3( R, 0, -1), R, material_right));
    // diff-add-end

    camera cam;

    cam.aspect_ratio      = 16.0 / 9.0;
    cam.image_width       = 400;
    cam.samples_per_pixel = 100;
    cam.max_depth         = 50;

    // diff-add-start
    cam.vfov = 90;
    // diff-add-end

    cam.render(world);
}
```

这将给出以下渲染：

<img
  src="https://raytracing.github.io/images/img-1.19-wide-view.png"
  width="600"
/>

# 12.2 相机的位置与朝向

为了能够从任意视角观察场景，我们首先为几个重要的位置命名。我们将相机所在的位置称为 `lookfrom`，而相机所注视的目标点称为 `lookat`。（之后如果需要，也可以改成定义观看方向而不是观看目标点。）

我们还需要一种方式来指定相机的滚转（roll），也就是相机向左右倾斜的角度：它表示相机绕着 `lookat` 与 `lookfrom` 连线所形成的轴进行旋转。另一种理解方式是，即使你固定了 `lookfrom` 和 `lookat`，你仍然可以像把头绕着鼻子前方的方向旋转一样，改变头部的倾斜角度。因此，我们需要一种方法来为相机指定一个「向上（up）」向量。

<img
  src="https://raytracing.github.io/images/fig-1.19-cam-view-dir.jpg"
  width="600"
/>

我们可以指定任何想要的向上向量，只要它不与视线方向平行即可。将这个向上向量投影到与视线方向垂直的平面上，便可得到一个相对于相机的向上向量。我采用常见的惯例，将这个向量命名为「view up（vup）」向量。经过几次叉积运算与向量正则化之后，我们便得到一组完整的正交归一基底 $(u,v,w)$，用来描述相机的朝向。$u$ 是指向相机右方的单位向量，$v$ 是指向相机上方的单位向量，$w$ 是指向与视线方向相反的单位向量（因为我们采用右手座标系），而相机中心位于原点。

<img
  src="https://raytracing.github.io/images/fig-1.20-cam-view-up.jpg"
  width="600"
/>

和之前一样，当我们固定的相机朝向 $−Z$ 时，现在这个任意视角的相机则朝向 $−w$。请记住，我们可以——但不是一定要——使用世界座标中的向上方向 $(0,1,0)$ 作为 vup。这样做很方便，而且会自然地让相机保持水平，直到你决定尝试一些疯狂的相机角度为止。

```cpp
class camera {
  public:
    double aspect_ratio      = 1.0;  // 影像宽高比（影像宽度与高度的比例）
    int    image_width       = 100;  // 渲染影像的宽度（以像素数表示）
    int    samples_per_pixel = 10;   // 每个像素的随机采样次数
    int    max_depth         = 10;   // 光线在场景中最多可反弹的次数

    double vfov     = 90;              // 垂直视角（视野角，Field of View）
    // diff-add-start
    point3 lookfrom = point3(0,0,0);   // 相机所在的位置
    point3 lookat   = point3(0,0,-1);  // 相机所注视的目标点
    vec3   vup      = vec3(0,1,0);     // 相机的「向上」方向（相对于相机的 up 向量）
    // diff-add-end

...

  private:
    int    image_height;         // 渲染影像的高度
    double pixel_samples_scale;  // 像素采样总和的颜色缩放系数
    point3 center;               // 相机中心
    point3 pixel00_loc;          // (0, 0) 像素中心的位置
    vec3   pixel_delta_u;        // 向右相邻一个像素的位移向量
    vec3   pixel_delta_v;        // 向下相邻一个像素的位移向量
    // diff-add-start
    vec3   u, v, w;              // 相机座标系的基底向量
    // diff-add-end


    void initialize() {
        ...

        // diff-add-start
        center = lookfrom;
        // diff-add-end

        // 计算视口的尺寸。
        auto focal_length = (lookfrom - lookat).length();
        ...

        // 计算相机座标系的 u、v、w 单位基底向量。
        // diff-add-start
        w = unit_vector(lookfrom - lookat);
        // diff-add-end
        u = unit_vector(cross(vup, w));
        v = cross(w, u);

        // diff-add-start
        // 计算视口水平与垂直边缘所对应的向量。
        vec3 viewport_u = viewport_width * u;    // 视口水平边缘方向向量
        vec3 viewport_v = viewport_height * -v;  // 视口垂直边缘方向向量（向下）
        // diff-add-end

        // 计算相邻像素之间在水平与垂直方向上的位移向量。
        pixel_delta_u = viewport_u / image_width;
        pixel_delta_v = viewport_v / image_height;

        // 计算左上角第一个像素中心的位置。
        // diff-add-start
        auto viewport_upper_left = center - (focal_length * w) - viewport_u/2 - viewport_v/2;
        // diff-add-end
        pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);
    }

...

  private:
};

```

我们会切回之前的场景，并使用新的视口：

``` cpp
int main() {
    hittable_list world;

    // diff-add-start
    auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
    auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
    auto material_left   = make_shared<dielectric>(1.50);
    auto material_bubble = make_shared<dielectric>(1.00 / 1.50);
    auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);

    world.add(make_shared<sphere>(point3( 0.0, -100.5, -1.0), 100.0, material_ground));
    world.add(make_shared<sphere>(point3( 0.0,    0.0, -1.2),   0.5, material_center));
    world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.5, material_left));
    world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.4, material_bubble));
    world.add(make_shared<sphere>(point3( 1.0,    0.0, -1.0),   0.5, material_right));
    // diff-add-end

    camera cam;

    cam.aspect_ratio      = 16.0 / 9.0;
    cam.image_width       = 400;
    cam.samples_per_pixel = 100;
    cam.max_depth         = 50;


    cam.vfov     = 90;
    // diff-add-start
    cam.lookfrom = point3(-2,2,1);
    cam.lookat   = point3(0,0,-1);
    cam.vup      = vec3(0,1,0);
    // diff-add-end

    cam.render(world);
}
```

得到：

<img
  src="https://raytracing.github.io/images/img-1.20-view-distant.png"
  width="600"
/>

并且我们可以改变 fov：

```cpp
    cam.vfov    = 20;
```

得到：

<img
  src="https://raytracing.github.io/images/img-1.21-view-zoom.png"
  width="600"
/>
