# 从零发布一个 Vite+Vue3 脚手架：动态生成项目与多平台发布

## 背景：从「重复 CV 工程师」到「自动化工具开发者」

在团队开发中，每次新建项目都要经历「复制老项目 → 疯狂删文件 → 重写配置」的循环。

为了解决这个问题，我给自己写了一个支持 **TypeScript/JavaScript 双版本**、**插件按需加载**的脚手架工具，并成功发布到 npm 和 GitHub Packages。

本文将分享完整实现过程。

## 一、成果预览：一行命令搞定项目初始化

### 核心特性

1. **零配置启动**

   ```shell
   pnpm create vvt [项目名称]
   ```

2. **语言按需选择**

   ![cli 演示 1](./images/cli 演示 1.png)

3. **插件化架构**

   1. `postcss-pxtorem`：在设计图转页面的时候比较好用
   2. `tailwindcss`：减轻想类名、编写大量 css的苦恼，but 协同开发维护起来也有点难顶
   3. `vite-svg-loader`：如果项目中有较多的 svg 文件，推荐使用

   ![cli 演示 2](./images/cli 演示 2.png)

4. 按照使用说明启动项目，结果如下：

   ![启动后项目页面](./images/启动后项目页面.png)

## 二、环境准备

- `node.js（>=v20）`
- `npm`
- `pnpm`
- `Vite 脚手架`
- `npm账号/github账号`

## 三、初始化项目

### 1. 准备基础模板

我是使用的`vite`脚手架创建了一个基础框架，然后添加了一些基础配置，作为自己的脚手架基础。项目结构概览如下，具体配置可以参考我的[**搭建一个自己的开箱即用的 VUE 脚手架**](https://juejin.cn/post/7485677817868140559) 和 [为 VUE 脚手架添加开发常用的插件/配置 ](https://juejin.cn/post/7485725589610512399) 这两篇文章。

![项目概览](./images/项目概览.png)

调整为能发布成脚手架的结构，整体结构如下：

```shell
├── bin
│   └── cli.js         # 命令行入口
├── templates          # 项目模板
│   ├── typescript     # TS 版本模板，里面的代码就是上面准备的基础模板
│   └── javascript     # JS 版本模板，根据 TS 进行部分删减
├── gitignore          # node_modules、.npmrc文件一定要放进来！！
├── .npmrc             # 填写 github packages 注册表 token
├── LICENSE            # 许可文件
├── package.json       # 会进行一些发布配置
├── publish-github.js  # 自动化发布脚本
└── README.md          # 脚手架使用说明文件
```

### 2. 创建`bin/cli.js`文件

```js
#!/usr/bin/env node

console.log('hello vvt')
...主内容
```

`#!`开头标识这个文件被当做执行文件，可以当做脚本运行。后面的`/usr/bin/env node`表示文件用node执行，基于用户安装根目录下的环境变量中查找node

### 3. 修改` package.json`

```json
{
  "name": "create-vvt", // 脚手架的名称，若要发布到 npm，必须唯一。创建命令也是这个
  "version": "0.1.0", // 每次发布需要更新版本号
  "type": "module",
  "description": "一个基于 Vite + Vue3 + TypeScript/JavaScript 的项目模板脚手架",
  "bin": {
    "vvt": "bin/cli.js" // 用于配置指令，指令由key、value组成，key是指令名，value为运行指令时运行的文件
  }
  ...
}
```

### 4. 本地测试

使用`npm link`将这个项目链接到全局，进行测试。关于链接方法的具体操作可以在[vite+ts发布 npm 包过程记录](https://juejin.cn/post/7485677817868386319#heading-13)这里找到。测试成功之后就进行核心代码部分了。

## 四、关键实现细节

> - **用户输入校验**：项目名非空检测
> - **模板选择**：TS/JS 双版本支持
> - **插件按需加载**：Tailwind CSS 等动态配置

### 1. 核心依赖

```shell
pnpm add chalk commander inquirer ora
# 用于构建用户友好的交互界面
```

| 工具        | 作用           | 官方文档                                        |
| :---------- | :------------- | :---------------------------------------------- |
| `commander` | 命令行参数解析 | [文档链接](https://npmjs.com/package/commander) |
| `inquirer`  | 交互式问答     | [文档链接](https://npmjs.com/package/inquirer)  |
| `chalk`     | 终端输出美化   | [文档链接](https://npmjs.com/package/chalk)     |
| `ora`       | 加载动画       | [文档链接](https://npmjs.com/package/ora)       |

### 2. 动态生成项目模板

```mermaid
graph TD
  A[用户输入] --> B{语言选择}
  B -->|TypeScript| C[加载 TS 模板]
  B -->|JavaScript| D[加载 JS 模板]
  C/D --> E[插件按需装配]
  E --> F[写入 package.json]
  F --> G[项目生成完成]
```

1. 定义可选的插件和语言选项

   ```js
   // 定义可选插件
   const PLUGINS = [
     {
       name: 'postcss-pxtorem',// 用在命令行显示
       value: 'pxtorem',
       description: '将 px 单位转换为 rem 单位',// 用在命令行显示
       devDependencies: {
         'postcss-pxtorem': '^6.1.0'
       }
     }
     ...其他插件
   ];
   
   // 语言选项
   const LANGUAGES = {
     typescript: {
       name: 'TypeScript',
       value: 'typescript',
       description: '使用 TypeScript 语法（类型安全的 JavaScript 超集）',
       templateDir: 'templates/typescript' // 对应模板所在文件夹路径
     },
     javascript: {
       name: 'JavaScript',
       value: 'javascript',
       description: '使用 JavaScript 语法（更简洁，无类型检查）',
       templateDir: 'templates/javascript'
     }
   };
   ```

   

2. 基于`commander`执行自定义命令指令，准备创建项目

   ```js
   // 初始化命令行工具
   import { Command } from 'commander';
   const program = new Command();
   
   program
     .name('create-vite-vue3-ts')
     .description('基于 Vite + Vue3 的项目模板生成工具')
     .version('0.1.0')
     .argument('[project-name]', '项目名称')
     .action(async (projectName) => {
       try {
         await createProject(projectName); // 使用action绑定主逻辑函数
       } catch (error) {
         console.error(chalk.red('错误：') + error.message);
         process.exit(1);
       }
     });
   program.parse(process.argv);
   ```

   

3. 使用`inquirer`收集用户输入，如项目名称、描述、作者等

   ```js
   // 创建项目主逻辑函数createProject
   // 1. 获取项目名称
   const { name, description, author } = await inquirer.prompt([
     {
       type: 'input',
       name: 'name',
       message: '请输入项目名称：',
       default: projectName || 'vite-vue3-project',
       // 对输入添加校验逻辑，避免空值
       validate: (input) => {
         if (!input.trim()) {
           return '项目名称不能为空';
         }
         return true;
       }
     },
     {
       type: 'input',
       name: 'description',
       message: '请输入项目描述：',
       default: '基于 Vite + Vue3 的项目模板'
     },
     {
       type: 'input',
       name: 'author',
       message: '请输入作者名称：',
       default: 'egg'
     }
   ]);
   
   // 2. 选择语言
   const { language } = await inquirer.prompt([
     {
       type: 'list',
       name: 'language',
       message: '请选择开发语言：',
       choices: Object.values(LANGUAGES).map((lang) => ({
         name: `${lang.name} (${lang.description})`,
         value: lang.value
       })),
       default: 'typescript'
     }
   ]);
   ```

4. 检查用户计划创建的文件是否存在，存在的话需要覆盖

   ```js
   const targetDir = path.join(process.cwd(), name); // name是用户填写的项目名
   
   if (fs.existsSync(targetDir)) {
       const { overwrite } = await inquirer.prompt([
         {
           type: 'confirm',
           name: 'overwrite',
           message: `目标目录 ${chalk.cyan(name)} 已存在。是否要覆盖？`,
           default: true
         }
       ]);
   
       if (!overwrite) {
         throw new Error('操作取消');
       }
   
       const spinner = ora('正在清理目录...').start();
   
       await fs.promises.rm(targetDir, { recursive: true, force: true });
       spinner.succeed(chalk.green('目录清理完成'));
     }
   ```



5. 指导用户选择安装插件

   ```js
   // 逐个选择插件
   const selectedPlugins = [];
   for (const plugin of PLUGINS) {
     const { install } = await inquirer.prompt([
       {
         type: 'confirm',
         name: 'install',
         message: `是否安装 ${plugin.name}（${plugin.description}）？`,
         default: false
       }
     ]);
     if (install) {
       selectedPlugins.push(plugin.value);
     }
   }
   ```



6. 创建项目

   ```js
   // 1. 创建项目Loading初始化
   const spinner = ora(chalk.bgYellow('正在创建项目...')).start();
   // 2. 根据选择的语言选择对应模板进行复制，也可以把模板文件放到 github上，复制操作换成从仓库下载，这样模板更新不用更新脚手架，用户也能直接使用。
   const templateDir = path.resolve(
     __dirname,
     '..',
     LANGUAGES[language].templateDir
   );
   fs.mkdirSync(targetDir, { recursive: true });
   await copyTemplate(templateDir, targetDir);
   // 3. 更新配置文件
   spinner.text = '正在更新配置文件...';
   // 这个方法是针对选择的插件对模板文件内容进行修改
   await updateProjectFiles(targetDir, selectedPlugins, {
     name,
     description,
     author,
     language
   });
   // 这个方法是针对选择的插件修改下载依赖配置
   await updatePackageJson(targetDir, selectedPlugins, {
     name,
     description,
     author
   });
   // 4. loading结束
   spinner.succeed(chalk.green('项目创建成功！'));
   
   // 5. 输出使用说明
   console.log('\n使用说明：');
   console.log(chalk.cyan(`  cd ${name}`));
   console.log(chalk.cyan('  pnpm install'));
   console.log(chalk.cyan('  pnpm dev\n'));
   ```

   

7. `updateProjectFiles`和`updatePackageJson`部分代码示例

   ```js
   /** updateProjectFiles 根据用户选择处理tailwindcss插件
    ** 主要需要修改导入路径，和类型导入的处理
    ** 其他插件处理方式类似 
   */
   
   // 根据选择的语言决定入口文件的扩展名
   const mainExtension = projectInfo.language === 'typescript' ? '.ts' : '.js';
   const mainPath = path.join(root, `src/main${mainExtension}`);
   
   // 处理 CSS 配置文件
   const cssConfigExtension =
     projectInfo.language === 'typescript' ? '.ts' : '.js';
   const cssConfigPath = path.join(
     root,
     `viteConfig/css/index${cssConfigExtension}`
   );
   
   // 根据选择的插件修改配置
   if (fs.existsSync(cssConfigPath)) {
     let cssConfig = fs.readFileSync(cssConfigPath, 'utf-8');
   
     // 处理 tailwindcss 插件
     if (!selectedPlugins.includes('tailwind')) {
       // 如果没有安装，则删掉导入代码
       cssConfig = cssConfig.replace(/import tailwindcss.*;\n/, '');
       cssConfig = cssConfig.replace(/\s*tailwindcss\(\),?\n?/, '');
       mainContent = mainContent.replace(/import '\.\/tailwind.css';\n/, '');
   		// 删掉tailwind.css配置文件
       const tailwindPath = path.join(root, 'src/tailwind.css');
       if (fs.existsSync(tailwindPath)) {
         fs.unlinkSync(tailwindPath);
       }
   		// 删掉tailwind.config.js/ts配置文件
       const tailwindConfigExtension =
         projectInfo.language === 'typescript' ? '.ts' : '.js';
       const tailwindConfigPath = path.join(
         root,
         `tailwind.config${tailwindConfigExtension}`
       );
       if (fs.existsSync(tailwindConfigPath)) {
         fs.unlinkSync(tailwindConfigPath);
       }
     }
   	// 重写 vite css配置文件 和main文件
     fs.writeFileSync(cssConfigPath, cssConfig);
     fs.writeFileSync(mainPath, mainContent);
   }
   ```

   ```js
   /** updatePackageJson 根据用户选择处理安装依赖配置 */
   const pkgPath = path.join(root, 'package.json');
   const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf-8'));
   
   // 获取选中插件的依赖
   const devDependencies = {};
   for (const plugin of PLUGINS.filter((p) =>
     selectedPlugins.includes(p.value)
   )) {
     Object.assign(devDependencies, plugin.devDependencies);
   }
   
   // 更新 package.json
   pkg.devDependencies = {
     ...pkg.devDependencies,
     ...devDependencies
   };
   
   // 移除未选中插件的依赖
   PLUGINS.forEach((plugin) => {
     if (!selectedPlugins.includes(plugin.value)) {
       Object.keys(plugin.devDependencies).forEach((dep) => {
         delete pkg.dependencies[dep];
         delete pkg.devDependencies[dep];
       });
     }
   });
   
   fs.writeFileSync(pkgPath, JSON.stringify(pkg, null, 2));
   ```

## 五、双平台发布：npm 与 GitHub Packages

为了能更便捷的在双平台进行发布，在项目里面做了特殊配置。

### 1. 发布到`npm`

发布到`npm`配置可以参考我这篇文章：[vite+ts发布 npm 包过程记录](https://juejin.cn/post/7485677817868386319#heading-13)

### 2. 发布到`github pacakges`

下面是使用 npm 注册表发布到`github pacakges`的简单说明(👉[官方教程地址](https://docs.github.com/zh/packages/learn-github-packages/publishing-a-package))：

1. 创建自己的`personal access token (classic)`

   - 在 GitHub 任意页面的右上角，单击个人资料照片，然后单击 “设置/Settings”。

   - 在左侧边栏中，单击“ 开发人员设置/Developer settings”。

   - 请在左侧边栏的“ Personal access tokens”下，单击“Tokens(classic)” 。

   - 单击“生成新令牌/Generate new token”。

   - 选择“Generate new token(classic)”。

   - 在“备注”字段中，为令牌提供一个描述性名称。

   - 要为令牌提供到期时间，请选择“到期时间”，然后选择默认选项或单击“自定义”以输入日期 。

   - 选择要授予此令牌的作用域。发布 packages 包必须要要选择`write:packages`、`read:packages`、`delete:packages`

   - 单击“生成令牌”。

   - 要将新令牌复制到剪贴板，保存好。

   - 如果不再需要 personal access token，请删除。

     

2. 使用 `personal access token (classic)` 向 `GitHub Packages` 验证

   在脚手架项目根目录创建`.npmrc`文件，然后添加到`.gitignore`中。内容如下：

   ```shell
   //npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
   @star-devil:registry=https://npm.pkg.github.com
   ```

   `${GITHUB_TOKEN}`就是上个步骤生成的`personal access token`，通过在命令行使用`export`语句传入，更安全。`star-devil`是我 github的昵称

3. 创建`publish-github.js`文件：为了更方便的同时在 npm 和 github 上发布包

```js
import fs from 'fs';
import { execSync } from 'child_process';

// 检查环境变量
if (!process.env.GITHUB_TOKEN) {
  console.error('错误: GITHUB_TOKEN 环境变量未设置');
  console.error('请使用以下命令设置: export GITHUB_TOKEN=your_token');
  process.exit(1);
}

try {
  // 读取原始 package.json
  const pkg = JSON.parse(fs.readFileSync('./package.json', 'utf8'));

  // 备份原始 package.json
  fs.writeFileSync('./package.json.backup', JSON.stringify(pkg, null, 2));

  // 修改为 GitHub Packages 所需的格式
  pkg.name = '@star-devil/create-vvt';
  pkg.publishConfig = {
    registry: 'https://npm.pkg.github.com',
    access: 'public'
  };

  // 写入修改后的 package.json
  fs.writeFileSync('./package.json', JSON.stringify(pkg, null, 2));

  console.log('Modified package.json for GitHub Packages');

  // 执行发布
  execSync('npm publish', { stdio: 'inherit' });
  console.log('Published to GitHub Packages successfully!');
} catch (error) {
  console.error('Error:', error);
} finally {
  // 恢复原始 package.json
  if (fs.existsSync('./package.json.backup')) {
    fs.copyFileSync('./package.json.backup', './package.json');
    fs.unlinkSync('./package.json.backup');
    console.log('Restored original package.json');
  }
}

```

4. `package.json`中增加发布脚本命令

```json
{
  "scripts": {
    "npm:publish": "npm publish", // 发布到 npm 使用这个命令
    "github:publish": "node publish-github.js" // 发布到 github 使用这个命令
	},
  "repository": {
    "type": "git",
    "url": "git+https://github.com/<github-name>/<package-name>.git"
  },
  "homepage": "https://github.com/<github-name>/<package-name>",
  "license": "MIT",
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org"
  },
}
```

## 六、完整源码与使用指南

- **GitHub 仓库**: [vite-vue3-ts-cli](https://github.com/your-repo-link)
- **npm 安装**: [create-vvt](https://www.npmjs.com/package/create-vvt)

当然，还有很多功能可以扩充和修改，后面会继续深入研究。完整代码可以查看我的 github 仓库～谢谢～
