# 病理切片标注-Vue 3 组件库，代码已开源

## 背景

前端要能展示病理切片图像，在上面查看 AI 标注结果，还能让医生手动修改标注、对比不同版本的差异。

调研了一圈，图像加载这块没什么好纠结的——病理切片都是几十亿像素级别的超大图，必须用瓦片金字塔方案，行业标准就是 **OpenSeadragon + DZI（Deep Zoom Image）格式**，跟谷歌地图加载卫星图一个原理。

标注绘制层则选了一个轻量的开源插件 **@wtsml/doodle**，基于 SVG overlay，支持多边形、矩形、圆、椭圆这些常用形状，还有撤销/重做能力。

这两个引擎本身都挺好用，但从"能跑"到"产品可用"中间还差不少东西：

- 标注数据格式要转换（后端吐的 JSON 跟 doodle 要的格式不一样）
- 图像信息要解析（尺寸、缩放级别、瓦片大小等）
- 要有一整套 UI 面板（缩放控制、工具切换、标注统计、区域列表……）
- 精修模式要管理状态切换
- 版本对比要做差异算法

每个项目都重新写一遍这些东西太费劲了，索性抽成一个组件库。

## 成品概览


[![示例截图](https://github.com/star-devil/pathology-dzi-viewer/blob/main/example_image/image.png?raw=true)](https://github.com/star-devil/pathology-dzi-viewer)

支持的功能：

- 🖼️ DZI 深度缩放图像浏览（缩放、平移、全屏、导航器）
- ✏️ 8 种标注工具（移动、直线、矩形、多边形、圆、椭圆、路径、闭合路径）
- 📊 标注统计面板（区域数量、类型分布）
- 🔧 AI 标注精修（画笔 + 撤销/重做/删除/提交）
- 🔍 版本对比（多版本差异可视化，最多 5 个版本）
- 🏗️ 完整三栏布局组件（信息面板 + 查看器 + 工具面板）
- 🧩 Composable API（`useAnnotationViewer` 一把梭管理所有状态）

## 安装

```bash
# Vue 3 项目直接装这个就行
npm install @pathology/dzi-viewer-vue
pnpm add @pathology/dzi-viewer-vue
  
# 另外这些 peer 依赖需要自己项目里已经装好
npm install vue element-plus @element-plus/icons-vue openseadragon
pnpm add vue element-plus @element-plus/icons-vue openseadragon
```

如果你不用 Vue 或者想自己搭 UI，可以只用核心引擎：

```bash
npm install @pathology/dzi-viewer-core
pnpm add  @pathology/dzi-viewer-core
```

GitHub 地址：[https://github.com/star-devil/pathology-dzi-viewer](https://github.com/star-devil/pathology-dzi-viewer)

## 怎么设计的

整个库分成两个包，一层的职责分得很清楚：

```
@pathology/dzi-viewer-vue     ← 你直接用的这一层
  │   Vue 3 组件 + Composable + 布局
  │   依赖 Element Plus 做 UI
  │
  └── @pathology/dzi-viewer-core   ← 纯 TypeScript 引擎
        │  创建查看器 / 标注管理 / 几何计算
        │  版本对比 / 坐标转换 / 图像加载
        │  零 UI 框架依赖
        │
        └── OpenSeadragon + @wtsml/doodle
```

为什么要拆？因为 core 层跟 Vue 没有任何关系，哪天需要在 React 或者原生 JS 项目里用，直接装 core 包就行。vue 包本质上就是给 core 包套了一层 Vue 3 的皮，加了面板组件和 composable。
  
构建上用了 pnpm workspace monorepo，core 用 tsup 打 ESM + CJS 双格式，vue 用 Vite 6 的 library mode 打包，类型声明用 vite-plugin-dts 自动生成。

## 三种工作模式

病理阅片的实际流程不是"打开图片看看"这么简单。我把流程梳理成了三种模式：

**预览模式（view）**：打开切片，查看 AI 标注结果。这是最常用的场景。
**精修模式（refine）**：医生发现 AI 某个区域标得不对，进入精修模式，用画笔工具修改边界，满意后提交。
**对比模式（compare）**：把 AI 标注和人工精修后的标注做差异可视化——哪些区域新增了、删除了、修改了、没变。最多支持 5 个版本同时对比，每个版本自动分配一种颜色。

模式之间可以自由切换：
```
view → refine → view
  ↓              ↑
compare ─────────┘
```

状态管理通过 `useAnnotationViewer` 这个 composable 全部封装好了，业务代码只需要传一个 `mode` 参数。

## 一个简单例子

最简情况下，几行代码就能跑起来：
```vue

<template>
  <AnnotationViewer
    ref="viewerRef"
    :image-url="dziUrl"
    :annotation-data="annotations"
    :current-tool="currentTool"
    @image-loaded="onImageLoaded"
  />
</template>

<script setup>
import { ref } from "vue";
import { AnnotationViewer } from "@pathology/dzi-viewer-vue";
import "@pathology/dzi-viewer-vue/dist/dzi-viewer-vue.css";

const dziUrl = ref("https://your-server.com/slide.dzi");
const currentTool = ref(null);
const annotations = ref([]);
  
function onImageLoaded(info) {
  console.log(`图像尺寸: ${info.width} × ${info.height}`);
}
</script>
```

如果需要一个完整的标注预览页面，直接用 `AnnotationDetailLayout`，传一堆 props 就能拼出三栏布局。

如果想更精细地控制业务逻辑，用 `useAnnotationViewer` composable 自己组合面板组件就行——它会管理所有内部状态，你只需要实现一个简单的数据接口告诉它"标注从哪个 API 取、精修提交到哪个 API"。

## 标注数据格式 

这里踩过一个坑。一开始想直接把 doodle 的内部格式（`pos` 数组）暴露出去，后来发现不同形状的 `pos` 结构完全不一样——直线的 `pos` 是 `[x1, y1, x2, y2]`，圆的 `pos` 是 `[x, y, r]`，多边形的 `pos` 是 `[x, y, x, y, ...]` 循环。让后端理解这些差异太累了。

最终定了一套更友好的 JSON 格式：
```ts
interface AnnotationRegion {
  external: AnnotationPoint[];   // 外边界
  holes: AnnotationPoint[][];    // 内部空洞
  shape?: "line" | "rect" | "polygon" | "circle" | "ellipse" | "path" | "closed_path";
}
```

把形状信息显式写到 `shape` 字段里，后端和前端用同一套协议，组件内部负责转成 doodle 需要的格式。转换逻辑都在 core 包的 `convertPointsToPos` 函数里，大概 10 行代码。

## 版本对比的差异算法

版本对比不只是把两个版本叠在一起看，还需要告诉用户"哪些区域变了"。

差异算法的大致思路：先做精确匹配（点坐标全部相同 → unchanged），再对剩余未匹配的区域基于**面积比 + 重心距离**计算相似度。相似度达到阈值（默认 0.7）的判断为 modified，完全找不到匹配的分别标记为 added 或 removed。

不算复杂，但能覆盖绝大多数实际场景。算法实现在 core 包的 `utils/annotationDifference.ts` 里。

## 数据对接，不绑定后端

组件不假设你用什么后端框架。暴露了一个 `AnnotationDataProvider` 接口：

```ts
interface AnnotationDataProvider {
  getAnnotation(imageId, versionId?): Promise<AnnotationData>;
  getVersions(params?): Promise<VersionListItem[]>;
  submitRefine(imageId, params): Promise<void>;
}
```

业务方实现这三个方法就行。不传 `getVersions` 就不展示版本对比，不传 `submitRefine` 就隐藏精修提交按钮，按需接入。

---
  
- **GitHub**：[star-devil/pathology-dzi-viewer](https://github.com/star-devil/pathology-dzi-viewer)

- **npm**：[@pathology/dzi-viewer-core](https://www.npmjs.com/package/@pathology/dzi-viewer-core) / [@pathology/dzi-viewer-vue](https://www.npmjs.com/package/@pathology/dzi-viewer-vue)

- **License**：MIT