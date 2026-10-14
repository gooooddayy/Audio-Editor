# 架构与代码走读

> Audio-Editor 是一个**单文件、零依赖、纯本地、不联网**的网页音频编辑器。
> 全部代码（结构 + 样式 + 逻辑）都在根目录 `index.html` 一个文件里，没有构建步骤、没有外部 CDN / npm 包。
> 换新电脑后：拿到代码 → 双击 `index.html`（或用本地静态服务器打开）即可运行和修改。

## 1. index.html 内部结构

| 区块 | 位置 | 内容 |
|---|---|---|
| `<style>` | 文件头部 | 全部 CSS（浅色主题界面样式） |
| HTML 骨架 | 中部 | 上传区 / 工具栏 / 波形画布 / 状态栏 / 提示文案 |
| `<script>` | 尾部 | 全部 JS |

JS 内部按出现顺序分为以下逻辑组：

1. **全局状态**：`actx`、`audioBuffer`、`segments`、`cursorTime`、`selectedId`、`viewStart/viewEnd`、`rate`、`fadeDur`、播放相关状态
2. **文件加载**：`loadFile` / `resizeCanvas`
3. **坐标换算与格式化**：`timeToX` / `xToTime` / `fmtTime`
4. **波形渲染**：`render` / `drawSegWave` / `drawSegFades`
5. **交互**：画布 `click` → `handleClick`；工具栏按钮绑定
6. **编辑操作**：`splitAtCursor` / `applyFade` / `deleteAtCursor`
7. **播放**：`play` / `stop` / `getPlayheadTL` / `animLoop` / `setRate`
8. **导出**：`exportAudio` / `encodeWAV`
9. **状态栏**：`updateStatus`

## 2. 核心数据模型

### 片段（segment）—— 整个编辑器的心脏

```js
segments = [
  { id: 1, s: 0,    e: 12.5, pos: 0,    fi: 0,   fo: 0 },
  { id: 2, s: 12.5, e: 30.0, pos: 12.5, fi: 0.5, fo: 0 },
  // ...
]
```

| 字段 | 含义 |
|---|---|
| `id` | 片段唯一标识（自增）。选中用 `selectedId` 跟踪，**不用数组下标** |
| `s` / `e` | 该片段在**原音频**中的起止时间（秒） |
| `pos` | 该片段在**输出时间轴**上的位置（秒） |
| `fi` / `fo` | 淡入 / 淡出时长（秒，0 = 关闭） |

要点：

- 「原音频时间」和「输出时间轴」是两套坐标系。默认二者一致；删除片段后，后续片段 `pos` 前移闭合空白。
- 用 `selectedId`（id）而不是下标跟踪选中段，排序/增删都不会丢引用。

### 其他关键状态

| 变量 | 含义 | 说明 |
|---|---|---|
| `cursorTime` | 光标位置（输出时间轴，秒） | 点击画布放置；播放起点；分割基准 |
| `selectedId` | 当前选中片段 id | 点击画布时设为光标所在段 |
| `viewStart` / `viewEnd` | 可视窗口（秒） | **缩放功能已移除**，现恒为 `0 → 原音频时长`，波形始终全宽显示 |
| `rate` | 倍速（0.5 / 0.75 / 1 / 2 / 3） | 仅影响试听，导出恒为 1x |
| `fadeDur` | 淡变滑块值（0–5s） | 点「淡入 / 淡出」时应用到选中段 |
| `MICRO_FADE` | 0.003（3ms） | 段边界自动微淡变，消除拼接爆音 |

## 3. 核心函数速查

| 函数 | 职责 |
|---|---|
| `ensureCtx()` | 懒加载 AudioContext；suspended 时 resume（应对浏览器自动播放策略） |
| `tlDur()` | 输出时间轴总长 = max(pos + 段时长) |
| `selIdx()` | 由 `selectedId` 反查片段下标 |
| `loadFile()` | 读文件 → `decodeAudioData` → 初始化为单段 → 全宽视图 |
| `timeToX(t)` / `xToTime(x)` | 时间 ↔ 画布像素换算（当前即线性全宽映射） |
| `snapZero(idx)` | 在 ±30ms 窗口内找最近零交叉点（削爆音） |
| `segAtTime(t)` | 给定时间找所在片段下标 |
| `render()` | 主渲染：中线、各段波形、段边界线、淡变三角、光标、播放头 |
| `drawSegWave()` | 单段波形：逐像素列取 min/max 峰值；样本比像素少时逐样本画 |
| `handleClick(x)` | 点击画布：吸附零交叉 → 设光标 → 选中所在段 |
| `splitAtCursor()` | 在光标处把选中段切成两段（前段继承 fi、后段继承 fo） |
| `applyFade(isIn)` | 选中段淡入 / 淡出开关切换（再点一次取消） |
| `deleteAtCursor()` | 删除选中段，后续段前移闭合空白，首段归零；至少保留 1 段 |
| `play()` | 从 `cursorTime` 开始调度所有片段播放（见第 5 节） |
| `exportAudio()` | 离线渲染 → 去首尾空白 → 编码 WAV 下载 |
| `encodeWAV()` | 手写 16-bit PCM WAV 编码（44 字节 RIFF 头 + 交织采样） |

## 4. 渲染流程（render）

1. 清屏 → 画水平中线
2. 遍历 `segments`：选中段用蓝色 `#3370ff`，其余灰蓝 `#7c93b3`，逐段画波形
3. 画每段起点的边界线（淡灰竖线）
4. 画每段淡变区域（半透明蓝三角 + 对角线）
5. 画光标（橙色虚线 + `mm:ss:mmm` 时间标签）
6. 播放中画红色播放头

波形绘制按「像素跨度 vs 样本跨度」自动分两种模式：

- **样本比像素多**（看全貌）：每像素列统计覆盖样本的 min/max，画 1px 竖条
- **样本比像素少或相等**：逐样本画 1px 竖线

## 5. 播放模型（play）

```
对每个片段：
  schedT = playStartCtx + (max(pos, cursorTime) - cursorTime) / rate
```

- `cursorTime` 之前的片段直接跳过；光标落在段中间则用 `off = cursorTime - pos` 从偏移处播
- 每段一个 `BufferSourceNode`（`playbackRate = rate`）+ 一个 `GainNode`
- 增益包络：用户设置了淡变就用淡变时长，否则用 3ms 微淡变；淡变时长钳制到不超过片段时长一半
- 停止 = 遍历 `activeNodes` 逐个 `stop()`

> 已知行为：播放中切换倍速会从**光标位置**重新开始，而不是当前播放头位置（见 `KNOWN_ISSUES.md` 第 2 条）。

## 6. 导出流程（exportAudio）

1. `new OfflineAudioContext(声道数, ceil(总长 × 采样率), 采样率)`
2. 按每段 `pos` 调度 BufferSource + Gain（包络逻辑同播放模型）
3. `startRendering()` 得到渲染结果
4. 扫描首尾：所有声道 `|采样| ≤ 0.001` 视为空白，切掉
5. `encodeWAV()` 编码 16-bit PCM，触发下载（文件名 = 原名 + `_edited.wav`）

## 7. 交互地图

| 操作 | 入口 |
|---|---|
| 上传音频 | 上传区点击 / 把文件拖进上传区 |
| 放置光标 + 选段 | 点击波形画布（自动吸附零交叉） |
| 播放 / 停止 | ▶ 播放按钮（从光标位置开始） |
| 倍速试听 | 0.5x / 0.75x / 1x / 2x / 3x |
| 分割 | ✂ 分割（在光标处切选中段，可多次） |
| 淡入 / 淡出 | 淡入 / 淡出按钮（作用于选中段，再点一次取消） |
| 删除段 | 🗑 删除（删选中段，其余自动吸附闭合） |
| 导出 | ⬇ 导出（去首尾空白，WAV） |

## 8. 新增功能时的约束（重要）

换电脑继续开发前先读这段，能避免踩雷：

1. **保持单文件、零依赖、不联网**：不引 CDN、不拆模块、不加构建步骤（业主强偏好）
2. **每行代码验证必要性**：能删就删，避免冗余（业主强偏好）
3. **按钮必须可重复使用**：不要「点一次就禁用」，状态切换用 toggle 语义（业主强偏好）
4. **被主动移除的功能不要加回来**：缩放 / 平移 / 长按拖动片段均已移除（见 `EXPERIENCE.md` 迭代历史），要恢复必须先问业主
5. 扩展片段模型时保持 `{id, s, e, pos, fi, fo}` 的语义
6. 涉及音频拼接处，保留 3ms 微淡变机制防爆音
