# Codex 连接嘉立创EDA：从安装到验证

本教程记录 Windows 本机连接流程。硬件示例为仓库中的 MSPM0G3507 最小系统板。连接需要 Codex 能够读取本地技能文件、执行本地命令并访问本机端口；远程执行环境里的 `127.0.0.1` 不代表你的 Windows 电脑。

## 1. 各组件的职责

| 组件 | 用途 |
|---|---|
| Codex | 接受设计要求、查询 API 文档、执行受授权的操作 |
| easyeda-api Skill | 提供连接流程、桥接脚本及 EDA API 参考 |
| Node.js | 运行本机桥接服务 |
| Bridge Server | 接收 Codex 的 HTTP 请求，通过 WebSocket 转发给 EDA |
| Run API Gateway | 安装在嘉立创EDA内，执行 `eda.*` API 调用 |
| 嘉立创EDA专业版 | 保存和编辑原理图、PCB及工程 |

本流程直接使用本机 HTTP/WebSocket 桥接。Skill 是工作流与参考文件，不等于 MCP 服务器；安装网关也不会自动把一个 MCP 工具注册进 Codex。

本项目使用过的版本：嘉立创EDA V3.2.149、Run API Gateway v1.0.5、easyeda-api 技能包 v1.1.3。其他版本的菜单与权限名称可能有差异，应以对应版本说明为准。

## 2. 获取工具与技能包

先准备本地 Codex、嘉立创EDA专业版，以及能运行 `node` 和 `npm` 的 Node.js 环境。技能包声明支持 Node.js 18+；安装时按上游当前要求选择受支持版本。

第三方工具入口：

- [嘉立创EDA专业版](https://pro.lceda.cn/)
- [Run API Gateway 扩展](https://jlc-ext.com/item/oshwhub/run-api-gateway)
- [嘉立创发布的 easyeda-api 技能包 ZIP](https://image.lceda.cn/files/easyeda-api.zip)

以上入口来自本项目使用的上游技能/扩展说明；本仓库不托管这些第三方安装包，下载内容与可用性由上游维护。

技能包要完整解压，不能只复制 `SKILL.md`。目录至少应有：

```text
easyeda-api/
  SKILL.md
  package.json
  scripts/bridge-server.mjs
  references/
```

注意避免出现 `easyeda-api/easyeda-api/SKILL.md` 这样的额外嵌套。

## 3. 让 Codex 找到技能，文件存放在 D 盘

按 [OpenAI 技能文档](https://learn.chatgpt.com/docs/build-skills)中的项目级发现规则，可以在本仓库本地目录下放置：

```text
D:/codex/3507/outputs/mspm0g3507-public/
  .agents/skills/easyeda-api/
    SKILL.md
    scripts/bridge-server.mjs
    references/
    package.json
```

在 Codex 中打开该仓库目录作为本地项目。仓库已忽略上述第三方技能目录，避免把依赖及上游代码一起提交。

如果技能发现列表没有刷新，重新打开本地任务；也可以明确告诉 Codex：

```text
请读取 D:\codex\3507\outputs\mspm0g3507-public\.agents\skills\easyeda-api\SKILL.md，
按其中说明连接本机嘉立创EDA。先检查连接和当前工程，不修改设计。
```

本次实践原先使用已登记的 `.codex/skills/easyeda-api` 路径。其他机器应按自己版本的技能配置使用实际目录，不要照搬原作者的用户名或系统路径。

## 4. 安装 EDA 扩展并授权

在嘉立创EDA专业版的扩展管理入口安装并启用 Run API Gateway；安装后顶部应出现 `API Gateway` 菜单。

根据扩展提示，由用户允许其外部交互权限。导出工程还需要相应的“工程管理 > 下载工程”权限。权限是否持久保存取决于 EDA 版本和用户设置；更新或重装后要重新检查。

启用后保持 EDA 运行。网关连接本身不要求已经打开工程；读取或修改原理图、PCB时，需要打开正确工程与页面。

## 5. 启动本机 Bridge Server

也可以让 Codex 按技能说明启动。手动操作时，在 PowerShell 中进入实际技能目录并安装依赖：

```powershell
Set-Location -LiteralPath 'D:\codex\3507\outputs\mspm0g3507-public\.agents\skills\easyeda-api'
npm install
```

安装成功后，在同一目录启动桥接服务，并保持这个 PowerShell 窗口运行：

```powershell
node .\scripts\bridge-server.mjs
```

如果由 Codex 代为启动，应让它后台运行，避免前台服务占住工具调用。不要重复启动多个服务实例。桥接通常在 `49620–49629` 中选择一个可用端口，不能一律假定是 `49620`。

本实践检查到的桥接脚本只监听 `127.0.0.1`。它能转发 EDA 操作代码，保持本机监听即可，不要为连接本机 EDA 开放公网端口。

## 6. 验证桥接和 EDA 都已连接

在另一个 PowerShell 窗口执行以下只读检查。它只扫描技能约定的本机端口，并验证服务标识：

```powershell
$bridgeBase = $null
foreach ($port in 49620..49629) {
    try {
        $candidateBase = "http://127.0.0.1:$port"
        $health = Invoke-RestMethod -Uri "$candidateBase/health" -TimeoutSec 1
        if ($health.service -eq 'easyeda-bridge') {
            $bridgeBase = $candidateBase
            $health | ConvertTo-Json -Depth 5
            break
        }
    } catch {
        continue
    }
}
if (-not $bridgeBase) { throw '没有找到本机 easyeda-bridge 服务' }
Invoke-RestMethod -Uri "$bridgeBase/eda-windows" -TimeoutSec 3 |
    ConvertTo-Json -Depth 5
```

健康检查至少应确认 `service` 为 `easyeda-bridge`，`edaConnected` 为 `true`。只有桥接进程运行、但 `edaConnected=false`，还不能称为连接成功。

单个 EDA 窗口一般会自动选中；多个窗口时，先根据 `/eda-windows` 返回的窗口 ID 选定目标，不能直接向不确定的窗口写入。

## 7. 做一次只读 API 调用

先在 EDA 中打开目标工程并点选原理图或 PCB 页。在运行完上面的端口发现代码、同一个 PowerShell 会话里执行：

```powershell
$code = 'return await eda.dmt_Project.getCurrentProjectInfo();'
$body = @{ code = $code } | ConvertTo-Json -Compress
Invoke-RestMethod -Uri "$bridgeBase/execute" -Method Post `
    -ContentType 'application/json; charset=utf-8' `
    -Body ([Text.Encoding]::UTF8.GetBytes($body)) -TimeoutSec 40 |
    ConvertTo-Json -Depth 10
```

`success=true` 且返回对应工程信息，说明这次工程读取成功。没有活动工程时，结果可能为空，不能据此认定桥接故障。所有写入前，Codex 应读取技能中的对应 API 签名，并确认活动工程、文档类型和目标窗口。

## 8. 直接复制给 Codex 的提示词

首次检查：

```text
使用 easyeda-api 技能连接本机嘉立创EDA，发现桥接端口并检查 health。
报告 EDA 是否连接、活动窗口、当前工程和当前页面类型。只做读取。
```

编辑现有工程：

```text
连接嘉立创EDA里的 mspm0g3507 工程，先确认活动工程和页面。
按我给的数据手册修改指定电路，保存后分别运行原理图和 PCB 的严格 DRC，
报告实际错误和警告。数据手册没有确定的引脚或参数先标记待确认。
```

重启后恢复：

```text
请按 easyeda-api 技能恢复本机 EDA 连接：先发现已有桥接，
没有桥接再后台启动；确认网关已连接后读取当前工程，不创建重复服务。
```

扩展和技能可以保留安装；但“已安装”不代表桥接进程、EDA窗口和外部权限永远有效。每次恢复连接都应重新检查状态。

## 9. 常见问题

| 现象 | 检查方向 |
|---|---|
| 找不到 `node` / `npm` | 检查 Node.js 安装与 PATH，重新打开终端 |
| 脚本提示缺少 `ws` | 在技能包的 `package.json` 所在目录安装依赖 |
| 所有端口都不可访问 | 检查桥接是否启动；Codex 沙箱需允许本机网络访问 |
| `edaConnected=false` | 检查 EDA 是否运行、扩展是否启用；在 API Gateway 菜单重连 |
| 工程结果为空 | 打开工程，点击实际原理图或 PCB 页，再读取 |
| 原理图 API 获取器件失败 | 检查当前是否误停留在 PCB 页；两种页面 API 不能混用 |
| 导出 API 持续拒绝 | 检查扩展下载工程权限，不能靠重试绕过权限限制 |
| 新任务找不到技能 | 检查完整技能目录、嵌套层级、项目根目录和技能发现状态 |
| 多个窗口连接 | 用 `/eda-windows` 获取真实窗口 ID，明确目标再调用 |

## 10. 本项目验证过什么

2026-10-28，在本机重新启动 EDA 与桥接后，检查到了 `edaConnected=true` 和一个活动 EDA 窗口，并通过该桥接成功导出本仓库中的完整 `.epro` 工程。以上端口发现与只读工程查询示例也在本机运行过；示例查询时 EDA 停留在起始页，返回 `success=true`、`result=null`，符合当时没有活动工程的状态。4 段 PowerShell 示例均通过语法解析检查，未重新执行安装依赖和启动服务的两段示例。

这验证的是本机连接和工程导出链路；没有在一台全新电脑上重做安装测试。公开教程中的 D 盘技能安装路径是可复现示例，不是要求其他用户使用原作者机器配置。
