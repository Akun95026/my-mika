## ADDED Requirements

### Requirement: 主模型不能看图时，图片文件由视觉模型转写返回
`read_file` 读到图片（png/jpg/jpeg/webp）且主模型不支持图片输入、而设置里配置了视觉模型时，系统 SHALL 把该图片交给视觉模型，将其回答作为文件内容（文本）返回给主模型，任务 SHALL 能据此继续。转写结果 MUST 标明来源（`source: "vision"`），让上层知道这是模型的读图而非文件原文。

#### Scenario: 附件截图问文字
- **WHEN** 用户把截图放进 `/workspace/inbox/` 让任务读，主模型无视觉能力但配置了视觉模型，智能体调 `read_file` 读该图
- **THEN** 工具返回 `ok: true`、`type` 为图片 mime、`content` 为视觉模型的转写（图里的文字与画面说明）、`source: "vision"`，任务不再空手而归

#### Scenario: 视觉模型调用失败
- **WHEN** 视觉模型对图片的调用抛错或返回空回答
- **THEN** `read_file` 返回 `vision_unavailable`（可恢复），不把失败静默成空内容

### Requirement: 主模型能看图或无视觉模型时维持现状
主模型支持图片输入时，`read_file` 读图片 SHALL 维持现状：原图字节直接返回给主模型（不绕视觉模型）。主模型不支持且未配置视觉模型时，SHALL 返回明确的不可读说明（`not_text`），消息里 MUST 指出附件图片可由配置了视觉模型的设置解决。

#### Scenario: 主模型有视觉
- **WHEN** 主模型支持图片输入，`read_file` 读 ≤5MB 的图片
- **THEN** 与现状一致：返回 `path/size/type` 并携带图片，不调用视觉模型

#### Scenario: 既无主模型视觉也无视觉模型
- **WHEN** 主模型不支持图片输入且设置里没有视觉模型
- **THEN** 返回 `not_text` 与「当前模型看不了图」说明，并提示配置视觉模型后附件图片可读

### Requirement: 图片大小上限与格式沿用现状
视觉回退路径 SHALL 沿用主模型直返路径的约束：超过 5MB 的图片 MUST 拒绝（`file_too_large`），支持的图片格式（png/jpg/jpeg/webp）不变；非图片文件不受本变更影响。

#### Scenario: 超大图片
- **WHEN** 主模型无视觉、配置了视觉模型，`read_file` 读一张 6MB 的截图
- **THEN** 返回 `file_too_large`（与主模型直返路径同一上限），不调用视觉模型
