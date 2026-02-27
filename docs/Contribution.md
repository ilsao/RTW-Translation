---
sidebar_position: 4
---

此处介绍项目规范与贡献要求。

# 如何开始

本项目使用框架 Docusaurus，具体使用请参考[文档](https://docusaurus.io/docs)。

这是本项目 [GitHub 仓库](https://github.com/ilsao/RTW-Translation#)的链接。

最后，附上 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html) 原文链接。

1. 取得仓库：`git clone https://github.com/ilsao/RTW-Translation.git`。

2. 使用 vscode 打开文件夹。

3. 在终端中输入 `npm start`。

4. 新增或修改 `RTW-Translation/docs/` 下的文件。注意，你不应该修改其他文件夹下的文件，除非你非常清楚自己在做甚么。

5. 准备部属时，先将所有改动 push 到你的 branch。然后在终端中输入 `cmd /C 'set "GIT_USER=<GITHUB_USERNAME>" && npm run deploy'`。

# 协作流程

1. 对于每个章节，开一个新分支。命名规则：`[TL] [章节标题]`。

2. 在更新分支前，确保此分支与 main 分支同步。
    - `git fetch origin` 
    - `git rebase main` 
    - `git push -f` 

3. commit 时，遵守以下规则：
    - 翻译文章：`TL: [章节标题] <备注>`。
    - 修改框架或外观：`<备注>`

4. 提 pr 时，应遵守：

```text
Title: [TL] [章节标题] <备注>
---
# 进度

# 补充信息
```

注意，仅在 pr 内使用 **简体中文**，其余部分使用**英文**。

# 语言

本项目使用**简体中文**作为第一部分的翻译语言，应该避免使用繁体中文专用语。

我制作了下表，帮助你校对两岸常见用语的差异：

| 英文 | 繁体中文 | 简体中文 |
| ---- | -------- | ------- |
| program | 程式 | 程序 |
| library | 函式库 | 库 |
| code | 程式码 | 代码 |
| debug | 除错 | 调试 |
| class | 类别 | 类 |
| declare | 宣告 | 声明 |
| variable | 变数 | 变量 |
| function | 函式 | 函数 |
| argument | 引数 | 参数 |
| pointer | 指标 | 指针 |
| array | 阵列 | 数组 |
| memory | 记忆体 | 内存 |
| stack | 堆叠 | 栈 |
| string | 字串 | 字符串 |
| polymorphism | 多型 | 多态 |
| template | 样板 | 模板 |
| process | 行程 | 进程 |
| thread | 执行绪 | 线程 |
| return value | 回传值 | 返回值 |
| operator | 运算子 | 运算符 |
| constructor | 建构子 | 构造函数 |
| deconstructor | 解构子 | 构析函数 |
| real-time rendering | 即时显影技术 | 实时渲染 |
| computer graphics | 电脑图学 | 计算机图形学 |