---
sidebar_position: 2
---

# 輸出圖像

# PPM 圖像格式

當你剛開始編寫渲染器時，你應該想一個方法來顯示圖像。最直接的辦法是將圖像寫入文件，但是圖像文件格式實在是太多了。許多格式都非常複雜，所以我通常使用純文本格式的 ppm 格式。這是 ppm 格式的 Wkipedia 介紹：

![圖 1：PPM 釋例](https://raytracing.github.io/images/fig-1.01-ppm.jpg)

讓我們寫點 C++ 代碼來輸出該格式：

```cpp
#include <iostream>

int main() {

    // 圖像

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

代碼中需要注意幾項：
- 像素按行寫入。
- 每行像素從左到右寫入。
- 行從上到下被寫入。
- 按照慣例，所有 紅/綠/藍 分量都被表示為介於 0.0 到 1.0 之間的實數變量。在我們將顏色顯示前，我們必須先將值拉伸到 0 到 255 間的整數。
- 紅色左至右從全暗(黑)到全亮(鮮紅)，綠色上至下從全暗(黑)到全亮(鮮綠)。將紅與綠光相加會得到黃色，所以我們應該可以在右下角觀察到黃色。

# 創建一個圖像文件

因為目前文件被寫入標準輸出流，所以你必須重定向其到一個圖像文件中。通常這通過在命令行中輸入 `>` 重定向操作符實現。

在 Windows 中，你可以通過運行下列代碼從 CMake 中構建 debug 版本。

```sh
cmake -B build
cmake --build build
```

如下使用你新構建的程序：

```sh
build\Debug\inOneWeekend.exe > image.ppm
```

之後為了提速，最好使用優化構建。在這種情況下，你應該如下構建：

```sh
cmake --build build --config release
```

並如下運行程序：

```sh
build\Release\inOneWeekend.exe > image.ppm
```

上例假設你使用 CMake 構建源碼中的 `CMakeLists.txt`。但你可以使用任意你熟悉的構建系統 (與語言)。

在 Mac 或 Linux 中，relese 版本如下運行：

```sh
build/inOneWeekend > image.ppm
```

完整的構建與運行步驟可以在 [項目 README](https://raytracing.github.io/README.md) 中找到。

打開輸出文件 (我在 Mac 上使用 `ToyViewer` 打開，但你可以用你喜歡的圖像查看器打開。若你的察看器不支持此格式，去 Google "pmm viewer") 會輸出此結果：

![圖 1：第一張 ppm 圖像](https://raytracing.github.io/images/img-1.01-first-ppm-image.png)

太好了！這是圖形學中的 "hello world"。若你的圖像與上圖不符，用文本編輯器打開輸出文件，看看文件內容。文件的起始應該如下：

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

若你的 PPM 文件不符，再次確認你格式化部分的代碼。若它**的確**符合，但無法正確顯示，那可能是由於行尾格式不同之類的問題而解析錯誤。為了幫助調試此問題，你可以在 GitHub 項目中找到 `images/test.ppm` 文件。這應該可以幫助確認你的察看器的確可以顯示 PPM 格式，並幫助對比你生成的 PPM 文件。

一些讀者回應道，在瀏覽 Windows 上生成的圖像時遇到了問題。這種情況通常由於 PowerShell 輸出時使用 UTF-16 編碼導致。若你遇到此問題，請到 [討論 1114](https://github.com/RayTracing/raytracing.github.io/discussions/1114) 查看解決方法。

若一切順利，那你幾乎解決了所有系統與 IDE 相關的問題。此系列剩餘部分皆使用相同方式生成渲染圖像。

若你想生成其他圖像格式，我是 `stb_image.h` 的紛絲。它是一個僅有 `.h` 的庫，你可以在 GitHub 上找到它：[https://github.com/nothings/stb](https://github.com/nothings/stb)。

# 添加一個進度顯示器

在我們繼續之前，讓我們在輸出處添加一個進度顯示器。當長時間渲染時，這是個方便的察看進度方法。而且，這還可以幫你確認程序是否因為無限循環或其他問題導致卡死。

我們的程序輸出到標準輸出流 (`std::cout`)，為了保持輸出乾淨，我們將日志寫到日志輸出流 (`std::clog`)。

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

現在當你運行時即可看見剩餘掃描行數量，希望你的程序跑的快到你幾乎看不到它。別擔心，當未來我們增進光線追蹤器時，你會有大把時間看到進度慢慢更新。