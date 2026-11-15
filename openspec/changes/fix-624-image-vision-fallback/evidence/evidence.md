# #624 附件图片回退视觉模型——验证取证

## 方法

真机（华为 Mate X2，TET-AN50，HarmonyOS 4.2 / API 31）+ 本机 mock OpenAI 兼容服务复现报告人的配置形态：

- 主模型 `mock-main`：无视觉（`vision:false`），支持工具调用——对应报告人的 raw-hy4（不支持图片输入）。
- 视觉模型 `mock-vision`：独立配置在「视觉」用途——对应报告人的 Gemma-4-31B-IT。
- mock 服务监听电脑 127.0.0.1:8900，`adb reverse` 转发;App 的 providers.json 临时替换为指向该服务（原配置已备份，验证后还原）。
- 附件经「电脑连接」`/api/v1/uploads` + `POST /api/v1/tasks` 注入（`shot.png`，2108 字节，白底黑字 "MIKA-624 TEST PICTURE"）。

## 端到端链路（server-mock-models.log，任务 5ebbbd5b）

1. `[main_tool_call]` 主模型（无视觉）对附件行里的路径发起 `read_file /workspace/inbox/shot.png`。
2. `[vision_call]` 视觉模型被调用：`has_image: true`，system 提示词正是新角色 `read_image`（"You read one image file (a screenshot, photo, or picture)…"）。
3. `[main_final]` read_file 工具结果：
   `{"ok":true,"path":"/workspace/inbox/shot.png","size":2108,"type":"image/png","content":"[文字] MIKA-624-回退成功\n[画面] 一张白底黑字的测试图","source":"vision"}`
4. 任务最终答案把转写内容带回给用户——修复前这条路是 `not_text`「cannot view images」直接失败（issue 里 11ms 即回、任务空手而归）。

截图 `02-after-vision-fallback-chat.png`：任务完成后的聊天界面（补充;文字判读以服务端日志为准）。

## 单元测试

`./gradlew :core:test :core:cliTest :app:testDebugUnitTest` 全绿。新增
`ToolDispatcherSandboxTest.imageReadFallsBackToVisionModelWhenMainModelCannotSee`：
无视觉模型指路 / 回退成功（断言图片字节原样到达视觉模型、system 为 read_image 角色）/ 视觉失败 `vision_unavailable` /
>5MB `file_too_large` 不调视觉模型;提示词表面快照按 `-PupdateSnapshots` 重生成。

## 过程记录

- mock 侧两次脚本 bug（路径正则字符类 `]` 未转义、命中系统提示词里的示例路径）曾让主模型拿错路径,与 App 无关,修正后一次通过。
- 验证后已还原 providers.json/provider-keys.json（还原后核对:服务「智谱 Coding Plan」、主模型 glm-5.3-flash）,关闭电脑连接,移除 adb reverse/forward,停掉 mock 服务。
