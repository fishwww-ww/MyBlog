# MyBlog

Hexo 博客项目，基于 `hexo-theme-butterfly` 自定义配置（参见 `_config.butterfly.yml`）。按照下述步骤即可完成安装与本地预览。

## 环境要求

- Node.js ≥ 18（推荐配合 `npm` 10+ 或 `pnpm` 8+）
- Git（用于部署到 GitHub Pages）

## 安装依赖

```bash
npm install
```

`package.json` 已声明 `hexo-theme-butterfly`，`npm install` 会把主题放到 `node_modules/hexo-theme-butterfly` 中。若 `themes/butterfly` 不存在，可执行：

```bash
rm -rf themes/butterfly
cp -R node_modules/hexo-theme-butterfly themes/butterfly
```

## 本地开发

```bash
npx hexo clean       # 可选：清理缓存与生成文件
npx hexo s           # 启动本地服务，默认 http://localhost:4000
```

修改 `source/` 下的文章或 `_config*.yml` 后，Hexo 会自动重新渲染。若需重新生成静态文件，可运行：

```bash
npx hexo g
```

## 部署

已在 `_config.yml` 中配置 Git 部署，可直接：

```bash
npx hexo clean
npx hexo g -d
```

命令会将生成的 `public/` 推送到 `main` 分支指定仓库 `https://github.com/fishwww-ww/fishwww-ww.github.io`。

## 常用路径说明

- `_config.yml`：Hexo 全局配置
- `_config.butterfly.yml`：Butterfly 主题配置
- `source/_posts/`：博客文章 Markdown
- `themes/butterfly/`：主题源码（如需升级主题，可重新执行上面的复制操作）

如启动遇到 `plugins.yml` 缺失，通常是主题未正确安装，重新执行“安装依赖”与“复制主题”步骤即可。
