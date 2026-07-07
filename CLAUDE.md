# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目状态

**这是一个尚未开始编码的项目。** 当前仓库只有设计文档，没有任何源码、`package.json` 或构建系统。

- **[DESIGN.md](./DESIGN.md) 是唯一的事实来源（source of truth）**——完整的软件设计书，任何实现决策都应先查它。下面的内容只是提炼其中跨多节、必须先内化的关键约束，不重复细节。
- **[TODO.md](./TODO.md) 是进度清单**，按 `P0 POC + M1~M5` 里程碑组织。**完成一项就勾选对应 `- [ ]` → `- [x]`**，保持它与实际进度同步。
- 计划技术栈（尚未搭建）：Vite + **Vue 3（Composition API）** + TypeScript + Pinia；`@ffmpeg/ffmpeg`（ffmpeg.wasm）。搭建脚手架时用这套；**始终只用一个 UI 框架**。届时才会有 `npm` / `vite` 相关的 build/dev/test 命令——现在没有，不要臆造。

## 这是什么

**纯前端（无后端）STT Web 应用**：用户上传音频 + 粘贴自己的 OpenAI API Key，浏览器端调用 OpenAI 转写模型完成转写，支持说话人分辨、多语言、长音频（≥2h），结果可下载，并统计 token 用量。

**第一原则：准确性 > 一切。** 所有默认参数一律向「更准」倾斜，哪怕更慢/更贵。这条会反复决定取舍，不要为了性能牺牲准确性。

## 架构骨架（大图）

**纯前端、无后端**：Key 全程留在浏览器，唯一外部请求是 `api.openai.com`。不做后端 / 账户 / 流量代理。

分层（DESIGN.md §3）：**UI 层（Vue，只做展示交互，可替换）** ⟶ **核心逻辑层（框架无关的 Web Worker / 纯 TS 模块）**。核心逻辑 = `AudioPreprocessor`（ffmpeg.wasm 转码/切块）、`Chunker`（静音感知切分）、`TranscriptionClient`（逐块调 OpenAI）、`Merger`（时间戳重建+段合并）、`UsageTracker`（累计 token/成本）。**核心逻辑必须与框架解耦**，放 Worker/纯 TS，UI 只调用它。

存储：Key 存内存/`sessionStorage`（默认不落盘，「记住」才写 `localStorage` 并警告）；中间结果 + 断点续传存 **IndexedDB**（每块结果持久化，刷新可恢复，不重跑不重复计费）。

## 必须先内化的约束（跨节，容易踩坑）

1. **不用 OpenAI SDK——直接 `fetch` + `FormData`。** SDK 会附带 `X-Stainless-*` 头触发 CORS 预检失败且默认禁止浏览器运行。只带 `Authorization` 头，**不要手动设 `Content-Type`**（让浏览器自动带 multipart boundary）。（§8.1）
2. **25MB / 单次请求是 OpenAI 硬限制**（限字节数不是时长），所以长音频**必须切块**。
3. **块长 ~5min（目标 300s / 硬上限 420s），且必须在静音处切。** 因为 diarize/transcribe 约 8–9min 会触发 2000 输出 token 截断，另有 1400s(~23min) 硬上限。绝不切在词/句中间；无静音点才硬切并加 overlap。（§5.3）
4. **跨块说话人一致性是本项目最大风险（🔴 R1，open problem）。** diarize 逐请求独立分配标签，官方只保证**单块内**一致，全程统一标签无可靠纯 OpenAI 方案。**默认走「局部标签 + 结果页人工合并/改名」（基线 A，可靠）**；`known_speaker_references` 自动锚定是**实验特性（基线 B，默认关闭）**，未验证前不要当默认路径，>4 人也无法锚定。（§6.3 / §14）
5. **两种模式**：模式 A = `gpt-4o-transcribe-diarize` + `diarized_json`（多人，唯一原生分辨说话人）；模式 B = `gpt-4o-transcribe` + `verbose_json` + `temperature=0` + `prompt` 续接（单人/极致文字准确率，无说话人标签）。
6. **用量统计读响应自带的 `usage`，不估算。** 需兼容两种形态：`{type:"tokens", ...}`（精确成本）与 `{type:"duration", seconds}`（按分钟估算并标注「按时长估算」）。diarize 到底返回哪种待 POC 确认。（§6.4）
7. **部署/本地开发都需跨源隔离**：ffmpeg.wasm 多线程依赖 `SharedArrayBuffer`，页面必须发送 `COOP: same-origin` + `COEP: require-corp`。GitHub Pages 无法设响应头 → 用 `coi-serviceworker.js` 或单线程构建。（§11）

## 工作顺序

**先做 P0 的 3 个去风险 POC**（§14.1 / TODO.md P0），它们的结论直接决定后续默认参数与架构：① CORS 浏览器直连是否可行（否则需代理降级）；② 截断/分块策略（`chunking_strategy` 无/auto/短块 的漏句率对比）；③ 说话人跨块锚定命中率。POC 结论要回填 DESIGN.md 的默认参数。之后按 M1（骨架打通链路）→ M5 顺序推进。
