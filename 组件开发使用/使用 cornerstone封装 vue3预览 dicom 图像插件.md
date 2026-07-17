## 基础

### 官方地址

cornerstone官方网址（目前是 3.x版本）：https://www.cornerstonejs.org/docs/getting-started/overview

### 介绍

`Cornerstone3D` 是一个轻量级 Javascript 库，用于在支持 HTML5 画布元素的现代 Web 浏览器中可视化医学图像。使用 `@cornerstonejs/core` 及其随附的库（如 `@cornerstonejs/tools`），您可以完成各种成像任务。

### 背景

没有找到关于使用 vue3 封装的cornerstone3.x版本的组件，3.x版本进行了很多性能优化，所以我打算从基础开始慢慢的完成这个组件的封装。同时也熟悉一下官方各种 API 的使用。本文主要记录我的开发过程，非概念讲解。

不过，掘金上的一篇文章（👉[地址在这里](https://juejin.cn/post/7326432875955798027)），详细讲述了 cornerstone(2.x) 的核心概念，给我提供了很大帮助，感兴趣的小伙伴可以移步。

### 安装

```shell
pnpm add @cornerstonejs/core @cornerstonejs/tools @cornerstonejs/dicom-image-loader @cornerstonejs/nifti-volume-loader
```

- **@cornerstonejs/core**: 核心库，提供医学影像渲染的基础功能
- **@cornerstonejs/tools**: 工具集，包含测量、标注等交互工具
- **@cornerstonejs/dicom-image-loader**: 加载和解析DICOM格式的医学影像
- **@cornerstonejs/nifti-volume-loader**: 加载和解析NIfTI格式的3D医学体数据。如果**不需要处理 NIfTI 格式的 3D 体数据**（如 `.nii`/`.nii.gz` 文件）→ 可不安装

