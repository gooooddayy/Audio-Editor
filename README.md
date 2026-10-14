# Audio-Editor
Audio Editor简单分割删除淡入淡出<br>
【WorkBuddy】index https://workbuddy.link/p/ombwakmCHohzOVTXhFkfRR?ext2=copy_link<br>
<img width="1218" height="566" alt="image" src="https://github.com/user-attachments/assets/a8c48cff-688b-4695-9630-40face0d7436" /><br>

## 功能
- 音频波形显示
- 自由位置、任意次数分割
- 选中片段淡入 / 淡出（0–5 秒可调）
- 删除片段，其余片段自动吸附闭合
- 从光标位置播放，0.5x / 0.75x / 1x / 2x / 3x 倍速试听
- 导出：自动去除首尾空白，16-bit PCM WAV

## 使用
浏览器直接打开 `index.html`，上传音频即可，无需安装任何依赖。

在线体验（GitHub Pages）：https://gooooddayy.github.io/Audio-Editor/

## 开发者文档
- [架构与代码走读](docs/ARCHITECTURE.md) — 单文件结构、数据模型、核心函数、渲染 / 播放 / 导出流程
- [开发与调试指南](docs/DEVELOPMENT.md) — 本地运行、调试入口、改后自检清单、浏览器兼容矩阵
- [开发经验与踩坑记录](docs/EXPERIENCE.md) — 迭代历史、Web Audio / Canvas 踩坑、工程铁律
- [已知问题与改进方向](docs/KNOWN_ISSUES.md) — 待修问题清单（含根因与建议修法）
