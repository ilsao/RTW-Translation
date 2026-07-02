---
sidebar_position: 3
---

# 3. `vec3` 類

几乎所有图形程序都有一些储存几何向量与颜色的类。在很多系统中，这些向量是四维的(几何中的三维位置加上一个齐次座标，或颜色中的 RGB 加上一个 Alpha 透明度分量)。为了我们的目的，三维座标便足够。我们将使用一样的 `vec3` 类来表示颜色、位置、方向、偏移量等等。有些人不喜欢这样，因为这无法预防你做出一些傻事，比如用一个颜色向量减去一个位置向量。他们的论点很对，但当没有明显错误时，我们将永远选择"代码最少"的路径。即便如此，我们的确为 `vec3` 声明了两个别名: `point3` 和 `color` 。由于这两种类型只是 `vec3` 的别名，如果你传递一个 `color` 给一个需要 `point3` 的函数，你不会得到任何警告，并且没有任何事会阻止你将 `point3` 和 `color` 相加，但这让代码更容易被阅读和理解一些。

我们在一个新的 `vec3.h` 头文件的上半部定义 `vec3` ，并在下半部定义一组有用的向量工具函数:

```cpp
#ifndef VEC3_H
#define VEC3_H

#include <cmath>
#include <iostream>

class vec3 {
  public:
    double e[3];

    vec3() : e{0,0,0} {}
    vec3(double e0, double e1, double e2) : e{e0, e1, e2} {}

    double x() const { return e[0]; }
    double y() const { return e[1]; }
    double z() const { return e[2]; }

    vec3 operator-() const { return vec3(-e[0], -e[1], -e[2]); }
    double operator[](int i) const { return e[i]; }
    double& operator[](int i) { return e[i]; }

    vec3& operator+=(const vec3& v) {
        e[0] += v.e[0];
        e[1] += v.e[1];
        e[2] += v.e[2];
        return *this;
    }

    vec3& operator*=(double t) {
        e[0] *= t;
        e[1] *= t;
        e[2] *= t;
        return *this;
    }

    vec3& operator/=(double t) {
        return *this *= 1/t;
    }

    double length() const {
        return std::sqrt(length_squared());
    }

    double length_squared() const {
        return e[0]*e[0] + e[1]*e[1] + e[2]*e[2];
    }
};

// point3 只是 vec3 的别名，但在代码中使几何意义更明确.
using point3 = vec3;


// 向量工具函数

inline std::ostream& operator<<(std::ostream& out, const vec3& v) {
    return out << v.e[0] << ' ' << v.e[1] << ' ' << v.e[2];
}

inline vec3 operator+(const vec3& u, const vec3& v) {
    return vec3(u.e[0] + v.e[0], u.e[1] + v.e[1], u.e[2] + v.e[2]);
}

inline vec3 operator-(const vec3& u, const vec3& v) {
    return vec3(u.e[0] - v.e[0], u.e[1] - v.e[1], u.e[2] - v.e[2]);
}

inline vec3 operator*(const vec3& u, const vec3& v) {
    return vec3(u.e[0] * v.e[0], u.e[1] * v.e[1], u.e[2] * v.e[2]);
}

inline vec3 operator*(double t, const vec3& v) {
    return vec3(t*v.e[0], t*v.e[1], t*v.e[2]);
}

inline vec3 operator*(const vec3& v, double t) {
    return t * v;
}

inline vec3 operator/(const vec3& v, double t) {
    return (1/t) * v;
}

inline double dot(const vec3& u, const vec3& v) {
    return u.e[0] * v.e[0]
         + u.e[1] * v.e[1]
         + u.e[2] * v.e[2];
}

inline vec3 cross(const vec3& u, const vec3& v) {
    return vec3(u.e[1] * v.e[2] - u.e[2] * v.e[1],
                u.e[2] * v.e[0] - u.e[0] * v.e[2],
                u.e[0] * v.e[1] - u.e[1] * v.e[0]);
}

inline vec3 unit_vector(const vec3& v) {
    return v / v.length();
}

#endif
```

在这里我们使用 `double` ，但有些光线追踪器使用 `float` 。 `double` 有更高的精确度和更大的范围，但它佔的内存的大小是 `float` 的两倍。如果你的程序内存受限(比如硬件着色器)，大小的增加可能很重要。两者都很好--随你自己的喜好。

# 3.1 颜色工具函数

利用我们的新创建的 `vec3` 类，我们会创建一个新的 `color.h` 头文件，并定义一个颜色工具函数，用以将单个像素的颜色写入标准输出流中。

```cpp
#indef COLOR_H
#define COLOR_H

#include "vec3.h"

#include <iostream>

using color = vec3;

void write_color(std::ostream& out, const color& pixel_color) {
    auto r = pixel_color.x();
    auto g = pixel_color.y();
    auto b = pixel_color.z();

    // 将 [0,1] 的分量值转换为 [0,255] 的字节范围
    int rbyte = int(255.999 * r);
    int gbyte = int(255.999 * g);
    int bbyte = int(255.999 * b);

    // 输出像素的颜色分量
    out << rbyte << ' ' << gbyte << ' ' << bbyte << '\n';
}

#endif
```

现在我们可以修改我们的 `main` 函数，同时使用这两者：

```cpp
// diff-add-start
#include "color.h"
#include "vec3.h"
// diff-add-end

#include <iostream>

int main() {

    // 图片

    int image_width = 256;
    int image_height = 256;

    // 渲染

    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

    for (int j = 0; j < image_height; j++) {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; i++) {
            // diff-add-start
            auto pixel_color = color(double(i)/(image_width-1), double(j)/(image_height-1), 0);
            write_color(std::cout, pixel_color);
            // diff-add-end
        }
    }

    std::clog << "\rDone.                 \n";
}
```

你会得到和之前完全相同的一张图片。
