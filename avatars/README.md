# 头像资源

WorkBuddy 专家包规范要求：

| 项 | 要求 |
|---|---|
| 文件名 | `avatars/expert.png` |
| 格式 | PNG（推荐）或 JPG |
| 尺寸 | 512 × 512 px |
| 大小 | ≤ 500 KB |

本仓库通过纯文本通道分发，因此**不内嵌二进制 PNG**，而是提供 `expert.svg` 作为矢量预览（可直接用浏览器打开）。

生成 / 获取合规 PNG 的三种方式：

1. **从发行包解压**：把 `stata-analysis-assistant.zip` 里的 `avatars/expert.png` 放到本目录。
2. **从 SVG 导出**：用浏览器、Inkscape、ImageMagick 或任意 SVG 转 PNG 工具把 `expert.svg` 导出为 512×512 PNG。
   ```bash
   magick -background none expert.svg -resize 512x512 expert.png
   ```
3. **自行生成**：任意 512×512 的 PNG 头像均可，`plugin.json` 里的 `avatar` 字段指向它即可。

`expert.png` 已写入 `.gitignore`，避免把二进制文件误提交；如希望它随仓库分发，删除该行后手动 `git add -f avatars/expert.png` 即可。

原始头像风格：青绿（cyan-teal）背景、Stata 分析师人设、笔记本上的 do 文件与回归输出表。
