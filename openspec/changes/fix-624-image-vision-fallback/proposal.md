## Why

#624（荣耀 MagicOS 7.1 / Mika 0.1.42）：用户在对话里以附件上传截图、问「图里显示了什么字」，任务空手而归——`read_file` 读 `/workspace/inbox/Screenshot_….jpg` 11ms 即失败（主模型不支持图片输入时，`SandboxTools.readOpened` 对图片一律回 `not_text`「cannot view images」），全程 3 次模型调用都是主模型，**设置里配好的视觉模型一次都没被叫到**。附件是用户亲手交进来的文件，路径上却没有任何一条能把它送到能看图的模型面前：`describe`/`find_visually` 只看当前屏幕，双击胶囊的「此刻的屏幕」走无障碍树/OCR 也不经过视觉模型——纯视觉内容（截图里的图、拍照的图）在「主模型无视觉」的配置下成了死角。

## What Changes

- `read_file` 读到图片（png/jpg/jpeg/webp）且主模型不支持图片输入时，若设置里配了视觉模型：把图片字节直接交给视觉模型转写（读出图里的全部文字 + 一句话说明画面），以文本内容返回给主模型，任务可以继续往下走。
- 主模型本就能看图时行为不变（图片原样返回）；没配视觉模型时行为不变（仍回「cannot view images」，文案补充指路）。
- 复用现有约束：>5MB 的图片照样拒（`file_too_large`）；视觉模型调用失败回 `vision_unavailable`（可恢复），不吞任务。
- 新增视觉角色提示词（`roles/read_image.md`）：只报图上有什么（逐字转写文字 + 一句话画面），不支招，风格对齐现有 `describe` 角色。
- 单测：`:core` 用假模型覆盖三分支（主模型无视觉+无视觉模型、主模型无视觉+有视觉模型回退、主模型有视觉原图直返）与失败/超限分支。

## Capabilities

### New Capabilities
- `sandbox-image-read`: 沙箱工作区图片文件的读取契约——主模型能看图时原图返回；不能看图但配了视觉模型时由视觉模型转写为文本返回；两者都不满足时明确告知并指路。附件图片（`/workspace/inbox/`）不再因主模型无视觉而无法处理。

### Modified Capabilities

（无——`openspec/specs/` 尚无主规格，本变更首次为该能力立规格。）

## Impact

- `core/src/main/kotlin/com/wangzan101/mika/core/cap/sandbox/SandboxTools.kt`（`readOpened` 图片分支加视觉回退；构造函数注入 `VisionService?`）
- `core/src/main/kotlin/com/wangzan101/mika/core/cap/sandbox/SandboxCapability.kt`、`core/src/main/kotlin/com/wangzan101/mika/core/assembly/TaskTools.kt`（把装配层已有的 `vision` 传进沙箱能力）
- `core/src/main/resources/prompts/roles/read_image.md`（新增）、`prompts/tools/read_file.md`（补一句回退说明）
- `core/src/test/kotlin/com/wangzan101/mika/core/tool/ToolDispatcherSandboxTest.kt`（新增用例，沿用 `says()` 假视觉模型模式）
- 不动 `describe`/`find_visually`（它们管屏幕，不管文件）；不动主模型图片直返路径与 5MB 上限。
