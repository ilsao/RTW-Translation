---
sidebar_position: 2
---

# 输出图像

# PPM 图像格式

当你刚开始编写渲染器时，你应该想一个方法来显示图像。最直接的办法是将图像写入文件，但是图像文件格式实在是太多了。许多格式都非常复杂，所以我通常使用纯文本格式的 ppm 格式。这是 ppm 格式的 Wikipedia 介绍：

![图 1：PPM 示例](https://raytracing.github.io/images/fig-1.01-ppm.jpg)

让我们写点 C++ 代码来输出该格式：

```cpp
#include <iostream>

int main() {

    // 图像

    int image_width = 256;
    int image_height = 256;

    // 渲染

    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

    for (int j = 0; j < image_height; j++) {
        for (int i = 0; i < image_width; i++) {
            auto r = double(i) / (image_width-1);
            auto g = double(j) / (image_height-1);
            auto b = 0.0;

            int ir = int(255.999 * r);
            int ig = int(255.999 * g);
            int ib = int(255.999 * b);

            std::cout << ir << ' ' << ig << ' ' << ib << '\n';
        }
    }
}
```

代码中需要注意几项：
- 像素按行写入。
- 每行像素从左到右写入。
- 行从上到下被写入。
- 按照惯例，所有 红/绿/蓝 分量都被表示为介于 0.0 到 1.0 之间的实数变量。在我们将颜色显示前，我们必须先将值拉伸到 0 到 255 间的整数。
- 红色左至右从全暗(黑)到全亮(鲜红)，绿色上至下从全暗(黑)到全亮(鲜绿)。将红与绿光相加会得到黄色，所以我们应该可以在右下角观察到黄色。

# 创建一个图像文件

因为目前文件被写入标准输出流，所以你必须重定向其到一个图像文件中。通常这通过在命令行中输入 `>` 重定向操作符实现。

在 Windows 中，你可以通过运行下列代码从 CMake 中构建 debug 版本。

```sh
cmake -B build
cmake --build build
```

如下使用你新构建的程序：

```sh
build\Debug\inOneWeekend.exe > image.ppm
```

之后为了提速，最好使用优化构建。在这种情况下，你应该如下构建：

```sh
cmake --build build --config release
```

并如下运行程序：

```sh
build\Release\inOneWeekend.exe > image.ppm
```

上例假设你使用 CMake 构建源码中的 `CMakeLists.txt`。但你可以使用任意你熟悉的构建系统 (与语言)。

在 Mac 或 Linux 中，release 版本如下运行：

```sh
build/inOneWeekend > image.ppm
```

完整的构建与运行步骤可以在 [项目 README](https://raytracing.github.io/README.md) 中找到。

打开输出文件 (我在 Mac 上使用 `ToyViewer` 打开，但你可以用你喜欢的图像查看器打开。若你的查看器不支持此格式，去 Google "ppm viewer") 会输出此结果：

![图 1：第一张 ppm 图像](https://raytracing.github.io/images/img-1.01-first-ppm-image.png)

太好了！这是图形学中的 "hello world"。若你的图像与上图不符，用文本编辑器打开输出文件，看看文件内容。文件的起始应该如下：

```text
P3
256 256
255
0 0 0
1 0 0
2 0 0
3 0 0
4 0 0
5 0 0
6 0 0
7 0 0
8 0 0
9 0 0
10 0 0
11 0 0
12 0 0
...
```

若你的 PPM 文件不符，再次确认你格式化部分的代码。若它**的确**符合，但无法正确显示，那可能是由于行尾格式不同之类的问题而解析错误。为了帮助调试此问题，你可以在 GitHub 项目中找到 `images/test.ppm` 文件。这应该可以帮助确认你的查看器的确可以显示 PPM 格式，并帮助对比你生成的 PPM 文件。

一些读者回应道，在浏览 Windows 上生成的图像时遇到了问题。这种情况通常由于 PowerShell 输出时使用 UTF-16 编码导致。若你遇到此问题，请到 [讨论 1114](https://github.com/RayTracing/raytracing.github.io/discussions/1114) 查看解决方法。

若一切顺利，那你几乎解决了所有系统与 IDE 相关的问题。此系列剩余部分皆使用相同方式生成渲染图像。

若你想生成其他图像格式，我是 `stb_image.h` 的粉丝。它是一个仅有 `.h` 的库，你可以在 GitHub 上找到它：[https://github.com/nothings/stb](https://github.com/nothings/stb)。

# 添加一个进度显示器

在我们继续之前，让我们在输出处添加一个进度显示器。当长时间渲染时，这是个方便的察看进度方法。而且，这还可以帮你确认程序是否因为无限循环或其他问题导致卡死。

我们的程序输出到标准输出流 (`std::cout`)，为了保持输出干净，我们将日志写到日志输出流 (`std::clog`)。

```cpp
    for (int j = 0; j < image_height; ++j) {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; i++) {
            auto r = double(i) / (image_width-1);
            auto g = double(j) / (image_height-1);
            auto b = 0.0;

            int ir = int(255.999 * r);
            int ig = int(255.999 * g);
            int ib = int(255.999 * b);

            std::cout << ir << ' ' << ig << ' ' << ib << '\n';
        }
    }

    std::clog << "\rDone.                 \n";
```

现在当你运行时即可看见剩余扫描行数量，希望你的程序跑的快到你几乎看不到它。别担心，当未来我们增进光线追踪器时，你会有大把时间看到进度慢慢更新。