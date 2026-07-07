# STT 工具 — 开发进度 TODO

> 配套设计书见 [DESIGN.md](./DESIGN.md)。勾选记录进度：`- [ ]` 未开始 · `- [x]` 完成。
> 每项后括号标注对应设计书章节。**建议先做 P0 的 POC**——其结论直接决定后续默认参数与架构选型。

---

## P0 · 去风险 POC（最先做，§14.1）
> 浏览器实跑工具：[poc/cors-usage-poc.html](./poc/cors-usage-poc.html)（粘 Key + 拖音频）。带 Key/音频的付费 POC 暂缓。

### 已确认（官方文档核对 + curl 实测响应头，无需 Key，2026-07-07）
- [x] **POC-1 CORS 直连 → 可直连，无需代理**。curl 实测 `api.openai.com/v1/audio/transcriptions`：preflight `OPTIONS→200`；`allow-methods: GET,OPTIONS,POST`；`allow-headers` 回显所请求头（含 `authorization`，连 `content-type`/`x-stainless-*` 也放行）；`allow-origin` 回显任意 Origin，无 auth 的 POST 返回 `*`；`max-age 86400`。CORS 结果完全由响应头决定，故确认：浏览器原生 `fetch`+`FormData`（仅带 `Authorization`、不手设 `Content-Type`）会放行 → **代理从默认降级为可选后备**（企业网关场景仍可能需要）（§8.1）
- [x] **输入硬限制**（官方 Speech-to-text 指南）：25MB / 请求（限字节数非时长）；官方受支持格式 `mp3 / mp4 / mpeg / mpga / m4a / wav / webm`
- [x] **diarize 参数形态**（官方指南，均与 DESIGN 一致）：`response_format` 支持 `json / text / diarized_json`（后者才带 speaker 段）；音频 >30s **必填** `chunking_strategy`（推荐 `auto`）；**不支持** `prompt` / `logprobs` / `timestamp_granularities[]`；`known_speaker_names[]` / `known_speaker_references[]` 最多 **4 人**、参考片段 **2–10s**、multipart 下编码为 **data URL**
- [x] **POC-3 先验**（官方社区专帖）：diarize **只保证块内一致**；`known_speaker_references` 跨块锚定**无官方文档、无已验证成功案例**（有用户最终自训模型）→ 坚持 **基线 A（局部标签 + 人工合并）为默认**、基线 B 保持实验关闭。这仍是最大风险 R1；实测只为量化命中率，不改默认策略

### 需回填 DESIGN 的两处修正（本轮确认，待改文档）
- [ ] **修正#1 上传格式**：§5.1 预处理输出 **FLAC**，但 FLAC / ogg **不在**官方受支持列表 → 先按 **`wav` 兜底**（16kHz 单声道 wav ≈ FLAC 两倍体积，但仍远 <25MB/块）；FLAC 是否被新 gpt-4o 系列接受留待实测
- [ ] **修正#2 SDK/CORS 因果**：§8.1「SDK 的 `X-Stainless-*` 头触发 CORS 预检失败」**实测不成立**（这些头会被回显放行）；SDK 不能在浏览器跑的真正原因是其客户端守卫 **`dangerouslyAllowBrowser`**。「用原生 fetch，不用 SDK」结论不变，仅需改写理由

### 暂缓 · 需真 Key + 真音频（付费调用，用 poc.html 跑）
- [ ] POC-旁 `usage` 形态：diarize 到底返回 `tokens` 还是 `duration`（§6.4 悬而未决）—— poc.html 会高亮 `usage.type`
- [ ] POC-1 端到端 200 复核 + FLAC 格式实测（顺带验证修正#1）
- [ ] POC-2 分块：同段 15–20min 音频对比「不加 chunking / auto / 5min 短块」的漏句率与幻觉率 → 定块长与 `chunking_strategy`（§5.3 / §7）
- [ ] POC-3 实测：多块 + `known_speaker_references` 跨块命中率（poc.html 待扩展此模式）→ 决定 §6.3-B 是否启用

---

## M1 · 骨架（打通链路，§13）
- [ ] 初始化 Vite + Vue 3 + TS + Pinia 项目（§3）
- [ ] 左配置面板：文件拖拽/选择 + API Key 输入（密码框 / 👁 切换 / 记住）（§9.1）
- [ ] 语言下拉 + 模式选择（§9.1）
- [ ] TranscriptionClient：原生 fetch + FormData 单块转写（<25MB，暂不分块）（§8.1）
- [ ] 读取响应 `usage`，显示 token/时长（§6.4）
- [ ] 右主区展示转写文本 + 纯文本 `txt` 下载（§9.2③ / §9.4）
- [ ] Key 存储：内存 / sessionStorage，可选 localStorage + 风险警告（§10）
- [ ] 基础错误处理（401 / 429 / 网络异常）（§12）

---

## M2 · 长音频（切分与合并，§13）
- [ ] 集成 ffmpeg.wasm + 跨源隔离（COOP/COEP，本地开发也需）（§11）
- [ ] AudioPreprocessor：解码 → 单声道 → 16kHz → FLAC（§5.1）
- [ ] 静音检测：`silencedetect` 解析停顿点（§5.2）
- [ ] Chunker：静音感知切分（TARGET 300s / HARD_MAX 420s）（§5.3）
- [ ] 硬切 overlap 标记（合并去重留到 M5）（§5.3）
- [ ] 逐块顺序上传 → 时间戳偏移重建 → 段合并（§4 / §6.2）
- [ ] 右主区进度态：总进度条 + 每块状态列表 + 暂停/取消（§9.2②）
- [ ] IndexedDB 中间结果持久化 + 断点续传（§4 / §12）
- [ ] 核心逻辑迁入 Web Worker，避免卡 UI（§3）

---

## M3 · 说话人分辨（§13）
- [ ] 模式 A：`gpt-4o-transcribe-diarize` + `diarized_json`（§6.1）
- [ ] 解析局部说话人标签（基线 A）（§6.3-A）
- [ ] 结果态对话稿 UI：色点 + 名字 + 时间戳 + 文本（§9.2③）
- [ ] 说话人管理弹层：改名 / 合并 / 指定颜色（§9.2③ / §6.3-A）
- [ ] 音频播放器：播放位置与对话稿高亮同步、点时间戳跳转（§9.2③）
- [ ] 导出 `srt` / `vtt`（§9.4）
- [ ] （实验，视 POC-3）`known_speaker_references` 自动锚定开关（§6.3-B）

---

## M4 · 健壮性 / 准确性 / Token 统计（§13）
- [ ] 模式 B：`gpt-4o-transcribe` + `verbose_json` + `temperature=0`（§6.1 / §7）
- [ ] 模式 B `prompt` 续接（上一块尾部 ~200 字符）（§7）
- [ ] 块级完整性校验（末段 end / 时长-字数比 / 空文本）+ 自动重试（§7）
- [ ] 429/5xx 指数退避重试（`Retry-After` 优先）（§8.2）
- [ ] UsageTracker：兼容 `tokens` / `duration` 两种单位累加（§6.4）
- [ ] 左面板常驻用量小结（实时刷新）（§9.3）
- [ ] 结果态用量摘要卡 + 每块明细表 + 费率表（可覆盖）（§9.3）
- [ ] 用量 `csv` / `json` 导出（§9.4）
- [ ] 开始前静态成本预估（§8.3 / §9.1）
- [ ] 高级设置面板：块长 / 静音阈值 / chunking / 采样率 / 声道 / 并发 / 代理 URL / temperature（§9.1）
- [ ] 多语言完善：语言列表 + 自动检测（§0.4 / §6.1）
- [ ] `md` / `json` 导出格式（§9.4）

---

## M5 · 进阶（可选，§13）
- [ ] overlap 区文本对齐去重（词级 / 最长公共子串）（§5.3）
- [ ] 二次核对：低置信块换模型/参数复转交叉比对（§7）
- [ ] >4 人客户端 speaker-embedding 聚类（transformers.js / WebGPU）（§6.3）
- [ ] 波形缩略图（§9.1）

---

## 部署 / 杂项
- [ ] CSP：`connect-src` 限 `api.openai.com`（+ 用户代理）（§10）
- [ ] 代理降级方案（极简 Cloudflare Worker 透传）+ 文档（§8.1）
- [ ] 部署：Vercel/Netlify/CF Pages 配 COOP/COEP，或 GitHub Pages + coi-serviceworker（§11）
- [ ] 深色模式（§9.0）
- [ ] 响应式：窄屏左面板收为顶部抽屉（§9.0）
- [ ] README + 使用说明 + 隐私声明（§10）
