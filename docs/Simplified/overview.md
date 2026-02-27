---
sidebar_position: 1
---

# 概览

我已经教授好几年图形学了。我通常通过实现一个光线追踪器来进行课程，因为这迫使学生自己写下所有代码。就算不调用 API，你仍可做出非常酷的图片。我决定将课程写成一个实战教程，带你尽快做出一个很酷的程序。虽然它不会是一个全功能光线追踪器，但其仍实现了间接光照 (间接光照使得光追成为电影中的常用技术)。跟随步骤，你会做出一个可拓展性极高的光线追踪器框架。若你对实现一个更进阶的光线追踪器感到兴趣，它应该非常容易改动。

当某人说 "光线追踪" ，可能意味着不同意思。从技术层面看，我接下来介绍的其实是一种非常通用的路径追踪器。虽然接下来的代码不会很难 (让计算机替我们计算！)，但我觉得你们一定会对其产出的图像感到惊喜。

我会带你从头编写一个光线追踪器，并在其中穿插一些调试技巧。最终，你能得到一个产出很赞图像的光线追踪器！你应该能在一个周末内完成此项目，但若多花了些时间，也别太气馁。虽然我使用 C++ 作为编写语言，但你可以选择另一个语言实现。但我仍建议你使用 C++，因为其高速、可移植，且大部分电影与游戏的渲染器都用的 C++ 实现。注意到我避免使用许多 C++ 的 "现代特性"，但继承与操作符重载对实现光线追踪器非常有帮助，我们不应该弃用它们。

> 我不会把代码放到网上，不过书里的代码都是真实且可运行的。除了 `vec3` 类中一些显而易见的操作，我几乎都写进了文中。我始终坚信，想真正学会写代码，就得自己动手写一遍。不过若有现成代码，我会直接用。所以，我只有在没有源码的情况下，才能真正学习到。所以，别找我要代码！

我把上段话留着，因为我现在的做法与当初完全相反，挺好笑的。一些读者在实现中遇到了些小错误，相信通过对比代码可以帮助找出问题。请自己写代码，但你可以在 [GitHub 上的 RayTracing 项目](https://github.com/RayTracing/raytracing.github.io/)找到本书最终的源码。

对于书中实现的代码，我们优先考虑以下几点：
- 代码应实现书中的概念。
- 虽然我们使用 C++，但保持尽可能的简单。我们的编码风格与 C 非常相近，但我们仍使用一些能令代码更易读的现代化特性。
- 我们的编码风格会尽可能的保持与最初的书籍一致，以保持连续性。
- 每行代码最长 96 个字符，确保代码库与书中的排版一致。

因此，代码仅是一个基准实现，其中包含许多问题可以供读者改进。优化或现代化代码的方法有许多，但我们优先选择最简单的实现。

我们假设读者熟悉向量 (如点乘与向量加法)。若你仍对此不甚了解，去做点复习吧。如果你需要复习或学习相关知识，可以看看 Morgan McGuire 的 [Graphics Codex](https://graphicscodex.com/)， Steve Marchner 与 Peter Shirley 写的《Fundamentals of Computer Graphics》，或 J.D.Foley 与 Andy Van Dam 写的《Computer Graphics: Principles and Practice》。

请去 GitHub 仓库上看看[项目 README](https://raytracing.github.io/README.md) 文件，这可帮助你取得与此项目相关的资讯。你能在那看到项目的目录结构、了解如何构建与运行项目，以及如何贡献与修正错误。

你可以在我们的 [Further Reading wiki page](https://github.com/RayTracing/raytracing.github.io/wiki/Further-Readings) 找到更多项目相关资源。

此系列书籍经过了格式化，可以直接在你的浏览器里得到不错的输出。我们在 [release](https://github.com/RayTracing/raytracing.github.io/releases/) 中的 "Assets" 区域提供书籍的 PDF 版本。

若你想与我们交流，欢迎发送邮件到：
- Peter Sherley, ptrshrl@gmail.com
- Steve Hollasch, steve@hollasch.net
- Trevor David Black, trevordblack@trevord.black

最后，若你在实现过程中遇到问题、对内容有疑问，或想分享你的想法与成果，请移步到 GitHub 项目中的[论坛](https://github.com/RayTracing/raytracing.github.io/discussions/)。

感谢所有对此项目提供帮助的人们。你可以在本书最终的[致谢](https://raytracing.github.io/books/RayTracingInOneWeekend.html#acknowledgments)部分找到他们的名单。

让咱们开始吧！