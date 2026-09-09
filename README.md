# 个人博客（Hexo + Butterfly）

线上地址：<https://stellacode98.github.io/>

## 分支说明

- `source`：博客源码（笔记、配置、主题），日常在这里写作
- `main`：`hexo deploy` 自动推送的构建产物，勿手动修改

## 新电脑初始化

前置：安装 Git、Node.js（20+ LTS，本仓库开发环境为 v22）。

```bash
git clone -b source --recurse-submodules https://github.com/StellaCode98/StellaCode98.github.io.git
cd StellaCode98.github.io
npm install
```

如果已经普通 clone 过（主题目录是空的），补一句即可：

```bash
git submodule update --init
```

> 主题 Butterfly 以 submodule 形式固定在 commit `f223b18`，与线上部署版本一致。

## 本地预览

```bash
npx hexo clean && npx hexo server
# 浏览器打开 http://localhost:4000
```

## 日常写作与发布

```bash
npx hexo new post "文章标题"              # 在 source/_posts/ 下写 Markdown
npx hexo clean && npx hexo g -d           # 生成并部署到 main，线上生效
git add . && git commit -m "新笔记" && git push origin source  # 源码同步
```

换电脑开工前先 `git pull origin source` 拉最新笔记。
