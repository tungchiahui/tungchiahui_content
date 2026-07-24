---
title: "Nuxt Studio教程"
---

## 简介


## 安装
### 安装
用VScode打开自己的项目，并用`Ctrl+~`调出终端界界面。

![alt text](../../../public/images/2025-07-01-Nuxt.js网站框架/1784896044093.png)

在项目根目录执行：

```bash
npx nuxt module add nuxt-studio
```

这是当前官方推荐的安装方式。它会安装 nuxt-studio，并自动把模块加入 Nuxt 配置。

执行过程中如果出现：

```text
Need to install the following packages...
Ok to proceed? (y)
```

输入`y`然后回车。

安装完成后，检查 `nuxt.config.ts`。正常应该类似：

![alt text](../../../public/images/2025-07-01-Nuxt.js网站框架/1784896238714.png)

检查 `package.json`。正常应该类似：

![alt text](../../../public/images/2025-07-01-Nuxt.js网站框架/1784896265858.png)

不需要手动填写版本号，以 npm 实际安装的版本为准。

### 测试

执行：

```bash
npm run dev
```

终端一般会显示：

Local: http://localhost:3000/

![alt text](../../../public/images/2025-07-01-Nuxt.js网站框架/1784896538963.png)


浏览器打开：

http://localhost:3000

安装成功后，页面左下角应该出现一个 Studio 悬浮按钮。点击它即可进入编辑模式。

![alt text](../../../public/images/2025-07-01-Nuxt.js网站框架/1784897542820.png)


