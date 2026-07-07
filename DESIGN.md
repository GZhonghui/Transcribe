# 纯前端 STT 工具 — 软件设计书

> 版本 v1.0 · 2026-07-07
> 一个**纯前端**（无后端）Web 应用：用户上传音频、粘贴自己的 OpenAI API Key，应用调用 OpenAI 转写模型完成转写，
> 支持**说话人分辨**、**多语言**、**长音频（≥2 小时）**，结果可下载，并统计 **Token 用量**。第一原则：**准确性 > 一切**。

---

## 0. 核心结论速览

1. **音频大小限制**：OpenAI 硬限制 **25 MB / 单次请求**，对所有转写模型一致。限制的是文件字节数，不是时长。
2. **必须切分**：2h 音频远超 25 MB，需在浏览器端切成多个 < 25MB 的块逐块上传。
3. **切分后保证准确性/连续性**的手段：
   - **按静音切**（`silencedetect` 找停顿，绝不切在词/句中间），块长控制在 **~5min** 以内以规避输出截断。
   - **时间轴连续**：每块加时间偏移量重建全局时间戳。
   - **说话人一致**：默认「每块局部标签 + 结果页人工合并/改名」（可靠）；`known_speaker_references` 自动锚定为**实验特性**（官方未保证，见 §6.3）。
4. **多语言**：`language` 参数支持 99+ 语言（ISO-639-1）。**建议显式指定**（更准、抗语言混用），也提供自动检测。

> **可行性结论**：上传→转写→下载→多语言→单块内说话人分辨，纯前端**完全可行**。
> 唯一不确定的是「**2h 全程说话人标签统一**」——它撞上 OpenAI API 当前的未解限制（详见 §14），
> 不是纯工程问题，需先做 POC 验证并对该点降低预期 / 用人工合并兜底。

---

## 1. 需求与非目标

### 1.1 功能需求
| 编号 | 需求 | 说明 |
|---|---|---|
| F1 | 上传音频 | 拖拽/选择本地文件；常见格式（mp3/m4a/wav/flac/ogg/webm/mp4 等） |
| F2 | 用户自带 Key | 用户粘贴自己的 OpenAI API Key，应用不托管 |
| F3 | 语音转文字 | 调用 OpenAI 转写模型 |
| F4 | 说话人分辨 | 输出带 `Speaker A/B/...` 标签的对话稿 |
| F5 | 多语言 | 可选择转写语言或自动检测 |
| F6 | 长音频 | 支持 ≥ 2 小时（默认上限 2h，可配置放宽） |
| F7 | 下载 | 导出 `txt` / `md` / `srt` / `vtt` / `json` |
| F8 | 准确性优先 | 默认参数一律向「更准」倾斜，哪怕更慢/更贵 |
| F9 | Token 用量统计 | 逐块读取响应自带的 `usage`，实时累计并展示 token 明细与估算成本 |

### 1.2 非目标（当前版本）
- 不做实时/流式转写（只做上传离线转写）。
- 不做后端、账户系统、流量代理（Key 全程留在浏览器）。
- 不追求 > 4 人的自动全局说话人一致（受 `known_speaker_references` 上限限制，见 §6.3）。

---

## 2. OpenAI 转写模型选型

| 模型 | 说话人分辨 | 时间戳 | `prompt` | 计价 | 适用 |
|---|---|---|---|---|---|
| **`gpt-4o-transcribe-diarize`** | ✅ 原生（`diarized_json`） | segment 级 | ❌ | 按 token：$2.50/M 输入音频 + $10/M 输出 | **多人对话/会议/访谈（主模式）** |
| `gpt-4o-transcribe` | ❌ | ✅ `verbose_json`（句级） | ✅ | $0.006/min | 单人/最高文字准确率 |
| `gpt-4o-mini-transcribe` | ❌ | ✅ | ✅ | $0.003/min | 成本敏感 |
| `whisper-1`（legacy） | ❌ | ✅ word/segment 级 | ✅ | $0.006/min | 需要 word 级时间戳时 |

**两种工作模式：**
- **模式 A（默认，多人）**：`gpt-4o-transcribe-diarize` —— 唯一原生分辨说话人的官方模型。
- **模式 B（单人/极致文字准确率）**：`gpt-4o-transcribe`（或 `whisper-1` 取 word 级时间戳）。可用 `prompt` 传上一块尾部文本保持术语/拼写连续，代价是没有说话人标签。

> ⚠️ diarize 模型已知问题（官方社区反馈）：偶尔幻觉多余说话人、漏读个别句子、指定语言后仍可能语言混用。缓解手段见 §7。

---

## 3. 系统架构

```
┌──────────────────────────── 浏览器（唯一运行环境） ─────────────────────────────┐
│  UI 层 (Vue 3 + TypeScript)                                                      │
│   ├─ 配置面板 / 进度视图 / 结果视图                                               │
│  核心逻辑层 (Web Worker，避免卡 UI)                                               │
│   ├─ AudioPreprocessor  ── ffmpeg.wasm：转码/降采样/静音检测/切块                  │
│   ├─ Chunker            ── 静音感知切分（§5）                                      │
│   ├─ TranscriptionClient── 逐块调用 OpenAI（raw fetch，非 SDK，§8）                │
│   ├─ Merger             ── 时间戳重建 + 段合并                                     │
│   └─ UsageTracker       ── 读取每块 usage，累计 token/成本（§6.4）                 │
│  存储                                                                            │
│   ├─ Key：内存 / sessionStorage（默认不落盘，"记住"→localStorage 并警告）          │
│   └─ 中间结果 & 断点续传：IndexedDB（每块结果持久化，刷新可恢复）                   │
└──────────────────────────────────────────────────────────────────────────────┘
                    │ HTTPS（唯一外部依赖）
                    ▼   api.openai.com/v1/audio/transcriptions
```

**技术栈：** Vite + **Vue 3（Composition API）** + TypeScript（只用这一个框架）；状态用 Pinia 或 `reactive`；
`@ffmpeg/ffmpeg`（ffmpeg.wasm）；音频处理放 Web Worker；不引入 OpenAI SDK（CORS 原因，见 §8），用原生 `fetch` + `FormData`。
核心逻辑（切分/转写/合并/用量）都在与框架无关的 Worker / 纯 TS 模块里，UI 层只做展示交互——框架可替换但**始终只用一个**。

---

## 4. 端到端数据流

```
上传文件
  → [预处理] ffmpeg.wasm：解码 → 降混单声道 → 重采样 16kHz → 输出 FLAC(无损)
  → [静音检测] silencedetect → 停顿点列表
  → [切分] 目标 ~5min + 就近静音点，切成 chunk[0..N]（每块 < 上限 & < 7min，§5.3）
  → for each chunk：
        ├─ 组装请求（model / language / response_format[/ chunking_strategy]）
        ├─ POST 上传 → 得到段 [{speaker,start,end,text}] + usage
        ├─ 读 usage 累加用量（§6.4）
        └─ 写入 IndexedDB（断点续传）
  → [合并] 每段时间戳 += chunk 偏移量；相邻同说话人短间隔合并
  → [说话人] 默认保留每块局部标签，结果页人工合并/改名（§6.3）
  → [导出] txt / md / srt / vtt / json
```

---

## 5. 音频预处理与切分（准确性的地基）

### 5.1 预处理（ffmpeg.wasm）
统一转成**语音识别友好且体积小的无损格式**：
```
ffmpeg -i input.<any> -ac 1 -ar 16000 -c:a flac output.flac
```
- `-ac 1` 单声道、`-ar 16000` 16kHz（语音识别标准采样率）。
- **FLAC 无损**：约为同规格 WAV 的一半体积、不丢信息 → 兼顾准确性与上传体积。16kHz 单声道 FLAC ≈ 0.4–0.8 MB/min。
- **例外**：若源是「每人独占一个声道」的双声道录音，提供「保留声道」选项（不降混）以利分辨。

### 5.2 静音检测
```
ffmpeg -i output.flac -af silencedetect=noise=-30dB:d=0.5 -f null -
```
从 stderr 解析 `silence_start`/`silence_end`，取每段静音中点为候选切点。`noise`（阈值）、`d`（最短静音）作为高级参数暴露。

### 5.3 静音感知切分算法
目标：块尽量长（少边界=更连续），但**必须在静音处切**，且受**输出截断**约束——
diarize/transcribe 约 8–9 分钟后会触发 2000 输出 token 截断，另有 1400s(~23min) 硬上限。故目标块长取 **~5min**，硬上限 ~7min。

```js
const TARGET = 300;   // 目标块时长(s)，~5min，留足余量规避 8–9min 截断
const HARD_MAX = 420; // 单块硬上限(s)，~7min（< 8min 截断线 & < 1400s API 上限）

function planChunks(duration, silencePoints) {
  const cuts = [0]; let cursor = 0;
  while (duration - cursor > HARD_MAX) {
    // 在 [cursor+TARGET, cursor+HARD_MAX] 窗口内找最近静音点
    const cand = silencePoints.filter(p => p >= cursor + TARGET && p <= cursor + HARD_MAX);
    const cut = cand.length ? cand[0]           // 就近静音切（干净）
                            : (markOverlap(cursor + HARD_MAX), cursor + HARD_MAX); // 无停顿→硬切+overlap
    cuts.push(cut); cursor = cut;
  }
  cuts.push(duration);
  return toRanges(cuts); // [{start,end,hardCut?}]
}
```
**边界补偿（仅硬切）**：硬切点两侧各加 2–3s overlap，合并时对 overlap 区做文本对齐去重，既不丢边界词也不重复。静音切通常无需 overlap。
最后用 ffmpeg 精确截取每块（`-ss`/`-to`）产出 `chunk_k.flac`。

---

## 6. 转写与说话人分辨

### 6.1 请求参数
`POST https://api.openai.com/v1/audio/transcriptions`（`multipart/form-data`）

| 字段 | 模式 A（diarize） | 模式 B | 说明 |
|---|---|---|---|
| `file` | `chunk_k.flac` | 同 | < 25MB |
| `model` | `gpt-4o-transcribe-diarize` | `gpt-4o-transcribe` | |
| `response_format` | `diarized_json` | `verbose_json` | |
| `language` | 用户所选（如 `zh`） | 同 | 显式指定，抗语言混用；"自动"时省略 |
| `chunking_strategy` | 取舍项（见 §7） | — | > 30s 时 diarize 需要此参数才不报错；`auto` vs 不加需实测 |
| `temperature` | — | `0` | 降低随机幻觉 |
| `prompt` | ❌ 不支持 | 上一块尾部 ~200 字符 | 保持术语/拼写连续 |
| `known_speaker_names[]` / `known_speaker_references[]` | 实验，见 §6.3 | — | 最多 4 人，参考片段 2–10s |

**返回**（`diarized_json`）：
```json
{ "segments": [ { "speaker": "Speaker A", "start": 0.0, "end": 4.2, "text": "……" }, ... ],
  "usage": { "type": "duration", "seconds": 27 } }
```

### 6.2 时间戳重建与段合并
```js
seg.globalStart = chunk.offset + seg.start;
seg.globalEnd   = chunk.offset + seg.end;
if (sameSpeaker(prev, cur) && cur.globalStart - prev.globalEnd < 0.8) merge(prev, cur);
```

### 6.3 跨块说话人一致性
diarize **逐请求**独立分配标签——第 1 块的 `Speaker A` 到第 2 块可能变成 `Speaker B`。官方能力**只保证单块内一致**，
跨块全局统一目前**无可靠的纯 OpenAI 方案**（见 §14）。因此分两层：

**A) 默认基线（可靠）：局部标签 + 人工合并。**
每块保留局部标签（如 `块k-A`），结果页提供说话人**合并/改名**交互，用户点几下即可把「块1-A / 块2-B / 块3-A」并为「张三」。不依赖任何未验证假设。

**B) 可选实验：`known_speaker_references` 自动锚定。**
从上一块为每个说话人抽取 2–10s 参考片段，作为下一块的 `known_speaker_references` 传入以「锚定」标签。
```js
let refs = []; // [{name, clipDataUrl}]，最多 4 个（API 上限）
for (let k = 0; k < chunks.length; k++) {
  const form = buildForm(chunks[k], {model:"gpt-4o-transcribe-diarize", response_format:"diarized_json", language});
  refs.forEach(r => { form.append("known_speaker_names[]", r.name);
                      form.append("known_speaker_references[]", r.clipDataUrl); });
  const { segments } = await transcribe(form);
  results.push({ offset: chunks[k].start, segments });
  for (const spk of distinctSpeakers(segments)) {            // 补充参考库
    if (refs.find(r => r.name === spk) || refs.length >= 4) continue;
    const seg = bestSampleSegment(segments, spk);            // 2–10s、非重叠、能量足够
    refs.push({ name: spk, clipDataUrl: await extractClip(originalAudio, chunks[k].start + seg.start) });
  }
}
```
**风险（默认关闭，须先验证）**：官方未承诺此用法有效，需 POC 实测跨块命中率；上限 4 人；且第 1 块若切分有误会**传播错误**到后续所有块——这点比人工合并更危险。>4 人或验证不达标一律退回基线 A。

### 6.4 Token 用量统计（响应自带，精确读取）
响应**默认带 `usage`**（无需 `include`），逐块累加即可，**不用估算**。不同模型单位不同，需兼容两种：

| 模型 / 格式 | `usage` 形态 |
|---|---|
| `gpt-4o-transcribe` / `mini`（json） | `{type:"tokens", input_tokens, output_tokens, total_tokens, input_token_details:{audio_tokens, text_tokens}}` |
| `whisper-1` / `diarized_json` | `{type:"duration", seconds}`（diarize 是否返回 token 待 POC 确认） |

```js
function accumulate(stats, usage, model) {
  stats.perChunk.push({ model, usage });
  if (usage.type === "tokens") {
    stats.inputTokens  += usage.input_tokens  ?? 0;
    stats.outputTokens += usage.output_tokens ?? 0;
    stats.audioTokens  += usage.input_token_details?.audio_tokens ?? 0;
    stats.totalTokens  += usage.total_tokens  ?? 0;
    stats.costUSD      += costByTokens(model, usage);         // 精确
  } else if (usage.type === "duration") {
    stats.durationSec  += usage.seconds ?? 0;
    stats.costUSD      += costByMinutes(model, usage.seconds);// 按分钟估算
    stats.estimated     = true;
  }
}
```
- `tokens` 是实际计费口径 → 精确成本；`duration` → 按费率表估算并标注「按时长估算」。
- 保留每块明细（模型、token/秒数、耗时、成本）供结果页表格与 CSV 导出；实时刷新 UI；随结果存 IndexedDB，续传不重复计。

---

## 7. 准确性策略汇总（"准确性 > 一切"）

1. **无损 16kHz 单声道 FLAC** 上传，不用有损压缩丢信息。
2. **静音感知切分**，绝不切在词/句中间；硬切点用 overlap + 文本去重兜底。
3. **显式指定 `language`**，抗语言混用与语种漂移。
4. **块长 ~5min**（§5.3），从源头规避 8–9min 输出截断。
5. **`chunking_strategy` 是取舍项**：不加→停顿处易截断；`auto`→不截断但噪声段会幻觉、准确率可能下降。对目标音频实测两种设置再定；块够短时可少依赖它。
6. **块级校验 + 重试**：每块检查末段 `end` 是否接近块时长（差距大=疑似截断）、时长-字数比、是否空文本；命中则缩短块长/换参数重试一次，仍异常则标红提示复核。这是抵御截断的最后防线。
7. `temperature=0`（模式 B）：降低随机幻觉。
8. **模式 B 的 `prompt` 续接**：跨块术语/专名/拼写一致。
9. **（进阶）二次核对**：低置信/关键块用另一模型或换参数复转做交叉比对。

---

## 8. 与 OpenAI 的接口细节

### 8.1 CORS：用原生 `fetch`，不用官方 SDK
官方 JS SDK 会附带 `X-Stainless-*` 等自定义头，触发浏览器 CORS 预检失败（且 SDK 默认禁止浏览器运行）。
**做法**：直接 `fetch` + `FormData`，只带 `Authorization` 头（`Content-Type` 由浏览器自动带 boundary，勿手动设）。
```js
await fetch("https://api.openai.com/v1/audio/transcriptions", {
  method: "POST",
  headers: { "Authorization": `Bearer ${apiKey}` }, // 不加任何 X-* 自定义头
  body: form,                                        // FormData
});
```
> ⚠️ 浏览器直连**尚需 POC 实测确认**（社区有「浏览器来源请求被临时封锁」的零星报告）。
> **降级方案**：提供可选「自建代理 URL」配置项（如极简 Cloudflare Worker 透传）。默认直连，代理仅作降级，不违背纯前端初衷。

### 8.2 并发、限速与重试
- **顺序处理块**（模式 B 可小并发 2–3；若启用 §6.3-B 锚定则必须顺序）。
- `429`/`5xx`：指数退避重试（优先 `Retry-After`），最多 N 次。
- 大文件上传用 `fetch`/`XHR` + 进度事件展示上传百分比。

### 8.3 成本估算
- 模式 B：`gpt-4o-transcribe` $0.006/min → 2h ≈ $0.72；mini ≈ $0.36。
- 模式 A：diarize 按 token 计，2h 约几毛~$1，与模式 B 同量级。
- **转写前**用时长×费率给静态预估；**转写中/后**用响应真实 `usage` 精确统计（§6.4），UI 实时展示（§9.3）。

---

## 9. UI / UX 设计

### 9.0 整体布局：左配置 + 右主区（两栏）
**极简、两栏、无多步向导。** 左侧一个窄配置面板（选文件 + 填配置 + 开始），右侧一个宽主工作区，
按状态显示「空闲提示 / 转写进度 / 转写结果」。大量留白，一个强调色 + 中性灰，默认浅色、支持深色。
响应式：窄屏时左面板折叠为顶部抽屉，主区占满。
```
┌──── 配置（左·窄，固定） ────┬──────── 主工作区（右·宽） ────────┐
│  ⬆ 选择音频                 │                                    │
│  API Key                   │  空闲：提示「配置好左侧后开始」       │
│  语言 / 模式                │  转写中：进度（§9.2 ②）             │
│  ▸ 高级设置                 │  完成：结果（§9.2 ③）               │
│  [ 开始转写 ]               │                                    │
│  ── 用量小结（实时）──       │                                    │
└────────────────────────────┴────────────────────────────────────┘
```

### 9.1 左侧配置面板
```
┌────────── 配置 ──────────┐
│ ┌─────────────────────┐  │
│ │ ⬆ 拖拽音频 / 点击选择 │  │  ← 选中后显示 文件名·时长·大小
│ └─────────────────────┘  │
│ API Key [••••••••] 👁     │  ← 密码框，👁 切换明文
│ ☐ 记住 Key（本机存储）⚠   │  ← 勾选才写 localStorage
│ 语言 [中文 ▾]             │  ← 含「自动检测」
│ 模式 [分辨说话人 ▾]        │  ← 分辨说话人 / 最高准确率
│ ▸ 高级设置（默认折叠）      │  ← 见下
│ 预计 ~24 块 · 约 2h · $0.8│  ← 开始前静态预估
│ [ 开始转写 ]              │  ← Key/文件缺失时禁用
│──────────────────────────│
│ 累计 152k tok · ≈ $0.62   │  ← 用量小结，转写中实时刷新（§9.3）
└──────────────────────────┘
```
**高级设置：** 目标块长（默认 5min）、静音阈值(dB)/最短静音(s)、`chunking_strategy`（无/auto）、
采样率（16k/24k）、保留声道、并发数（模式 B）、说话人自动锚定开关（§6.3-B，默认关）、自定义代理 URL、`temperature`。均带默认值与一句话说明。

### 9.2 右侧主工作区（三态，同区切换）
**① 空闲态**：居中提示「在左侧选择音频并配置，点击开始转写」。

**② 进度态**
```
┌──────────────── 转写进度 ────────────────┐
│ ▓▓▓▓▓▓▓▓▓░░░░░░  第 7/24 块 · 转写中       │  ← 总进度条 + 文字
│ 已用 03:12 · 预计剩余 04:40                │
│ 块 01  00:00–04:58  ✓ 完成  38k tok       │  ← 每块：序号·时间范围·状态·用量
│ 块 07  28:10–32:55  ⟳ 转写中              │
│ 块 08  32:55–37:30  · 排队                │
│              [ 暂停 ]  [ 取消 ]           │
└───────────────────────────────────────────┘
```
状态徽章：`切分 → 上传N% → 转写中 → ✓完成 / ⚠重试 / ✕失败`，失败块可单独重试。
断点续传：刷新后已完成块从 IndexedDB 恢复，不重跑、不重复计费。

**③ 结果态**
```
┌──────────────────── 转写结果 ────────────────────┐
│ 🔍搜索  👥说话人(改名/合并)  ⬇导出▾  ⧉复制  ✎编辑 │  ← 工具条
├───────────────────────────────────────────────────┤
│ ● 张三  [00:04]  今天我们讨论一下……               │  ← 色点+名字+时间戳+文本
│ ● 李四  [00:11]  我觉得第一点……                   │     行内可编辑；点时间戳跳转
│ ● 张三  [00:19]  对，另外……                       │
├───────────────────────────────────────────────────┤
│ ▶ ━━━━●───────────────  00:19 / 2:03:11          │  ← 播放器，与对话稿高亮同步
├───────────────────────────────────────────────────┤
│ ▸ 用量与成本摘要（可展开，见 §9.3）                 │
└───────────────────────────────────────────────────┘
```
**说话人管理弹层**：列出检测到的说话人 → 改名 / 合并（把「块3-A」并入「张三」）/ 指定颜色。对应 §6.3 基线 A。

### 9.3 Token 用量统计
同一数据源（§6.4），两处呈现：
- **左面板底部小结（常驻）**：累计 token + 估算成本，转写中随每块实时刷新。
- **右侧结果态摘要卡（可展开明细表）**：
```
┌ 用量与成本摘要 ─────────────────────────────────────┐
│ 模型 gpt-4o-transcribe-diarize · 24 块 · 音频 2:03:11 │
│ 输入 128,400 tok（音频 126,900 / 文本 1,500）         │
│ 输出  24,100 tok      合计 152,500 tok                │
│ 估算成本 ≈ $0.62   ⓘ 按当前公开费率估算，非账单       │
│ ────────────── 每块明细（可折叠） ─────────────────  │
│ 块 时间范围      模型     输入tok 输出tok 耗时  $     │
│ 01 00:00–04:58  diarize   38,200  6,100  6.2s .15    │
│                       [ 导出用量 CSV/JSON ]           │
└───────────────────────────────────────────────────────┘
```
若模型只返回 `duration`（whisper-1／可能的 diarize），该列显示**秒数**并标注「按时长估算」。费率表内置且可在设置里覆盖。

### 9.4 导出
`txt` / `md`（`**张三** [00:04] ……` 对话稿）/ `srt` / `vtt` / `json`（全部段 + 全局时间戳 + 说话人 + 每块 usage）/ 用量 `csv`。

---

## 10. 安全与隐私
- **Key**：默认只存内存 + `sessionStorage`（关标签即清）；「记住」才写 `localStorage` 并警告（XSS / 共享设备风险）。绝不上传到 OpenAI 以外任何地方。
- **音频**：全程本地处理，仅音频块上传至 OpenAI；IndexedDB 中间结果可一键清除。
- **CSP**：`connect-src` 仅允许 `api.openai.com`（及用户配置的代理）。
- UI 明示「本应用无后端，音频与 Key 只发往 OpenAI」，并提示勿上传敏感/违规内容。

---

## 11. 部署（关键约束：跨源隔离）
ffmpeg.wasm 多线程构建依赖 `SharedArrayBuffer`，要求页面跨源隔离：
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```
- **Vercel / Netlify / Cloudflare Pages**：直接配置这两个响应头（推荐）。
- **GitHub Pages**（不能设响应头）：用 `coi-serviceworker.js` 客户端注入，或用 ffmpeg.wasm 单线程构建（不需 SAB，较慢）。
- 所有资源（含 ~31MB 的 ffmpeg-core.wasm）需同源或满足 CORP，避免被 COEP 拦截。

---

## 12. 错误处理与边界情况
| 场景 | 处理 |
|---|---|
| 文件解码失败/格式不支持 | ffmpeg 兜底转码；仍失败则提示 |
| 某块仍 > 25MB | 缩短目标块长重切该块 |
| 连续讲话无静音点 | 硬切 + overlap（§5.3） |
| 疑似截断（末段 end 远小于块长） | 缩短块长/换参数重试（§7 第 6 条） |
| diarize 幻觉多余说话人 | 结果页手动合并 |
| 语言混用 | 显式 `language` + 提示改语言重转 |
| 429/5xx | 退避重试；持久失败允许单块手动重试 |
| 刷新/断网 | IndexedDB 断点续传，已完成块不重跑 |
| >4 说话人 | 局部标签 + 手动合并 |

---

## 13. 里程碑
> 逐项开发进度追踪见 [TODO.md](./TODO.md)（勾选式清单，按 P0 POC + M1~M5 组织）。

1. **M1 骨架**：上传 + Key + 单块（<25MB）直连转写 + 读 `usage` 显示 token + 纯文本下载（打通链路）。
2. **M2 长音频**：集成 ffmpeg.wasm、静音切分、时间戳重建、多块合并；部署跨源隔离。
3. **M3 说话人**：diarize 模式 + 对话稿 UI + 说话人手动合并/改名 + srt/vtt。
4. **M4 健壮性/准确性**：模式 B + prompt 续接、块级校验重试、断点续传、Token 用量统计与成本（F9）、多语言完善。
5. **M5 进阶**：`known_speaker_references` 自动锚定（验证后）、overlap 去重、二次核对、>4 人客户端聚类。

---

## 14. 主要风险与限制（按严重程度）
- **🔴 R1 跨块说话人一致性 = open problem**：官方只保证单块内一致，全程统一标签**无可靠纯 OpenAI 方案**。默认走人工合并；自动锚定需验证；>4 人无法自动锚定。**本项目最大不确定性。**
- **🔴 R2 长块截断 + chunking 取舍**：~8–9min 截断、1400s 硬上限；`chunking_strategy=auto` 消截断但降准确率。靠短块（~5min）+ 块级校验缓解，但无法根除模型侧漏句/幻觉。
- **🟡 CORS 直连未实测**：原生 fetch 理论可行，受 OpenAI 侧策略影响，需 POC + 代理降级。
- **🟡 ffmpeg.wasm 性能/内存**：2h 音频吃内存（可能撞 wasm 上限），需流式/分段处理，低配设备体验差。
- **🟢 成本**：diarize 按 token 计，2h 约几毛~$1，与模式 B 同量级，转写前动态预估即可。

### 14.1 上线前必做的 3 个去风险 POC
1. **说话人跨块**：15–20min 多人音频切 3–4 块，测 `known_speaker_references` 锚定命中率 → 决定 §6.3-B 是否启用。
2. **截断/分块**：同段音频对比「不加 chunking / auto / 短块」的漏句率与幻觉率 → 定块长与 chunking 策略。
3. **CORS 直连**：静态页原生 fetch 直接 POST 一个音频块到 OpenAI → 决定是否需要代理。

---

## 附录：参考资料
- OpenAI Audio API FAQ / Speech-to-text 指南（25MB 限制、`chunking_strategy`、`diarized_json`、`known_speaker_*`、`language`、`usage`）。
- `gpt-4o-transcribe-diarize` 模型页（16k 上下文、token 计价）；Create transcription API 参考（各模型 usage 单位）。
- ffmpeg.wasm 文档、`silencedetect` 滤镜、coi-serviceworker（GitHub Pages 跨源隔离）。
