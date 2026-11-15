# design：#624 附件图片回退视觉模型

## Context

- `read_file` 的图片分支在 `SandboxTools.readOpened`（core/…/cap/sandbox/SandboxTools.kt:275-279）：主模型有视觉（`visionEnabled`）→ 原图字节随结果返回；无视觉 → `notText`「cannot view images」，11ms 即回，与 #624 报告的失败时长吻合。
- `SandboxTools` 由 `SandboxCapability` 构造（只拿到 `visionEnabled` 布尔），由 `assembly/TaskTools.kt:157` 装配——那里 `vision: VisionService?`（看图模型，未配置为 null）本来就在作用域内，只是没传进沙箱。
- `VisionService.ask(system, text, images: List<ByteArray>)`（core/…/vision/VisionService.kt:19）接受任意图片字节，编码（JPEG/缩放）由 `ModelImage.part` 统一处理；现有调用方（`Locator`、核查器）都走它。
- 看图模型相关的既有工具 `describe`/`find_visually`（VisionTools.kt）只针对**当前屏幕**（截图来自 `ScreenPort`），与工作区文件无关。
- 测试模式现成：`ToolDispatcherSandboxTest` 用 `toolDispatcher(FakeScreen(), vision, sandbox = sb)` 装配，`says()` 假模型喂 `VisionService`。

## Goals / Non-Goals

**Goals:**
- 主模型无视觉、且配置了视觉模型时，`read_file` 读图片 → 视觉模型转写 → 文本返回，任务可继续（#624 的期望行为）。
- 回退路径与主模型直返路径共用同一套约束（5MB 上限、格式识别），不新开政策。
- 全部改动在 `:core`，假模型可单测；真机验证走附件上传 + 无视觉主模型配置。

**Non-Goals:**
- 不做「带具体问题读图」的新工具（如 `describe_file(path, question)`）——issue 要的是别空手而归，通用转写（文字 + 一句话画面）已覆盖「图里有什么字」一类诉求；问题式读图留给维护者拍板。
- 不动 `describe`/`find_visually`（管屏幕）、不动主模型图片直返路径、不动附件复制进 inbox 的流程（`Attachments.kt`）。
- 不涉受保护页面策略：文件是用户亲手上传的，本就不在防截屏管辖内。

## Decisions

1. **回退放在 `SandboxTools.readOpened`，`VisionService?` 经构造函数注入**（而非在 `SandboxGroup` 的 execute 包装层拦截错误结果）。
   - 文件读取的图片分支逻辑聚在一处；包装层拦截要靠错误码反查、还得重新打开文件，更绕。
   - 装配层 `TaskTools` 已有 `vision`，经 `SandboxCapability` 传一层即可；`SandboxTools` 构造函数加 `vision: VisionService? = null`（默认 null），现有测试构造点不破。
   - 备选（否决）：新建独立工具 `read_image(path, question)`——多一个工具面、模型多一次选择负担，且 `read_file` 是模型读附件的自然入口，失败处就地兜底最贴合报告场景。

2. **`readFile` 改为 suspend**。`vision.ask` 是 suspend；调用点 `SandboxGroup.execute` 本就是 suspend + `withContext(Dispatchers.IO)`，`readOpened`/`readFile` 内部加挂起点不动调用方签名。视觉调用放在 `Dispatchers.IO` 块内（与文件读同一调度域）。

3. **新角色提示词 `roles/read_image.md`，不复用 `roles/describe.md`**。`describe` 是屏幕坐标系契约（0-999 坐标、可交互元素清单），对任意图片不适用。新角色风格对齐：只报图上有什么（逐字转写全部可见文字 + 一句话画面），不支招、不猜图外信息，回答保持简短。

4. **结果形状**：`{ok: true, path, size, type, content: <转写>, source: "vision"}`。`source` 字段让上层（时间线/证据）能区分「模型读图」与「文件原文」；失败回 `fail("vision_unavailable", …, recoverable = true)`，与 `Describe` 工具的错误码一致。无视觉模型时的 `not_text` 文案补一句指路（配置视觉模型后附件图片可读）。

## Risks / Trade-offs

- [视觉模型转写有额外延迟与 token 成本] → 只在主模型无视觉时才触发（有视觉的用户零变化）；单次调用、无重试。
- [转写质量取决于用户配的视觉模型] → 失败/空回答如实回 `vision_unavailable`，不静默编造；结果带 `source: "vision"` 供上层标注。
- [`readFile` 变 suspend 波及调用方] → 全仓只有 `SandboxGroup` 一个调用点（测试直接调也已在 runTest 内）；编译期即可确认无遗漏。
- [看图模型收到用户任意附件] → 与现有 `view_user_screen`（用户主动发的屏幕）同级别：用户亲手上传的内容，本就是任务输入；不涉及受保护页面策略。

## Migration Plan

纯增量行为（原路径两个条件分支不变），无数据/设置迁移；回滚 = revert 单个提交。

## Open Questions

（无——实现中若发现 `SandboxCapability` 在 `TaskTools` 之外还有构造点，按同参数补齐即可。）
