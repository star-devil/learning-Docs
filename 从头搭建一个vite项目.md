# 

# 一、初始搭建

> Vite 需要 [Node.js](https://nodejs.org/en/) 版本 18+ 或 20+。然而，有些模板需要依赖更高的 Node 版本才能正常运行，当你的包管理器发出警告时，请注意升级你的 Node 版本。

## 1. 使用脚手架搭建

```shell
pnpm create vite

## 然后按照提示操作即可！
```

**or**

```shell
pnpm create vite my-vue-app --template vue-ts

## 通过附加的命令行选项直接指定项目名称和你想要使用的模板。
## 这里我构建的是 Vite + Vue-ts项目。
```

    其他模板：`vanilla`，`vanilla-ts`, `vue`, `vue-ts`，`react`，`react-ts`，`react-swc`，`react-swc-ts`，`preact`，`preact-ts`，`lit`，`lit-ts`，`svelte`，`svelte-ts`，`solid`，`solid-ts`，`qwik`，`qwik-ts`

## 2. 手动安装

    `pnpm add -D vite` 并创建一个 `index.html` 文件。相关操作不做更多介绍

*执行成功之后即可得到一个初始框架项目，但是想要高效开发还远远不够....*

# 二、搭配各种扩展插件

> 使用各种插件，提高代码编写效率

## 1. 使用文件路径别名

1. 使用自定义符号代替指定路径，简化模块导入路径
   
   `pnpm add @types/node --save-dev`

        

2. 进入`vite.config.ts`
   
   ```js
   import path from 'path'
   
   export default defineConfig({
   ......
   resolve: {
       alias: {
           '@': path.resolve('./src'), // @代替src
           '@c': path.resolve('./src/components') // @c代替./src/components
           // 可根据后续开发情况自行添加
           },
       },
   })
   ```

## 2. 添加代码格式化扩展

>    代码格式化工具可以使编码更专注，在协同开发时统一代码风格十分有用

1. 安装 [prettier]([什么是 Prettier？ · Prettier 中文网](https://prettier.nodejs.cn/docs/en/index.html)) 扩展(v3.x)： 代码美化
   
   ```shell
   pnpm add --save-dev --save-exact prettier
   ```

2. 创建配置文件
   
   ```shell
   node --eval "fs.writeFileSync('.prettierrc','{}\n')"
   
   ## 这将会在项目中添加一个空的配置文件，告诉编辑器和其它工具已经启用`Prettier`
   ```
- 命令执行成功之后可在项目根目录发现文件`.prettierrc`，可以为配置文件添加以下简单内容（缩进默认为2个空格，就没有再进行设置，更多参数默认值请看[文档]([Options · Prettier](https://prettier.io/docs/en/options))）：
  
  ```json
  {
      "jsxSingleQuote": false, // 在JSX中使用双引号
      "singleQuote": true, // 在普通代码中使用单引号
      "endOfLine": "lf", // 设置行尾字符为换行符（Line Feed，LF）
      "semi": true, // 在语句末尾添加分号
      "trailingComma": "none", // 在对象、数组、函数的最后一个参数后不添加逗号。如果设置为es5表示末尾要添加逗号，且遵循ES5标准。
      "insertPragma": true, // 在文件头增加是否已经格式化的注释
      "requirePragma": false // 默认值为false,初始可以不设置。设置为true时需要与inserPragma搭配使用,当逐渐将大型、未格式化的代码库过渡到 Prettier 时，设置为true会非常有用
  }
  ```

- `pnpm exec prettier . --write` 格式化所有内容
  
  - 如果某些文件格式化没有生效，比如缩进还是4个空格，请检查编辑设置是否冲突；比如VScode，可以在编辑的`setting.json`中检查比如json文件是否使用了其他格式器：
  
  ```json
  "[json]": {
      "editor.defaultFormatter": "xxxx"
  }
  ```
  
  - 或者你的配置文件如果使用了`useTab:true`，就算设置了`tabWidth:2`，某些文件也会使用4个空格的制表符

- `npx prettier . --check` 检查文件是否已经格式化
3. 通过编辑器运行`Prettier`

> 以`vscode`为例说明。配置成功后可以通过键盘快捷键或者保存文件时自动运行。另外，编辑器插件将选择你本地版本的 Prettier，确保你在每个项目中使用正确的版本。[编辑器配置文档]([编辑器集成 · Prettier 中文网](https://prettier.nodejs.cn/docs/en/editors.html#google_vignette))

- 在`vscode`中安装插件 `prettier-vscode`

- 安装好后再介绍界面点击**齿轮**图标会出现一个菜单，选择**设置**可进入配置编辑页面，每一个配置都有对应解释，可以根据自己需要进行设置，或者打开编辑器`setting.json`文件编辑。
  
  - 我设置后的setting.json文件如下：
    
    ```json
    // prettier扩展配置
      "editor.defaultFormatter": "esbenp.prettier-vscode",
      "prettier.bracketSameLine": true,
      "prettier.vueIndentScriptAndStyle": true,
      "prettier.jsxSingleQuote": false,
      "prettier.singleQuote": true,
      "prettier.trailingComma": "none",
      "prettier.insertPragma": true,
      "prettier.requirePragma": false,
      "[vue]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode" // 无论默认格式化器是什么，都使用 prettier
      },
    ```
  
  - 当`setting.json`和`.prettierrc`存在冲突时，通常情况下会以`.prettierrc`优先，但是如 `editor.tabSize` 或 `editor.insertSpaces`可能会在特定场景中影响格式化，还有上文提到的为特定文件类型定义格式化设置（如 `"[json]"`）。如果这样设置，编辑器可能会在格式化特定文件类型时优先使用这些设置，而不遵循 `.prettierrc` 配置。
  
  - 如果项目中不存在`.prettierrc`文件时，`setting.json`将作为后备格式器美化代码格式。

- 配置好后如果你想开启**保存**文件时**自动格式化**，需要在`setting.json`中添加：
  
  ```json
  // 保存时格式化当前文件（具体取决于 Prettier 本身支持的语言）
  "editor.formatOnSave": true
  // 如果你只想在保存时格式化指定的文件，如JavaScript：
  "editor.formatOnSave": false
  "[javascript]": {
      "editor.formatOnSave": true
  }
  ```

- 如果想开启使用键盘快捷键格式化当前文件，或者当前选中代码，需要下载一个插件（VScode）：`Formatting Toggle`，下载好之后启用插件。
  
  <img src="file:///Users/wangqiaoling/Library/Application%20Support/marktext/images/2024-11-06-10-49-19-image.png" title="" alt="" data-align="center">
  
  - 官方文档格式化快捷键是 (`CMD + SHIFT + P`/`OPT + SHIFT + P`)，如果没有生效，你就需要检查一下你自己的编辑器格式化代码的快捷键是怎么定义的，检查方法：通过快捷键`Ctrl + K, Ctrl + S`或`Cmd + K, Cmd + S`打开键盘快捷键设置，搜索格式或者format，可以看到我的格式化快捷键是：`SHIFT + CMD + F`
    
    ![](/Users/wangqiaoling/Library/Application%20Support/marktext/images/2024-11-06-10-48-36-image.png)

- 如果你的项目中有`.editorconfig` ，Prettier 会对其进行解析，并将其属性转换为对应的 Prettier 配置。此配置会被 `.prettierrc` 等覆盖。

- （共享 Prettier 配置很简单：只需发布???一个导出配置对象的模块，比如 `@company/prettier-config`，并在你的 `package.json` 中引用它：）

## 3. 添加代码校验扩展

> 使用 Prettier 来解决代码格式问题，使用 linter 来解决代码质量问题。但是Linters 通常不仅包含代码质量规则，还包含风格规则。使用 Prettier 时，大多数风格规则都是不必要的，但更糟糕的是——它们可能与 Prettier 发生冲突！下面使用的是官方提供的解决方案：



1. 安装`eslint` v9.x
   
   9.0版本弃用和删除了大量的API和规则，包括配置文件更新，Flat 取代 eslintrc。
   
   关于查询具体规则，除了可以查看文档，我强烈推荐使用[@eslint/config-inspector](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Feslint%2Fconfig-inspector "https://github.com/eslint/config-inspector")，它目前已经被eslint内置，我们只需在项目中运行`npx eslint --inspect-config`，就会启动一个网页 ，可以很方便的查看已配置的插件和规
   
   如需要从旧版本升级，请前往官网查看迁移方法。另外，ESlint9.0支持的Node.js版本是LTSv18.18.0+和v20，不支持v19以及之前的所有版本。
   
   
   
   ```shell
   pnpm create @eslint/config@latest`
   
   ## 根据提示选择适合你项目的选项，最后我的项目下载了这些依赖：
   ## + @eslint/js 9.14.0
   ## + eslint 9.14.0
   ## + eslint-plugin-vue 9.30.0
   ## + globals 15.12.0
   ## + typescript-eslint 8.13.0
   
   ## 并且项目根目录下增加了eslint.config.js文件
   ```



2. 安装插件：以下是与eslint配合使用的扩展插件，且支持扁平模式
   
   ```shell
   pnpm add vite-plugin-eslint2 ## 在 Vite 项目中集 ESLint,在每次保存文件时自动运行 ESLint 检查
   
   ## eslint的其他规范化插件
   pnpm add -D eslint-plugin-markdown
   pnpm add -D eslint-plugin-prettier
   pnpm add -D eslint-config-prettier
   
   
   ```



3. 在`eslint.config.js`中添加一些基础的校验规则，新版本配置文件同时还弃用了`.eslintignore`文件
   
   ```js
   import globals from 'globals';
   import pluginJs from '@eslint/js';
   import tseslint from 'typescript-eslint';
   import pluginVue from 'eslint-plugin-vue';
   import prettierPlugin from 'eslint-plugin-prettier';
   import eslintConfigPrettier from 'eslint-config-prettier';
   import markdown from "eslint-plugin-markdown";
   
   export default [
     { files: ['**/*.{js,mjs,cjs,ts,vue}']},
     {
       // 忽略检查的文件写在这里
       ignores: [
         '**/node_modules/**',
         '**/dist/**',
         '**/build/**',
         '.git',
         '.vscode',
         '.husky',
         'Dockerfile',
         'package-lock.json',
         'pnpm-lock.yaml'
       ]
     },
     // `languageOptions` 配置了浏览器环境下的全局变量。
     {
       languageOptions: {
           ecmaVersion: 'latest', // 使用最新的 ECMAScript 语法
           sourceType: 'module', // 代码是 ECMAScript 模块
           globals: globals.browser 
       }
     },
     // 这些recommended文件中已设置了很多规则，可以根据官网说明在rules中自行添加或修改一些规则
     // https://eslint.org/docs/latest/rules
     // https://typescript-eslint.io/rules
     // https://eslint.vuejs.org/rules/
     pluginJs.configs.recommended,
     ...tseslint.configs.recommended,
     ...pluginVue.configs['flat/essential'],
     ...markdown.configs.recommended,
     {
       files: ['**/*.vue'],
       languageOptions: {
         parser: pluginVue.parser, // 用于解析 <template> 中的 Vue 模板
         parserOptions: {
           ecmaVersion: 'latest',
           sourceType: 'module',
           parser: tseslint.parser, // 用于解析 <script> 中的 TypeScript
           ecmaFeatures: {
             jsx: true,
             tsx: true
           }
         }
       }
     },
     // 自定义的rules规则
     {
       rules: {
           'no-var': 'error', // 不允许使用var定义
           'prefer-const': 'error', // 强制在可能的情况下使用 const 声明变量，而不是 let
           'no-empty': ['error', { allowEmptyCatch: true }] // 允许catch空着
          }
      }
   ]
   ```



4. 在`vite.config.ts`中配置eslint插件
   
   ```ts
   // 在启动项目和打包代码时进行代码检查, 如果检查有error类型的问题就启动或打包失败, warn类型的问题不影响启动和打包
   import eslint from 'vite-plugin-eslint2';
   // https://vite.dev/config/
   export default defineConfig({
     plugins: [eslint({ lintOnStart: true, cache: false, fix: true })]
   });
   ```



5. 此时可以使用命令检查和修复文件的代码语法：
   
   ```shell
   npx eslint yourFilesPath(eg:./src/App.vue) ## 检查指定文件
   npx eslint yourFilesPath(eg:./src/App.vue) --fix ## 修复
   
   ## or 
   pnpm eslint . ## 检查所有文件
   pnpm eslint . --fix ## 修复
   ```

        

6. 通过编辑器运行`eslint`
   
   执行上述命令后，如果代码不符合规则约定的话，会发现终端会输出错误。但是编辑器并没有出现预期的红色波浪线提示，或者保存时自动修复错误。所以需要和`prettier`一样在编辑器中运行`eslint`
   
   -  安装插件`eslint`扩展插件
     
     ![](/Users/wangqiaoling/Library/Application%20Support/marktext/images/2024-11-14-14-30-26-image.png)
     
     
   
   - 勾选开启使用插件
     
     ![](/Users/wangqiaoling/Library/Application%20Support/marktext/images/2024-11-14-14-31-44-image.png)
     
     
   
   - 勾选开启扁平模式（打开设置搜索use flat config）
     
     ![](/Users/wangqiaoling/Library/Application%20Support/marktext/images/2024-11-14-14-34-14-image.png)
     
     
   
   - 下面是我编辑器setting.json关于eslint的配置
     
     ```json
     // eslint配置
       "eslint.useFlatConfig": true,
       "eslint.enable": true,
       "eslint.run": "onType",
       "eslint.validate": [
         "javascript",
         "javascriptreact",
         "typescript",
         "typescriptreact",
         "vue",
         "html",
         "json",
         "markdown"
       ],
       "eslint.codeAction.showDocumentation": {
         "enable": false
       },
     ```



## 4. 添加style样式代码格式化扩展（css、scss、less）
