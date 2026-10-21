# 小康的博客

个人技术博客，基于 [Hexo](https://hexo.io/) + [Cactus](https://github.com/probberechts/hexo-theme-cactus) 主题，通过 GitHub Pages 部署。

🌐 在线访问：<https://xiaokang1993.github.io/blog>

## 技术栈

- **静态生成器**：[Hexo](https://hexo.io/) v8
- **主题**：[hexo-theme-cactus](https://github.com/probberechts/hexo-theme-cactus)（dark 配色）
- **包管理**：[pnpm](https://pnpm.io/)
- **部署**：GitHub Actions → GitHub Pages

## 本地开发

```bash
# 安装依赖
pnpm install

# 本地预览（默认 http://localhost:4000/blog）
pnpm dev

# 生成静态文件到 public/
pnpm build

# 清理生成产物
pnpm clean
```

## 目录结构

```
blog/
├── source/          # 博客源文件（Markdown 文章）
│   ├── _posts/       # 文章
│   └── _data/        # 数据
├── themes/cactus/    # 主题（git submodule）
├── _config.yml       # Hexo 站点配置
├── package.json      # 依赖与脚本
└── .github/workflows/  # CI/CD 部署工作流
```

## 写作

新建文章：

```bash
pnpm hexo new "文章标题"
```

文章生成在 `source/_posts/` 下，编辑后提交即可，GitHub Actions 自动构建发布。

## 部署

推送到 `main` 分支后，GitHub Actions 自动构建并发布到 GitHub Pages：

- 仓库：<https://github.com/xiaokang1993/blog>
- 站点：<https://xiaokang1993.github.io/blog>

## License

MIT
