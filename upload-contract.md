# Media Go 统一上传契约

交付确认与撤回采用 [AUDIO-24](https://app.plane.so/mian-personal-workspace/browse/AUDIO-24/) 和[统一录音结束确认与暂存撤回规范](https://app.plane.so/mian-personal-workspace/projects/3ae98cc9-2167-4b22-8fa1-660cf4d8ab8a/pages/cc64f275-3c2a-43af-9204-5b8b483a4fc1)。其余契约按 [ORB-935 需求决定 §1–§6](https://paperclip.tail0bda2c.ts.net/ORB/issues/ORB-935#document-decisions) 修订；相关决定见 [ADR 0001–0003](https://github.com/QinMian5/media-go/blob/main/docs/adr/)。本文规定待实施的行为，既有验收记录不能作为本次变更已实现的证据。架构与实施拆分见 [ORB-936 方案](https://paperclip.tail0bda2c.ts.net/ORB/issues/ORB-936#document-plan)。

## 1. 范围与流程

- 媒体条目由采集按钮启动，沿用开始采集时的按钮快照：交付模式、片段时长和标注字段；同时固定设备的采集时区。之后修改或删除按钮不改变已有条目。
- 音频格式固定为 PCM16/WAV、48 kHz、单声道。
- 持续上传：按按钮快照中的片段时长封存并上传音频片段；片段时长为 1–600 整数秒。正常停止时封存不足该时长的非空尾段。用户确认标注后提交录音清单及结束信息。
- 整段上传：停止录音并确认标注后，以 `file` 操作上传完整 WAV；确认前不向接收端发送该条目的任何内容。录音不设业务时长上限。
- 确认前可修改标注；确认时按按钮快照检查必填项和取值类型。确认后固定最终请求，不再修改标注，重试保持提交内容不变。
- 两种模式在点击结束后都提供上传、重录和不上传／放弃。只有上传校验必填标注；放弃删除本地整次录音，并撤回目标暂存内容。重录先放弃旧录音，再以新媒体身份沿用按钮快照和已有标注采集。
- 整段上传失败可重传完整文件；持续上传按片段补传，不要求字节级断点续传。
- 接收端保存片段、录音清单、最终标注和采集情况后即可确认交付完成，无须等待合并为单一文件或后续处理。

## 2. 同一 URL，五种操作

所有操作 POST 到用户配置的同一接收 URL，使用 `multipart/form-data`：

| metadata.operation | 含义 | 表单部分 |
| --- | --- | --- |
| `file` | 整段上传的完整 WAV | 一个 `file` 和一个 `metadata` |
| `segment` | 持续上传的一个音频片段 | 一个 `file` 和一个 `metadata` |
| `finalize` | 提交录音清单、采集情况及最终标注 | 只有一个 `metadata` |
| `test` | 设置页测试连接，不保存内容 | 只有一个 `metadata` |
| `delete` | 按媒体身份撤回交付 | 只有一个 `metadata` |

`metadata` 是 UTF-8 JSON 对象的序列化字符串，部分 Content-Type 为 `application/json`。`file` 部分携带文件名，Content-Type 为 `audio/wav`；接收端校验非空 WAV 的格式和有效音频帧。文件名不充当媒体身份。

可配置完整的 Authorization 请求头；未配置则省略。凭据不进入 metadata、媒体文件或队列中的明文内容。

使用标准 multipart 表单，不另建持续连接。文件名按 UTF-8 百分号编码写入 `filename`，接收端对应解码。[multipart/form-data 规范 §4](https://datatracker.ietf.org/doc/html/rfc7578#section-4)

## 3. 公共身份与标注

除 `test`、`delete` 外，每个 metadata 都含以下字段；`delete` 只包含 operation 和 media_id，见第 12 节：

| 字段 | 约定 |
| --- | --- |
| `operation` | `file`、`segment` 或 `finalize` |
| `media_id` | 客户端生成并持久保存的 UUID；持续上传的全部片段和结束信息共用此 ID |
| `capture_timezone` | 设备开始采集时的 IANA 时区标识，例如 `Asia/Shanghai`；同一条目的 file、segment、finalize 与重试沿用该持久快照 |
| `annotations` | JSON 对象，内容由按钮的标注字段决定；没有标注时为 `{}` |

`media_id` 同时标识该媒体条目的录音会话，无须另一套会话身份。新采集生成新 ID，重试不换 ID。同一 ID 的交付模式固定，不得混用 `file` 与 `segment`/`finalize`。

`capture_timezone` 由客户端自动提供，不是用户的标注字段。设备更换时区、条目恢复和队列重试
不重新读取当前时区；不发送设备的实际录音开始时间。该字段用于之后的新采集，
已有条目及已固定请求不回填或改写。

接收端校验提供的 `capture_timezone` 为可解析的 IANA 时区字符串；`null`、非字符串、
空值或未知时区返回 HTTP 422 / `invalid_metadata`。同一 `media_id` 首次成功保存时，
接收端持久固定该时区，之后的片段、finalize 和重试必须使用相同值。
旧条目可全程省略该字段；同一条目不能混用省略与提供时区，或改成另一个时区，
否则返回 HTTP 409 / `idempotency_conflict`。失败的请求不固定时区。

用户只定义 `annotations` 内的内容，不改写协议字段的名称、位置或结构。标注字段的键用点号分层，例如 `speaker.name` 生成 `{"speaker":{"name":"张三"}}`。每层为一个或多个字母、数字或 `_`；同一按钮内不能重名，也不能同时存在 `speaker` 和 `speaker.name` 这种父子键。

文本和单选以 JSON 字符串发送，数字以 JSON 数字发送，开关总是发送 JSON 布尔值 `true` 或 `false`。只有空白的文本视为未填；非开关可选字段未填时省略该键，空的父对象也不生成。数字可为负数或小数，不设业务取值范围。按钮字段的名称、顺序、默认值和必填规则由客户端校验，不要求接收端持有按钮定义或重复执行表单校验。

`segment` 固定发送 `annotations: {}`。最终标注只在 `finalize` 提交；旧片段重试不得覆盖最终标注。整段上传的最终标注随 `file` 提交。接收端把 `annotations` 作为 JSON 对象保存，不赋予某个键名额外业务含义。

## 4. 整段上传示例

以下 JSON 均为 metadata 部分，UUID 为示例。不要求文件摘要。

~~~json
{
  "operation": "file",
  "media_id": "00000000-0000-4000-8000-000000000001",
  "capture_timezone": "Asia/Shanghai",
  "annotations": {
    "kind": "recording",
    "speaker": {"name": "张三"},
    "offset": -1.5,
    "consent": false
  }
}
~~~

`file` 部分附带对应非空 WAV。接收端完整接收请求与文件、核对 metadata 和 WAV，并持久保存文件及 metadata 后才返回保存回执。

## 5. 片段示例与时间线

~~~json
{
  "operation": "segment",
  "media_id": "00000000-0000-4000-8000-000000000002",
  "capture_timezone": "Asia/Shanghai",
  "annotations": {},
  "segment": {
    "index": 0,
    "start_frame": 0,
    "frame_count": 28800000
  }
}
~~~

- `index` 是从 0 开始递增且不复用的整数片段序号。客户端优先补传最早未确认的片段；接收端按序号与媒体时间识别内容，不依赖 HTTP 到达顺序。
- `start_frame` 是非负整数，`frame_count` 是正整数，均基于 48 kHz 音频时间线。协议上限为 600 秒，即 28,800,000 帧。客户端每段还不能超过该条目快照的片段时长乘以 48,000；非空尾段可以更短。布尔值不是整数帧数。
- 快照的片段时长只控制客户端切分，不新增上传字段。接收端按协议上限和清单核对，无须知道按钮配置。
- 接收端使用 WAV 读取能力核对格式和实际有效帧数，不能把文件总字节数当成帧数。
- 确认前片段只在接收端暂存，不启动转写、说话人处理或声纹登记。点击结束不提交交付确认；用户确认后提交 `finalize`，且声明音频收齐后才允许后续处理。

## 6. 结束信息与清单

以下示例表示完整的 602 秒录音，片段时长设为 600 秒，尾段为 2 秒。`finalize` 不携带媒体文件。

~~~json
{
  "operation": "finalize",
  "media_id": "00000000-0000-4000-8000-000000000002",
  "capture_timezone": "Asia/Shanghai",
  "annotations": {"meeting": {"topic": "今天的组会"}},
  "capture": {
    "end_reason": "stopped",
    "completeness": "complete"
  },
  "segments": [
    {"index": 0, "start_frame": 0, "frame_count": 28800000},
    {"index": 1, "start_frame": 28800000, "frame_count": 96000}
  ]
}
~~~

清单至少含一个有效片段，按 `index` 排序且不重复，每项须匹配已接收片段的序号和时间范围。接收端核对所声明片段全部存在，且清单没有遗漏该录音已接收的片段；不能缩减清单来掩盖待传或丢失的片段。

正常完整录音从 0 帧开始、首尾相接且不重叠。存在缺口时不能声明 `complete`；可确定的范围保持原位置，不挤掉缺失区间来伪造连续音频。

保存清单、最终标注、采集情况并完成核对后，接收端才返回 `finalize` 保存回执。此后已保存请求仍可按第 9 节重试，但不能覆盖内容或加入新片段。未确认标注时可继续上传片段，不能发送 `finalize`。

## 7. 异常录音

`capture.end_reason`：`stopped` 为正常结束，`interrupted` 为采集意外中断。

`capture.completeness`：

- `complete`：没有已知或未决的采集缺失，完整范围可核对。
- `incomplete`：已知有音频缺失。
- `unknown`：无法确定是否缺失或缺失了多少音频。

崩溃恢复不能仅因现有 WAV 可读就声明 `complete`。无法确定尾段保全情况时使用 `unknown`；知道尾段丢失则使用 `incomplete`。缺失长度未知时不编造时长。

持续上传的可恢复片段先自动补传；用户确认标注后再提交 `finalize`。清单及异常信息也被保存后，条目可成为「已上传」，同时保留采集中断、缺失或完整性未知的提示。完全没有可恢复音频时不能成功归档。

整段上传的可恢复 WAV 仍需用户确认后发送 `file`；本地保留异常提示，`file` 不新增 `capture` 字段。此处异常信息描述采集损失；接收端缺少清单声明的文件仍是传输未完成，不能用 `incomplete`/`unknown` 绕过补传。

## 8. 保存回执与连接测试

`file`、`segment`、`finalize` 保存成功均返回 HTTP 200、Content-Type 为 `application/json`，响应体只有：

~~~json
{"status": "received"}
~~~

客户端在 HTTP 200、完整且有效的 JSON 对象响应中，只核对 `status` 是否严格等于字符串 `received`。不检查其他 JSON 字段；收到额外字段也不把它们当作成功条件。`processing`、缺少 `status`、解析失败、截断或仅有 HTTP 成功，均不能标为已上传。

- `file` 回执表示整个 WAV 及最终标注已保存。
- `segment` 回执仅表示该片段及其 metadata 已保存，不代表整个媒体条目已上传。
- `finalize` 回执表示清单声明的音频、最终标注和采集情况已保存并核对，整次录音交付完成。

客户端根据发起请求时已固定的本地任务记录，将回执归到相应条目、操作、片段和接收目标。回执丢失属于结果未知，保留数据并重试。接收端返回错误的保存确认时客户端无法核实真实保存结果，这是 [ADR 0003](https://github.com/QinMian5/media-go/blob/main/docs/adr/0003-status-only-receipts.md) 已接受的限制。

### 连接测试

向同一 URL 发送仅含 `metadata` 的 multipart 请求，携带当前输入的 Authorization（未填则省略）：

~~~json
{"operation": "test"}
~~~

`test` 不带其他 metadata 字段，不保存媒体或去重记录。认证通过后返回 HTTP 200、Content-Type 为 `application/json`，响应体只有：

~~~json
{"status": "ok"}
~~~

客户端只核对 JSON 对象中的 `status` 是否严格等于字符串 `ok`。测试结果仅作设置页提示，不改变媒体条目的上传状态。

| 结果 | 客户端提示 |
| --- | --- |
| 网络、超时、DNS、TLS 错误 | 无法连接 |
| HTTP 401/403 | 认证失败 |
| HTTP 404/405/422，或 HTTP 200 但回执无效 | 地址可达，但不是 Media Go 接收端 |
| 其他 HTTP 状态 | 显示 HTTP 状态码 |

HTTP 和 Content-Type 检查属于传输封装检查；保存回执与测试连接的响应体都仅以 `status` 判定。

## 9. 幂等与冲突

不增加单独的 Idempotency-Key 或请求 UUID。每个接收端内的逻辑操作身份为：

| 操作 | 唯一身份 |
| --- | --- |
| `file` | media_id + file |
| `segment` | media_id + segment + segment.index |
| `finalize` | media_id + finalize |

客户端须持久保存并固定提交的 metadata 和媒体文件；重试不能重新编码、改写文件或复用身份提交另一份内容。metadata 对象键顺序、JSON 空白和 multipart boundary 不改变请求含义；文件名和文件部分的媒体类型也随首次提交固定。

接收端以同一操作身份下首次成功保存的文件和 metadata 为准，持久保存操作身份、metadata 与媒体关联，不能只用内存缓存去重。重复请求的 metadata 语义、文件名和媒体类型与已有记录一致时，仍返回 `{"status":"received"}`，不创建重复条目或覆盖文件。并发重复请求同样只产生一份保存结果。媒体保留期间保留其已确认操作的去重信息。响应体变简单不改变这些请求侧规则。

不要求 SHA-256、其他文件摘要或逐字节比较。接收端不承诺识别身份及 metadata 相同但文件字节不同的错误重试，这时仍保留首次成功保存的文件。客户端负责保持文件内容不变。

相同操作身份但采集时区、标注、时间范围、清单、文件名或媒体类型不同，返回 HTTP 409 / `idempotency_conflict`。同一 `media_id` 的交付模式或采集时区变化也属于冲突，包括不同操作之间的变化。

`finalize` 成功后，符合上述规则的旧请求重试仍成功；清单外的新片段返回 HTTP 409 / `recording_finalized`。未成功的 `finalize` 不得提前锁定录音。旧片段请求的空标注不覆盖 `finalize` 的最终标注。

## 10. 缺段与失败处理

`finalize` 发现缺段返回 HTTP 409，例如：

~~~json
{
  "status": "error",
  "error": {"code": "missing_segments", "segment_indices": [1]}
}
~~~

`segment_indices` 必须是非空、无重复、非负整数数组，不接受布尔值。客户端先核对所有索引均属于已固定清单且本地文件可用，只补传这些片段，再重试原 `finalize`。无效或清单外索引按协议错误处理，不改变确认记录；本地文件无法恢复时保留条目并提示需处理，不缩减清单，不重发所有已确认片段。

接收端已有额外片段，或片段序号、时间范围与清单不一致时返回 HTTP 409 / `manifest_conflict`，不要求按摘要或字节比较文件。

| 情况 | 行为 |
| --- | --- |
| 网络断开、超时、HTTP 408/429、暂时性 5xx | 保留数据、自动退避重试，遵守有效 Retry-After |
| 409 / missing_segments | 先补声明的缺段，再重试结束请求 |
| 409 / idempotency_conflict、manifest_conflict、recording_finalized | 保留数据并暂停，提示协议或数据冲突 |
| 401/403、明确无效地址/404/405、413、415、无效 metadata | 暂停并提示需处理，修正后重试，不换身份覆盖内容 |
| HTTP 200 但回执无效 | 保留数据，提示回执错误，不能标成功 |

可解析的应用错误使用 `status: error` 与 `error.code`；无效 metadata 或操作使用 HTTP 422。代理或网络层不一定返回 JSON，客户端仍按传输结果保留数据。错误响应仍按错误码处理，成功回执的简化不删除错误信息。

错误码为本契约的应用约定。[HTTP 冲突](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.10) 与 [Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.3) 遵循 HTTP 语义。

## 11. 队列、目标与后台边界

- 待传文件、媒体身份、按钮快照、采集时区、标注草稿、已提交 metadata 和确认记录须在重开后恢复。整个条目取得最终保存回执前，不自动清理仍可能用于补传的音频。
- 未配置地址时保留本地内容，等待配置。不能将不同接收端的片段确认记录拼成成功归档。
- 条目首次取得保存回执时绑定发出该请求的接收目标，此后一直发往该目标。尚无回执的条目跟随当前设置地址。不提供改投入口；原目标永久不可用的条目停在「需处理」。后台任务保留原请求目标和身份，响应不能按当前页面或当前设置重新归属。
- 凭据可按接收目标更新，不参与内容身份；更新凭据不改变媒体或请求内容。不随 HTTP 重定向改投。
- 活跃录音按快照片段时长封存片段并尽快发送。停止后后台调度可能延迟，不承诺秒级完成或强制退出后继续采集。
- 上传结束后的本地音频保留、容量阈值与逐条删除规则沿用[本地保留与清理决议](https://paperclip.tail0bda2c.ts.net/ORB/issues/ORB-17)。不自动删除未上传内容腾空间；删除媒体条目不撤回接收端已保存的内容。

## 12. 交付撤回

向实际发送过音频的同一完整接收 URL 发送 multipart 请求，仅含一个 metadata：

~~~json
{"operation":"delete","media_id":"00000000-0000-4000-8000-000000000002"}
~~~

无需音频、annotations 或 capture_timezone；授权沿用该目标的原规则。
先持久终止身份，再尝试清理。未知身份同样有效，不以 404 代表完成，也不创建占位录音。
重复撤回保持同一终止身份并重试清理；撤回后迟到的 file、segment、finalize 在提交前被拒绝，
返回 HTTP 410、error.code=recording_deleted，不能用 received 当作成功。
终止标记在重启后继续有效，不因文件清理或普通去重记录移除而消失。

| HTTP | JSON 回执 | 事实 |
| --- | --- | --- |
| 200 | `{"status":"deleted"}` | 身份已终止，受管音频和已有在途暂存排空 |
| 202 | `{"status":"deletion_pending"}` | 身份已终止，物理清理等待上传占用、读取或执行 |
| 202 | `{"status":"deletion_failed"}` | 身份仍终止，清理报告故障，修复后可以重试 |

客户端按持久任务关联媒体和目标；逻辑接受不能记为物理清理完成。
对 202 保留待办，以同一命令查询／重试；认证、地址、未知操作和无效回执显示需处理。
不新增版本号或身份回显；固定 status 无法独立核验接收端真实执行的限制沿用 ADR 0003。

放弃决定、目标集合与清理待办独立于媒体条目持久保存，不保留原音频或明文凭据。
发送前记录实际完整目标 URL，回执丢失或首次回执前改地址仍须撤回所有曾发送目标。
放弃后终止旧身份的新交付和迟到回调；本地删除不等待网络恢复，目标清理独立继续。
普通已确认条目的列表删除仍沿用原本地删除语义。

App 重开、回到前台及现有交付推进机会会恢复本地删除并推进撤回待办。临时故障和
`deletion_pending` 按持久退避时间重试；认证、不支持撤回和 `deletion_failed` 暂停，
保存修正后的接收端设置或点击 Retry cleanup 后重试。请求在途时的修正同样生效。
凭据始终按原完整目标 URL 从 Keychain 读取，更换当前接收地址不会改投旧撤回任务。
此恢复机制不保证 App 挂起或强制退出后的常驻后台清理。

媒体页按实际目标显示 Cleanup pending、Needs attention、Cleaned；只有全部目标完成且
本地移除成功才算整次放弃完成。已清理的最小记录保留供查看，迟到回执不能将其降回待办。

## 13. 实施验收清单

以下需由实施任务验证，本次文档修订不代表已通过运行验证：

1. 持续上传分别使用 1、5、600 秒片段时长，边界和非空尾段帧数准确；600 秒片段可接受，超过 28,800,000 帧拒收。仅收到片段回执时整条仍未完成。
2. 两种交付模式都能保存；整段上传确认前没有请求，确认后仅提交完整 WAV。非 WAV、非 48 kHz 单声道 PCM16 或空音频拒收。
3. 点号嵌套、可选空值省略、负数、小数、布尔 `false` 和单选文字正确保存；接收端不增加按钮字段的业务校验。持续上传片段标注为空，最终标注随清单保存。
4. 接收成功但回执丢失时，原请求重试不产生重复内容；相同身份的不同 metadata、文件名或媒体类型返回冲突。
5. 断网继续采集，恢复后优先补传历史；停止或重开后恢复队列。修改或删除按钮不改变条目快照和固定请求。
6. `missing_segments` 的 `segment_indices` 触发指定片段补传；无效索引不影响确认记录，补齐后原 `finalize` 成功；清单冲突不被覆盖。
7. 崩溃后按快照片段时长恢复尾段，保留中断和完整性提示，不自动开启麦克风；无有效音频不能成功归档。
8. HTTP 200 加 `{"status":"received"}` 可确认对应操作；错误 status、截断回执、错误凭据不能成功。旧任务回调、目标切换不能串用本地确认记录。
9. 接收端尚未合并音频或执行后续处理时，已保存声明内容仍可返回保存回执；`processing` 不算保存回执。
10. `test` 的请求为 `{"operation":"test"}`，HTTP 200 加 `{"status":"ok"}` 即连接成功，不产生媒体或去重记录。认证错误、错误地址、无效回执分别显示规定结果。
11. 仅在最终回执持久化后进入「已上传」和本地音频清理流程；接收端错误返回固定成功状态的风险按 ADR 0003 接受，不声称客户端能识别这种误报。

12. 未知身份与已有暂存内容均可幂等撤回；首次上传乱序、回执丢失、接收端重启不恢复旧身份。占用或清理故障返回 202，物理排空才返回 deleted；确认前无处理，确认且收齐后正常处理。
