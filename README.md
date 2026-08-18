# Ray Tracing in One Weekend 中文翻译

本项目是 Peter Shirley 等人所著《Ray Tracing in One Weekend》的中文翻译，希望用易懂的中文介绍光线追踪的基础概念与实现方式。

![《Ray Tracing in One Weekend》封面](https://raw.githubusercontent.com/RayTracing/raytracing.github.io/release/images/cover/CoverRTW1-small.jpg)

📖 在线阅读：
[https://ilsao.github.io/RTW-Translation/](https://ilsao.github.io/RTW-Translation/)

🌐 原作：
[https://raytracing.github.io/books/RayTracingInOneWeekend.html](https://raytracing.github.io/books/RayTracingInOneWeekend.html)

## 翻译进度

仓库目前包含以下译文：

- [1. 概览](docs/Simplified/overview.md)
- [2. 输出图像](docs/Simplified/Output-an-Image.md)
- [3. `vec3` 类](docs/Simplified/vec3-class.md)
- [4. 光线、简单相机与背景](docs/Simplified/Rays-a-Simple-Camera-and-Background.md)
- [5. 加入一个球体](docs/Simplified/Adding-a-Sphere.md)
- [6. 表面法向量与多物体](docs/Simplified/Surface-Normal-and-Multiple-Objects.md)
- [7. 将相机代码移至它自己的类](docs/Simplified/Moving-Camera-Code-Into-Its-Own-Class.md)
- [8. 抗锯齿](docs/Simplified/Antialiasing.md)
- [9. 漫反射材质](docs/Simplified/Diffuse-Materials.md)
- [10. 金属](docs/Simplified/Metal.md)
- [11. 介电质](docs/Simplified/Dielectrics.md)
- [12. 可定位相机](docs/Simplified/Positionable-Camera.md)
- [13. 散焦模糊](docs/Simplified/Defocus-Blur.md)
- [14. 接下来做什么？](docs/Simplified/Where-Next.md)
- [15. 致谢](docs/Simplified/Acknowledgments.md)
- [16. 引用本书](docs/Simplified/Citing-This-Book.md)

## 本地开发

请先安装 Node.js 20 或更高版本，然后克隆仓库并安装依赖：

```bash
git clone https://github.com/ilsao/RTW-Translation.git
cd RTW-Translation
npm ci
```

启动本地开发服务器：

```bash
npm start
```

开发服务器支持热更新，大多数内容变更无需重新启动即可预览。

## 构建与预览

生成生产环境的静态文件：

```bash
npm run build
```

构建结果会输出至 `build/`。在本地预览生产构建：

```bash
npm run serve
```

## 参与贡献

欢迎协助校对、修正译文或翻译新章节。开始之前，请阅读[贡献指南](docs/Contribution.md)，其中包含翻译用语、分支命名、提交信息和 Pull Request 格式等协作规范。

如果发现翻译不准确或内容有误，也可以直接[提交 Issue](https://github.com/ilsao/RTW-Translation/issues)。

## 技术栈

本网站使用 [Docusaurus 3](https://docusaurus.io/) 构建。
