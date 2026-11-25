# iDock CLI 工作流

以下示例中的 `RECORDING_ID`、`FLASH_ID`、`JOB_ID`、`CURSOR` 和 `NEXT_OFFSET` 都要替换为当前实际返回值，不直接把占位符传给 CLI。ID 保持字符串。默认 JSON，文件格式用 `--file-format`，终端格式用 `--format json|text`。

## 录音、闪记和 RAG

```text
idock recording list --time 7d --limit 5
idock recording list --time last-week --keyword "产品会议" --source offline --limit 5
idock recording get RECORDING_ID --section summary --length 4000
idock recording get RECORDING_ID --section transcription --offset NEXT_OFFSET --length 4000
idock search "上次产品会议确定了哪些行动项" --limit 5 --length 2000
idock search "产品会议行动项" --time 7d --limit 5
idock flash note list --time today --limit 5
idock flash todo list --time 7d --limit 5
idock flash bill list --limit 5
idock flash idea list --limit 5
idock flash todo get FLASH_ID --length 2000
```

- `note` 随手记，`todo` 待办，`bill` 账目，`idea` 灵感；每个类型均有 `list/get`，不能跨类型混用 ID。
- 相对时间：`today` 当天，`7d` 含当天的七个自然日，`last-week` 上一自然周。使用用户时区；用户指定时可加全局 `--timezone Asia/Shanghai` 等 IANA 时区。
- 自定义范围用 `--from`、`--to` ISO 时间（含起点，不含终点），与 `--time` 互斥。例如 `--from 2026-10-01T00:00:00+08:00 --to 2026-11-01T00:00:00+08:00`。根据当前日期和用户意图生成，不能照抄固定示例日期。
- RAG 的时间范围按录制时间过滤；问题中的“上周”等词不会自动限制候选，需显式传入时间参数。用 `metadata.recording_id/title/recorded_at` 引用来源，不能把 `source_id/doc_id/point_id` 当作录音编号。`low_confidence=true` 或空片段时说明未找到相关内容，不用召回分数猜答案。
- 默认 JSON 保留上述来源字段供后续调用，`--format text` 只展示来源标题、录制时间与正文，无匹配时提示“未找到相关内容。”。`score_type=rerank_score` 表示重排分数，不代表答案正确率；时间窗口与来源映射由服务端校验，CLI 不按文件名猜测。
- 单数录音和闪记列表返回 `data.items`、`data.next_cursor`，部分响应同时在 `meta.next_cursor` 提供游标。下一页复用原过滤条件并加 `--cursor CURSOR`。游标为空、重复或不前进时停止；这些入口遇到失效游标最多重查一次首页并给 `cursor_reset=true`，此时重新确认范围，不能直接追加重复内容。
- 旧 `recordings list` 同样使用 ID 游标：`--page-size 20 --cursor CURSOR`，返回 `data.dataList/pageSize/next_cursor/has_more`，没有 `total/totalPages/pageNum`。保留 `--group-id/--app-tmp-id/--keyword`；`--page` 仅接受首页 1，不能用递增页码模拟翻页。`browse/repl` 保存已访问游标支持上一页，首尾重复操作不请求；Agent 自动化继续使用单次命令。
- 正文返回 `data.content/truncated/next_offset/total_chars`。`truncated=true` 时按 `next_offset` 续读；检查偏移确实向前推进，避免无限循环。板块为 `all/transcription/chapters/notes/summary`；检查 `sections` 的 `ready/not_ready/locked` 等实际状态，不把尚未生成当空内容。
- 需要完整转写/纪要且已有对应导出权限时，优先下面的 TXT/Markdown 导出，然后在本机按需分块读取；`recording get` 没有 `--full`。窗口接口仍会重复读取/处理正文，不能把字符 offset 当作数据库游标或认为连续拉取没有成本。无导出权限时只读取任务确实需要的已授权窗口，不改 scope 或绕过锁定。
- RAG 没有列表游标；`--offset/--length` 是文字窗口。命中范围受上游影响，不能声称搜索覆盖了全部录音。

## 下载与导出

```text
idock recording download --id RECORDING_ID --source original --out "./downloads"
idock recording download --ids ID1,ID2 --source mp3 --out "./downloads"
idock recording export --id RECORDING_ID --type transcription --file-format txt --out "./exports"
idock recording export --id RECORDING_ID --type summary --file-format word --out "./exports"
idock recording export --ids ID1,ID2 --type transcription --file-format md --out "./exports"
```

`ID1,ID2` 要替换成逗号分隔的录音 ID。类型：`transcription/summary`；格式：`txt/md/markdown/pdf/word`（Word 生成 `.docx`）。每批最多 200 条，按用户请求的范围执行。路径有空格时引用整个参数。

默认不覆盖现有文件；用户明确要求覆盖才加 `--overwrite`。查看 `data.results` 中每项 `success/path/errorCode/errorMessage` 及总计 `succeeded/failed`，成功文件要检查实际存在。`PARTIAL_FAILURE` 时分别汇报成功与失败，不丢弃已生成文件或重复整批。

原音按真实格式或分段保存，不自行合并或伪装成 MP3。原音缺失时可按用户音频需求尝试 `--source mp3`；无痕、锁定或权限拦截不能靠换来源绕过。`TINGWU_SUMMARY_NOT_READY` 表示该条纪要尚未生成，不能生成空文件并称导出成功。

转写的 TXT、md/markdown 自动使用“申请临时 URL → CLI 无凭证直连 OSS 下载 JSON → 本地转换保存”，无需 Agent 手工获取 URL 或拼接带 Key 的下载请求。只需 `transcription:export`，签名最长 15 分钟且不晚于 Key 到期；PDF/Word 与所有纪要格式继续由后端生成。URL 不出现在命令结果中，Agent 只使用返回且已存在的本地路径，不传播下载签名。

OSS 过期、断连等可重试错误默认额外重签一次，`--retries 0..5` 控制次数；与下面 API 用户限流的等待分开计算。源 JSON 的实际解压后字节上限为 16 MiB，异常/超限会清理临时源，不生成不完整文档。`TRANSCRIPTION_TOO_LARGE` 时如用户可接受 PDF/Word，可在相同录音和授权范围内改用该格式；不能声称已经获得完整 TXT。`TRANSCRIPTION_FORMAT_INVALID/TRANSCRIPTION_URL_INVALID` 时报告错误并核对前后端版本，不手工访问原始 URL 或静默反复拉取正文。

## 每用户限流与重试

所有 CLI API 按已认证用户统一限流，相邻两次放行至少间隔 1000 毫秒；多终端、多个接口、服务节点和重置后的 Key 共用额度，不同用户独立。不要并发发起同一用户的多条命令。导入、批量导出或 `doctor` 可能包含多次 API 调用，因此等待是正常行为；OSS 文件传输本身不计入后端 API 次数。

服务端在业务执行前拒绝时返回 HTTP 429 / `IDOCK_RATE_LIMITED` / `Retry-After: 1`。新版 CLI 已自动等待并最多额外请求两次，沿用原连接、请求体和幂等键；持续限流的最终退出码为 6，附 `error.details.retry_after_seconds`。此时停止额外循环并告知用户稍后重试，不能再叠加 Agent 的多层重试或换账号/Key/节点。批量可能以退出码 1 / `PARTIAL_FAILURE` 携带逐项限流错误，保留成功文件，只处理必要的失败项。

`IDOCK_RATE_LIMIT_UNAVAILABLE` 表示共享 Redis 限流不可用，服务端返回 503 且未执行业务。无可信限流重试头的 RAG 上游 429、401/403、503 和网络异常不走上述 API 自动重试。未知写入结果按导入的原任务/幂等键恢复，不盲目重放写入；任务等待超时继续查询原编号，不重新上传。

## 导入与任务恢复

确认用户要上传的文件与**实际时长**。`--duration-seconds` 必填，整数 1–14400 秒，不凭文件大小猜时长；可以复用用户给出的时长或当前环境已有可信媒体元信息工具，缺失时再询问。单文件非空且不超过 2GB，仅支持 wav/mp3/mp4/m4a，扩展名不区分大小写；服务端共享上传配置可能更低，不能承诺 2GB 一定可用。

```text
idock recording import "./会议.wav" --duration-seconds 3600
```

无 `--yes` 时只输出计划，不读取文件、不校验文件存在、不上传，也不代表 Key 已具备导入权限。核对路径、时长与上传转写意图；用户已明确授权这项操作时继续，否则说明会使用现有存储和转写额度并取得授权。

```text
idock recording import "./会议.wav" --duration-seconds 3600 --yes
idock job get JOB_ID
idock job wait JOB_ID --timeout 300 --interval 3
```

- 导入、任务查询和等待都要求 `recording:import`，默认六项读取权限不包含它。
- 默认以文件 SHA-256 为幂等键；也可在用户需要固定业务标识时设置 `--idempotency-key meeting-20261028`。键为 1–128 位字母、数字、下划线或横线，同一业务重试必须保持文件内容、大小、时长、类型和键一致。
- 保存实际返回的 `job_id/idempotency_key`，并检查 `status`。准备/上传/处理中不等于转写完成；`completed` 后使用 `recording_id` 定位结果。`failed` 根据原任务错误处理。
- `JOB_TIMEOUT`：任务仍在后台继续，稍后查询或等待相同编号。`UPLOAD_UNCERTAIN/UPLOAD_FAILED` 或完成确认失败：优先使用 `error.details.job_id/idempotency_key` 查询原任务，需要恢复时复用同一文件和键，不能换新键盲目重传。
- 任务元信息保留七天，超过期限任务查询可能失效；不能因为查不到就自动重新触发转写，先按用户目标查已保存的业务录音。

## 权限与故障

| 能力 | 所需 scope |
| --- | --- |
| 录音列表/正文 | `recording:read` |
| 原音/MP3 下载 | `recording:download` |
| 转写导出 | `transcription:export` |
| 纪要导出 | `summary:export` |
| RAG 检索 | `rag:search` |
| 四类闪记 | `flash:read` |
| 导入、任务查询/等待 | `recording:import` |

| 返回 | 处理 |
| --- | --- |
| 退出码 0，`executed=false` | 计划已生成，远端操作尚未执行 |
| 退出码 2 | 参数问题，核对对应 `--help` 后修正 |
| 退出码 3 / `IDOCK_ACCESS_KEY_INVALID` | 用户在自己的终端重新登录，不输出或替用户重置 Key |
| 退出码 5 / `IDOCK_SCOPE_FORBIDDEN` | 说明 `missing_scopes`；由用户在 App/管理页授权后重新登录，不能自行扩权 |
| 退出码 6 | CLI 内置限流重试已结束，报告等待提示/秒数并停止额外循环 |
| `IDOCK_RATE_LIMIT_UNAVAILABLE` | 限流存储故障，报告服务不可用，不换节点或 Key 绕过 |
| `TRANSCRIPTION_TOO_LARGE` | 未生成转写文本；如符合用户目标，可改导出同条录音的 PDF/Word |
| `IDOCK_RAG_UNAVAILABLE` | 说明向量检索不可用，不以数据库查询冒充语义搜索 |
| `RECORDING_AUDIO_UNAVAILABLE` | 读取具体原因，区分无痕限制、原音缺失和手机文件未上传 |
| `PARTIAL_FAILURE` | 保留成功项，汇报失败项，恢复时只处理失败项 |
| `RELEASE_CONFIG_INVALID` | 联系维护者获取已配置服务地址的包，不让普通用户猜网关 |

`idock doctor` 检查网络、认证与服务版本；不证明 RAG、OSS、后台转写均可用。空列表/空命中是成功结果，不是服务故障。

`idock auth logout` 只清除当前 profile 的本地 Key，环境变量注入的 Key 要由宿主另行清除。`idock auth revoke` 无 `--yes` 只是预览；只有用户明确授权撤销当前 Key 时才运行 `idock auth revoke --yes`，所有使用这把 Key 的设备都会失去授权。不要在普通排障时调用撤销。
