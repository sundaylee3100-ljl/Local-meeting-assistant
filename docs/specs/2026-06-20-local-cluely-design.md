# 本地免费 Cluely 桌面端 — 设计 Spec

> 日期：2026-06-20
> 路线：Fork Pluely（Tauri, GPL-3.0）改造，STT 先云端后本地
> 状态：待用户审查

## 1. 目标与非目标

### 目标
- 在本地构建一个免费的 Cluely 风格桌面 AI 助手，调用用户自己的 LLM/STT API。
- 隐形 overlay（对屏幕共享不可见），实时读屏(OCR/视觉)+听音(STT)→LLM→浮窗提词。
- **纯本地、零服务端**：所有数据（录音、截图、对话、密钥）只存本机，无任何外部回传。

### 非目标（明确排除）
- 不做服务端、不做账号体系、不做付费/授权层。
- 不绕过专业监考软件（Pearson VUE / Proctorio / CoderPad 主动检测）。
- 不做 iOS/移动端。
- 不保证 Linux 上隐身可用（Linux 无统一隐身 API，降级为可见透明窗）。

## 2. 拍板假设（已与用户确认）
| 维度 | 决定 |
|---|---|
| 平台 | Windows 10+ 优先，macOS 次之，Linux 仅开发/构建 |
| API | OpenAI 兼容为主（base_url+key+model），含 Claude/Gemini/Ollama |
| 场景 | 会议/销售实时辅助 + 纪要为主，面试提词为可选高风险模式 |
| 构建 | Fork Pluely 改造 |
| STT | 先云端（Whisper API / Deepgram），后本地（whisper.cpp） |

## 3. 现状基线：Pluely 已有能力（fork 后继承）
经源码核验，Pluely v0.1.9 已实现：
- **隐身**：`WebviewWindowBuilder.content_protected(true)`（[window.rs:165,177](file:///workspace/pluely/src-tauri/src/window.rs#L165)）
- **截屏**：`xcap` 多显示器捕获 + 区域选择 overlay（[capture.rs](file:///workspace/pluely/src-tauri/src/capture.rs)）
- **音频**：`cpal` + 平台分支（Windows `wasapi` / macOS `cidre` / Linux `libpulse`）（[speaker/](file:///workspace/pluely/src-tauri/src/speaker)）
- **密钥**：`tauri-plugin-keychain` 系统钥匙串
- **存储**：`tauri-plugin-sql` SQLite（对话历史、系统提示词）
- **Provider**：`@bany/curl-to-json` 粘 cURL 加任意 provider；预置 openai/claude/grok/gemini/mistral/cohere/groq/perplexity/openrouter/ollama（[ai-providers.constants.ts](file:///workspace/pluely/src/config/ai-providers.constants.ts)）
- **VAD**：`@ricky0123/vad-react` 语音端点检测
- **快捷键/托盘/自启动**：`tauri-plugin-global-shortcut` / `tray-icon` / `tauri-plugin-autostart`
- **流式输出**：`streamdown` + `api::chat_stream_response`

## 4. 改造清单（核心工作）

### 4.1 必须移除（违反"纯本地零服务端"）
| 项 | 位置 | 原因 |
|---|---|---|
| PostHog 遥测 | [lib.rs:10,55-67](file:///workspace/pluely/src-tauri/src/lib.rs#L55), [analytics.ts](file:///workspace/pluely/src/lib/analytics.ts) | 回传使用数据 |
| machine-uid | [lib.rs:68](file:///workspace/pluely/src-tauri/src/lib.rs#L68) | 机器标识，配合遥测/license |
| License 激活系统 | [activate.rs](file:///workspace/pluely/src-tauri/src/activate.rs) 全部, `check_license_status`/`set_license_status`/`LicenseState` | 付费层 |
| Pluely 自有 API | [pluely.api.ts](file:///workspace/pluely/src/lib/functions/pluely.api.ts), `api.rs` 中 `fetch_models`/`fetch_prompts`/`create_system_prompt`/`get_activity` | 连 Pluely 后端 |
| 付费/推广 UI | `GetLicense.tsx`/`PluelyApiSetup.tsx`/`Promote.tsx`/`Usage.tsx` | 商业化组件 |
| Updater 默认源 | `tauri-plugin-updater` | 连 Pluely 更新服务器；改为禁用或自托管 |

### 4.2 必须新增（对抗性审查要求）
| 项 | 实现位置 | 原因 |
|---|---|---|
| macOS 15 隐身降级 | 启动时检测 OS 版本，≥15 弹窗告知"隐身可能露黑块" | Apple 在 Sequoia 改坏 `sharingType=.none` |
| 合规提示 | 首次启用音频捕获时弹窗：录音需对方同意、不绕过监考 | 窃听法 + 反作弊检测产业链 |
| 快捷键默认改键 | 默认非 `Cmd/Ctrl+Enter`（CoderPad 主动检测此键） | CoderPad 检测 |
| CSP 严格 + postMessage origin 校验 | `tauri.conf.json` security.CSP | 防 Jack Cable 式 postMessage 截屏漏洞 |
| 回答标注"AI 生成，需核实" | overlay 每条消息 footer | 防幻觉穿帮 |

### 4.3 保留不动
隐身、截屏、音频、密钥、SQLite、cURL provider、VAD、快捷键、流式输出——这些正是我们要的，不重写。

## 5. 架构（改造后）

```
┌─────────────────────────────────────────────┐
│  UI 层 (React 19 + Tailwind 4 + shadcn)     │
│  - 隐形 overlay 浮窗 / Dashboard / 设置      │
│  - 对话历史 / 系统提示词 / 快捷键管理        │
│  - [新] 合规提示弹窗 / [新] AI 标注          │
└─────────────────────────────────────────────┘
                    ↕ Tauri IPC
┌─────────────────────────────────────────────┐
│  Rust 后端 (src-tauri)                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ capture  │ │ speaker  │ │ api          │ │
│  │ xcap截屏 │ │ cpal音频 │ │ chat_stream  │ │
│  │ 区域选择 │ │ VAD      │ │ transcribe   │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
│  ┌──────────────────────────────────────┐   │
│  │ window: content_protected 隐身        │   │
│  │ shortcuts: 全局快捷键(可重映射)       │   │
│  │ db: SQLite 本地存储                   │   │
│  │ keychain: 密钥系统钥匙串              │   │
│  └──────────────────────────────────────┘   │
│  [移除] posthog / machine-uid / activate    │
│  [新增] os检测降级 / 合规门                 │
└─────────────────────────────────────────────┘
                    ↕ 仅出站 HTTPS
┌─────────────────────────────────────────────┐
│  用户的 API: OpenAI兼容/Claude/Gemini/Ollama │
│  用户的 STT: Whisper API/Deepgram (v1本地)   │
└─────────────────────────────────────────────┘
```

## 6. 数据流（核心闭环）

### 6.1 截图问答模式（MVP）
```
快捷键触发 → start_screen_capture(多显示器)
  → 用户框选区域 → capture_selected_area → base64 PNG
  → chat_stream_response(provider配置, [system, user{文本+image}])
  → 流式 token → overlay 实时渲染
  → 存 SQLite 对话历史
```

### 6.2 实时听音模式（v1）
```
start_system_audio_capture(cpal loopback)
  → VAD 检测说话结束 → 切片 WAV
  → transcribe_audio(STT provider) → 文本
  → chat_stream_response(上下文滑动窗口)
  → 流式 → overlay
```

## 7. 分阶段路线

### MVP（先跑通最小闭环）
1. Fork Pluely，移除 4.1 全部遥测/license/付费层，确保编译通过
2. 在 Linux 沙箱构建跑起来（隐身降级为可见，验证 UI+IPC）
3. 接入用户 OpenAI 兼容 API，跑通：快捷键→截图→LLM→浮窗
4. 密钥走 keychain
5. 加 CSP + postMessage 校验

### v1（实时化）
6. 接入云端 STT（Whisper API），跑通系统音频→STT→LLM 流式
7. VAD 端点检测调优，目标延迟 2-3s
8. macOS 15 降级检测 + 合规门
9. 快捷键默认改键

### v2（生产力 + 本地化）
10. 本地 whisper.cpp 替换云端 STT（可选开关）
11. 本地知识库 RAG（上传 PDF/笔记，SQLite + 向量）
12. 会议纪要自动生成
13. Windows 实机验证隐身（沙箱无法验证）

## 8. 风险与约束（对抗性审查结论）

| 风险 | 约束/缓解 |
|---|---|
| macOS 15+ 隐身露黑块 | 启动检测+告知，不静默失败；推荐 Win |
| Linux 隐身不可靠 | 仅开发用，不承诺隐身 |
| CoderPad 检测 Cmd+Enter | 默认改键，全部快捷键可重映射 |
| 检测产业链（Sherlock/FabricHQ） | 文档明示不绕过专业监考；定位"对方可知情的会议助手" |
| 窃听法 | 默认只捕获系统音频，麦克风手动开；首次录音弹合规提示 |
| 幻觉穿帮 | 每条回答标注"AI 生成，需核实" |
| 服务端泄露（Cluely 83k 翻车） | 纯本地零服务端，无泄露面 |
| postMessage 漏洞（Jack Cable） | CSP 严格 + origin 校验 |
| GPL-3.0 传染 | fork 改造合法，但衍生作品须开源；不可闭源商用 |

## 9. 验证策略
- **Linux 沙箱**：能编译、能跑、UI+IPC+API 调用通；隐身不验证（已知不可靠）
- **Windows 实机**（用户侧）：用 Zoom/Teams 屏幕共享验证 overlay 不可见
- **macOS 实机**（用户侧）：≤14 验证隐身；≥15 验证降级提示
- 单测：cURL 解析、provider 适配、SQLite migration
- 手测：截图→LLM 闭环延迟、音频→STT→LLM 闭环延迟

## 10. 开放问题（实现时再定）
- STT 云端选 Whisper API 还是 Deepgram？（v1 时按用户偏好）
- 本地 whisper.cpp 用哪个模型大小？（tiny/base/small，权衡速度精度）
- RAG 向量用 sqlite-vec 还是独立向量库？（v2）
