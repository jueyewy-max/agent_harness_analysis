# 图表渲染记录

六张图使用 Mermaid 12.2.0 的 parse/render API 和 Chromium 成功渲染，中文字体使用 Noto CJK。

SVG 使用文本标签（htmlLabels: false），统一自上而下布局、linear 连线及蓝绿橙紫配色。每张图旁边保留 .mmd 源码；修改后需重新导出 SVG。

可用 Mermaid CLI 在仓库根目录对 docs/diagrams/*.mmd 逐一生成同名 SVG；本次因浏览器限制 file 页面导航，使用 Puppeteer 的内存页面加载 Mermaid 本地脚本并调用渲染 API。

主题：base；fontFamily：Noto Sans CJK SC, Noto Serif CJK SC, sans-serif；fontSize：16px；nodeSpacing：30；rankSpacing：35。
