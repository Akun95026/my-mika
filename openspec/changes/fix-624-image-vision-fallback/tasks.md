## 1. 提示词与工具说明

- [x] 1.1 新增 `core/src/main/resources/prompts/roles/read_image.md`：视觉模型读任意图片的角色契约（逐字转写全部可见文字 + 一句话画面说明；不支招、不猜图外信息；简短）
- [x] 1.2 更新 `core/src/main/resources/prompts/tools/read_file.md`：补「主模型看不了图时，配了视觉模型则返回其转写」

## 2. 核心实现

- [x] 2.1 `SandboxTools`：构造函数加 `vision: VisionService? = null`；`readFile`/`readOpened` 改 suspend；图片分支在 `!visionEnabled` 且 `vision != null` 时读字节（≤MAX_IMAGE）→ `vision.ask(read_image 角色, …)` → 返回 `{ok, path, size, type, content, source: "vision"}`；`vision == null` 时维持 `not_text`（文案补指路）；视觉调用失败回 `vision_unavailable`（recoverable）
- [x] 2.2 `SandboxCapability` 与 `assembly/TaskTools.kt`：把装配层的 `vision` 传到 `SandboxTools`；全仓扫其它构造点按需补齐
- [x] 2.3 确认 `SandboxGroup` 调用点无需改（execute 已 suspend）；`./gradlew :core:test` 编译通过

## 3. 单元测试

- [x] 3.1 `ToolDispatcherSandboxTest`：主模型无视觉 + 有视觉模型 → `read_file` 图片返回 `source: "vision"` 的 `content`（`says()` 假模型断言转写文本与传入的图片字节）
- [x] 3.2 同文件：视觉模型抛错 → `vision_unavailable` 且 recoverable；>5MB → `file_too_large` 且未调视觉模型
- [x] 3.3 回归：主模型有视觉 → 原图直返不变；无视觉且无视觉模型 → `not_text` 指路文案
- [x] 3.4 `./gradlew :core:test :core:cliTest :app:testDebugUnitTest` 全绿

## 4. 真机验证与取证

- [x] 4.1 构建装机（debug/perf），配置无视觉主模型 + 视觉模型，聊天附件上传截图问「图里有什么字」，任务应返回图中文字（对比修复前空手而归）
- [x] 4.2 截图/日志取证存 `openspec/changes/fix-624-image-vision-fallback/evidence/`（敏感内容不入库），写 evidence.md 时间线

## 5. 收尾

- [x] 5.1 `openspec` 产物随分支提交；PR 正文关联 #624，标题「现状 → 改动」句式，首屏给实测证据
- [ ] 5.2 push 分支到 fork、建 PR（base: main，head: TroyLiu2023:fix/624-image-vision-fallback）
