# DSH-LINK-02 验收记录

状态：开发完成，等待本地插件部署验收。

## 范围和根因

基线 main / v0.6.11。没有修改已安装插件、Codex MCP 配置、DSH 核心或 Notion。
原有 sessionLog() 已支持 inspect 和 controller.events() 分页读取。read_session 仅返回消息及工具名称，运行事件摘要也仅保留结果状态，因此原始结果未对 MCP 暴露。
只读检查本机 DSH app.asar 中 toolCallUpdate / toolResultUpdate 确认：调用使用 data.callId/name/arguments，结果使用 data.message.toolCallId/content/isError。未从助手文字提取结果。

## 修改

- host/lib/index.js：新增 read_tool_events，复用 sessionLog，按调用 ID 关联，保留 call_event/result_event 原始对象。
- src/core.mjs：新增 Host 转发；无 Host 则报错，不回退至助手文本。
- src/mcp.mjs：注册只读 dsh_read_tool_events，同时可经 dsh_call 访问。
- tests/host.test.mjs：复用既有模拟 Host 和真实本地 socket，增加关联、异常、截断、MCP 端到端测试。host-paged 自动重复分页验证。

## 接口

```json
{"session_id":"session-xxxx","tool_name":"mcp__notion_poc__check_notion_connection","last":10}
```

session_id 必填；tool_name 可选，精确匹配；last 默认 10，整数 1–50。
返回 source=session_tool_events、total_matches、has_more、truncated、events。
每行含 call_id、tool_name、arguments（保持原始字符串）、state、isError、call_event、result_event。
结果正文位于 result_event.data.message.content，时间及 seq 保留在原始事件中。
state：pending 表示未见结果；failed 表示 isError=true 或事件含 error；succeeded 表示明确 isError=false；未提供错误标志的结果为 completed，避免推断成功。
无匹配：events=[]、total_matches=0。输入无效或读取失败：MCP isError=true。
没有对应调用的结果保留为 tool_name=null、call_event=null，不猜测名称；指定名称过滤时不包含这种行。
按调用事件顺序返回最后 N 次调用；没有调用的结果按结果出现位置排列。

输出约 240 KB（UTF-8 JSON，另有少量元数据）；超长行替换为 truncated=true、original_bytes、raw_json_prefix。该前缀不是完整 JSON，不可作为完整结果验收。has_more 表示数量限制省略了较旧调用，与体积截断不同。

## 尚未验证与限制

真实 Desktop / Notion 验收待部署后完成。本阶段不能声称已核验真实 state、writes_enabled 或三项连接检查。
现有 sessionLog 会读取全会话历史；本次只限制返回数量和体积，没有改造底层历史分页机制。附件引用保持原样，不下载附件。只关联正式 tool/call 和 tool/result，不将 assistant/message 的文字或内容块充当工具事件。

## 部署和回退（仅方案，须用户确认后执行）

依据本机 Desktop 自带插件安装说明：添加插件接受本地目录；同名已安装包须先卸载后重新安装。因此仅停用旧版可能仍阻止本地安装。

1. 记录旧版版本、来源、启用状态，备份 Codex config.toml 中 dsh-baton 配置。结束正在运行的 DSH 工作后，在插件页停用旧版。
2. 因包名相同，在插件管理中卸载旧 dsh-baton-host（不删除会话）。添加插件，输入 `C:\Users\Chucky\Work\dsh-baton\host`，按 Desktop 安装预览加载本地目录并启用。如提示需重启则重启 Desktop。不要手工覆盖安装目录。
3. 本地 Host 启用时可能自动注册 MCP，这是现有机制。确认注册结果指向本仓库 `src/server.mjs`，确认没有仍占用同一本地 socket 的旧 Host；必要时重启 Codex，新建聊天。
4. 确认 tools/list 包含 dsh_read_tool_events；随后执行上述调用。版本仍为 0.6.11，不能只用版本号判断是否切换成功。旧聊天可尝试 dsh_call，但仍须确认 MCP 与 Host 都加载了修改版。
5. 对已有目标会话读取 check_notion_connection，核对原始 result_event 中 callId 关联、state=succeeded、writes_enabled=false 及三项连接检查。另核对同会话多次调用。不发送 DSH 指令，不调用写入工具，不需要打开轨迹。
6. 回退：停用并卸载本地版，在插件页添加记录的官方 npm 包版本并启用；重启提示涉及的应用。确认 MCP 注册恢复官方安装路径；必要时恢复已备份的 dsh-baton 配置段。调用 dsh_list_workspaces 和 dsh_read_session 验证原功能。

本地目录安装、自动注册具体效果与真实 Notion 原始结果均属于待部署验收项目。

## 自动化测试结果

- `npm test`：274 项，270 通过、3 跳过、1 失败（新增公开工具使旧目录总体字符上限失败：12793 字符）。没有生产行为回归失败。
- 保留既有工具的原字符预算，新增工具单独限制在 1000 字符内；修改 tests/catalog.test.mjs 后重跑 `node --test tests/catalog.test.mjs`：2/2 通过。未再次运行全量套件。
- `node --test tests/host-paged.test.mjs tests/host.test.mjs`：130 项，128 通过、2 跳过、0 失败。
- 新增定向测试：4/4 通过，包含 MCP→core→Host socket→原始事件读取与 dsh_call 同路径调用。
- `git diff --check`：通过。

目录体积测试另有一处最小调整：为新增接口提供独立预算，旧接口预算和行为保持不变。
基线提交：27856b4。

## 2026-11-12 部署前检查（步骤 1–3）

- 完整 `npm test` 已重新执行：274 项，271 通过、3 跳过、0 失败，退出码 0，耗时约 172 秒。覆盖前次目录预算修复。日志保存在下述备份目录 npm-test.txt。
- 当前分支 main；HEAD=27856b463a7344952da32263719acb6ae97f8bef。保留全部未提交改动：host/lib/index.js、src/core.mjs、src/mcp.mjs、tests/catalog.test.mjs、tests/host.test.mjs，以及本验收文档。已逐项检查 diff；git diff --check 通过。没有新增功能或修改业务代码。
- 配置备份目录：`C:\Users\Chucky\.codex\backups\DSH-LINK-02-20261112-142824`。config.toml 为完整私有备份；SHA-256：7491BF8579371247C4723CCE03F02FE2E2D080886C15E7A4AF18B0B435C26784。备份与当前配置一致。
- 同目录另保存 Desktop package.json、cordis.yml、cordis.patch.yml、pnpm-lock.yaml 与基线文件哈希，供比对 QQ / Notion 配置及回退参考。备份可能包含敏感配置，不提交 Git。
- Codex 当前注册：command=`C:\Program Files\nodejs\node.exe`，args=`C:\Users\Chucky\.dsh-baton\app\src\server.mjs`，cwd=`C:\Users\Chucky`，tool_timeout_sec=3600。未显式配置 enabled=false；已有 MCP 进程从该官方副本启动，dsh_list_workspaces 实际读取成功。
- Desktop profile=`C:\Users\Chucky\.dsh\profiles\desktop`；依赖 dsh-baton-host=`^0.6.11`，bundles 包含该插件；已安装 Host 包版本 0.6.11，位于该 profile 的 node_modules/dsh-baton-host。只读 ping 返回 name=dsh-baton-host、version=0.6.11、pid=11220，确认启用且运行。
- 未卸载、停用或替换任何插件，未改写 Codex 配置，未开启写入或创建 Notion 任务。QQ / Notion 此时仅建立配置基线，运行回归待用户切换后进行。

下一步由用户在 Desktop 中卸载官方 dsh-baton-host，并通过本地目录 `C:\Users\Chucky\Work\dsh-baton\host` 安装、启用修改版。用户完成切换后继续检查 Host / MCP 来源和真实原始工具事件；当前不标记最终验收通过。

## 2026-11-12 用户切换后的验证

状态：本地 Host 与本地 MCP 子进程验证通过；等待 Codex 重新连接后的最终验收，不提前标记全流程通过。

- Desktop profile 的 dsh-baton-host 依赖已为 link:C:/Users/Chucky/Work/dsh-baton/host；node_modules 中 Junction 目标为同一路径。Host ping PID 从 11220 变为 129788，新增 read_tool_events 方法已实际返回原始事件。这些证据确认运行修改版，不仅依赖 0.6.11 版本号。
- 发现既有 autoRegister 会保留有效的旧 MCP 注册（state=kept），所以 Host 切换后 Codex 仍使用 ~/.dsh-baton/app。经用户明确授权，仅将 config.toml 中 dsh-baton 的 args 路径替换为 C:\Users\Chucky\Work\dsh-baton\src\server.mjs；替换前再次备份为 config.before-local-path.toml，并通过字节级逆向比较确认无其他变更。未改写其他 MCP 配置。
- 本地 src/mcp.mjs 子进程通过 tools/list 和 tools/call 成功读取真实 Desktop 会话 session-da44329c-1c1c-4eec-9aed-a38f057fa158；并非助手文字推测。
- 目标工具 mcp__notion_poc__check_notion_connection 有 5 次调用。名称过滤读取未截断。最新 callId=call_00_ET_IxL6Ig1nKFzttWiQVg5Z1283，call_event.seq=240，result_event.seq=241；两者 callId/toolCallId 一致，arguments="{}"，isError=false，state=succeeded。实际正文为：

```json
{"state":"succeeded","writes_enabled":false,"checks":{"identity":{"state":"succeeded","id":"40ca063a-a11c-8139-9682-0027cbf575c6"},"backlog":{"state":"succeeded","id":"2aea063a-a11c-804f-9393-000b961b95f0"},"sprint":{"state":"succeeded","id":"241a063a-a11c-80c8-a8df-000b38c8f3b9"}}}
```

- 历史 seq 177/178 的检查正文 writes_enabled=true，最新 seq 240/241 为 false；如实保留历史差异。本次只读取历史，不代表执行了新的在线连接探测。
- 同会话无名称过滤返回 26 条记录，所有具有完整调用/结果的记录 ID 均匹配。较长记录明确 truncated=true；存在无调用的结果时名称为 null，不猜测。不存在工具名的查询返回 total_matches=0、events=[]、truncated=false。
- 私有证据保存在备份目录 local-mcp-evidence.json，含本地 MCP tools/call 的真实响应。该宽查询有部分行截断，不用于宣称所有历史正文完整。
- 原有 dsh_list_sessions、dsh_status、dsh_read_session 实际调用成功；会话 busy=false，无新任务。dsh_send 等会改变会话的工具仅以此前自动化测试覆盖，未做真实发送回归。
- Desktop cordis.yml、cordis.patch.yml 与切换前备份逐字节相同；QQ 插件包清单哈希保持基线。QQ/Notion 配置未被本次切换修改。但尚未独立核验切换后的 QQ 在线收发和 Notion 新的网络连接状态；历史记录不能替代当前在线检查。
- 未开启 Notion 写入，未创建新 Notion 任务，未修改 DSH 核心或 qbot_dsh 源码。

剩余步骤：用户重新连接 MCP 或重启 Codex，再从聊天实际连接的 dsh_read_tool_events（或 dsh_call）读取同一目标事件，并核对运行进程路径；随后确认最终验收范围与 QQ/Notion 当前运行状态。若重连失败，优先通过备份还原该 args 路径并恢复官方插件。

## 2026-11-12 Codex 重连后的最终验收

状态：DSH-LINK-02 核心能力与部署验收通过；QQ 实际收发及 Notion 新在线连接探测未执行，不将配置一致性扩大为全业务验收。

- 当前聊天的工具目录已直接暴露 dsh_read_tool_events，并已通过实际连接调用成功（不是临时子进程代替最终连接）。运行中的 server.mjs/mcp.mjs 进程路径均为 C:\Users\Chucky\Work\dsh-baton\src；观察到 server PID 107016/56408/118508，mcp PID 135596/192956/135764。Host 本地目录链接及实际新增方法的证据见上一节。
- 当前聊天调用 dsh_read_tool_events，session_id=session-da44329c-1c1c-4eec-9aed-a38f057fa158，tool_name=mcp__notion_poc__check_notion_connection，last=1，返回最新 seq 240/241、callId=call_00_ET_IxL6Ig1nKFzttWiQVg5Z1283。实际 result_event.data.message.content 的 JSON 确认 state=succeeded、writes_enabled=false、identity/backlog/sprint 均 succeeded；isError=false，truncated=false。
- last=10 返回同名 5 次调用，所有 call_id/call_event.data.callId/result_event.data.message.toolCallId 相同，未截断。没有匹配名称的查询 total_matches=0、events=[]。失败与超长场景由全套自动化测试及前述真实宽查询覆盖，未故意触发外部工具失败。
- 当前连接的 dsh_list_workspaces、dsh_list_sessions、dsh_status、dsh_read_session 均实际调用成功。未运行会发送指令的 dsh_send，相关行为以 271 通过、3 跳过、0 失败的完整测试为据。
- 实际连接返回证据已保存到私有备份目录 codex-connected-tool-result.json。无需人工打开 Desktop 轨迹。
- 重连后再次比较 Desktop cordis.yml / cordis.patch.yml 与部署前备份，保持一致。QQ 插件未被改动，Notion server.py 进程存在，配置 DSH_NOTION_WRITES_ENABLED=false。未测试 QQ 真实收发，也未新发起 Notion 网络请求；本次读取的是 2026-11-11 的已有事件，不证明当前远端服务可用性。
- 未创建 Notion 任务、开启写入或修改 DSH 核心/qbot_dsh 代码；没有丢弃或提交 Git 改动。

部署已保留本地版。回退仍按本文方案：停用/卸载本地插件，安装 dsh-baton-host@0.6.11；将 Codex args 恢复为 C:\Users\Chucky\.dsh-baton\app\src\server.mjs（只恢复该字段，避免覆盖以后新增的其他配置），再重新连接 MCP。完整旧配置及安装基线位于上述私有备份目录。
