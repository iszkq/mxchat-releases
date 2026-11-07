# mxchat 源码与版本发布流程

## 仓库职责

- 源代码、问题修复、版本号、工作流、测试与发布说明：[iszkq/mxchat](https://github.com/iszkq/mxchat)。以后仍在这里更新原代码。
- 公开下载、GitHub Releases、客户端自动更新元数据：[iszkq/mxchat-releases](https://github.com/iszkq/mxchat-releases)。此仓库不维护应用源码。
- 源码仓库可以保持私有，**版本仓库必须保持公开**，否则无凭证的客户端无法更新。客户端不保存 GitHub Token。

## 版本规则与客户端行为

版本使用递增稳定语义版本，例如 `0.3.58` → `0.3.59`，标签为 `v0.3.59`。不要覆盖已经发布的标签或附件。草稿、预发布和低版本不推送给稳定客户端。

Windows 正式安装版启动约 15 秒后检查，以后每 6 小时检查；「设置 → 关于 mxchat」也可手动检查。不会自动下载，不会在普通退出时自动安装。下载后校验 SHA512，点击「重启安装」并确认才启动安装程序。数据目录、appId 和安装方式保持不变。开发模式/网页不支持安装更新。

0.3.57 及更早版本需要手动安装 0.3.58 一次。新版安装到原目录，保留本机数据。当前安装包未签名，哈希不是签名，正式推广前建议配置 Windows 代码签名并验证证书轮换。

## 每次发布

1. 在**源码仓库**更新、测试代码；修改 `package.json` 与 `package-lock.json` 的版本号，新建 `release-notes/v版本号.md`。不要改变 `build.appId`、更新仓库、NSIS 安装标识和数据目录。
2. 执行 `npm ci`、相关测试、`npm run build:tools-dashboard`、`npm run dist:win -- --publish never`。推荐干净构建；必须使用同一轮构建的 `.exe`、`.exe.blockmap`、`latest.yml`。不要使用 `--publish always` 直接发布不完整版本。
3. 提交并推送源码到 `iszkq/mxchat` 的 `main`，确认 Actions 构建通过。源码提交号将写入公开版本说明（访问私有源码仍需权限）。
4. 在源码仓库目录设置仅对版本仓库具有 Contents 读写权限的 `GH_TOKEN`，然后执行 `npm run release:public`。脚本先验证版本、大小与 SHA512，创建公开仓库中的**草稿**，上传全部附件，验证上传大小/摘要后才发布为 Latest。令牌只留在进程环境，不得写入任何源码、客户端、YAML、日志或文档；执行完清除环境变量。
5. 未配置发布 Token 时，也可在版本仓库网页手动建草稿，按下面清单全部上传、检查后再正式发布。GitHub 自动生成的 Source code.zip 只是版本仓库文档，**不是安装包或应用源码**。
6. 用旧版正式客户端验证发现新版本、下载校验、取消/重试；在受控测试设备验证确认后的重启安装、版本号及原登录/本机数据保留。不能以上传成功代替安装验收。

```powershell
# 已完成上述构建，源码已推送。不要将真实 Token 写入脚本。
node scripts/publish-release.mjs --verify-only
# 在当前终端安全设置 GH_TOKEN 后：
npm run release:public
Remove-Item Env:GH_TOKEN
```

`--verify-only` 不访问网络，可先验证本地附件。脚本失败会保留草稿，不自动删除/覆盖；先检查草稿和附件，再决定补齐或删除未发布草稿后重试。使用官方来源和正常 TLS，不关闭证书或安装包校验。

## 一个 Release 必须包含

| 文件 | 用途 |
| --- | --- |
| `mxchat-版本号-Setup.exe` | Windows x64 NSIS 安装包 |
| `mxchat-版本号-Setup.exe.blockmap` | 增量下载辅助；不支持时回退完整下载 |
| `latest.yml` | 与安装包同构建的版本、文件名、大小、SHA512 |
| `mxchat-版本号-Setup.exe.sha256` | 手工校验 |

可选附件仅包括后台 `worker.txt`。具体小工具源码、数据、ZIP、校验文件只在本地制作，使用独立文件托管；不得附加到源码仓库、Actions 产物或客户端 Release。发布脚本不自动收集小工具。

## GitHub Actions

推送 `main` 只构建并上传 Actions 产物，不自动对外发布。手动运行 **Build Windows installer**，勾选 `publish_release` 才会发布到版本仓库。需先在**源码仓库** Actions Secrets 设置 `RELEASES_TOKEN`（建议 Fine-grained PAT，仅选择 `mxchat-releases`，Contents: Read and write）。默认 `GITHUB_TOKEN` 只对源码仓库有效，不能跨仓库上传。不要把本机已登录的宽权限凭证自动复制为 Secret。建议配置仓库环境审核及签名。

## 更新故障和回退

- GitHub 访问受限、API 限流或无正式 Release：保留当前应用，稍后手动重试，或从公开版本仓库手工下载。
- 摘要失败：不安装，检查同一 Release 的安装包和元数据是否匹配。不要跳过验证。
- 新版有问题：发布递增补丁版本；自动更新禁止降级。手动回退前先备份数据，较新数据格式不保证可被旧客户端读取。
- 后台部署独立进行：把 `dist-admin/worker.txt` 部署到现有 Cloudflare Worker，保留 `CATALOG` KV 与 `ADMIN_TOKEN`，刷新浏览器。发布桌面版本不会自动更新后台。

## 小工具：组件、独立窗口与软件下载

分类名称（如阅读、软件）只负责分组；运行类型决定按钮。计算 SHA256 时会按真实格式和工具清单自动识别：

| 内容 | 类型与按钮 |
| --- | --- |
| 表情清单 JSON、支持的语音/数据 ZIP 清单 | mxchat 内组件：安装、更新、已安装 |
| ZIP 清单 `kind: web` | 独立窗口：安装后使用 |
| EXE / MSI / 7Z、无 mxchat 清单的普通 ZIP / MSIX | 软件资源：始终下载，可重复下载 |

后台选择同一个本地资源文件即可生成 SHA256，无需把文件上传给 Worker；云盘跨域失败时优先用本地计算。后台仅保存目录，文件需先托管为 HTTPS 直链（使用独立资源托管，不使用本仓库 Release）。软件资源还要核对保存文件名，支持 `.exe .msi .msix .zip .7z`，上限 900 MB。不支持脚本、快捷方式，也不把未知 JS/DLL 当作 mxchat 插件执行。有工具清单的 ZIP 优先按清单识别；若只想当文件分发，可手动选择“软件 / 资源下载”后重新计算。

客户端下载按钮在卡片与详情均可见。每次点击先弹系统保存路径；保存前 SHA256 和格式校验，取消或失败不替换已有文件，成功后仍可再次下载。不自动打开/安装，也不在 mxchat 已安装清单中登记。实际安装时用户应自行核对发布者与签名。

在本地制作并托管新版工具 ZIP 后请同步更新后台直链和 SHA256；已有工具应编辑原条目，保留 ID，关闭工具窗口再更新。后台也须部署本版 `dist-admin/worker.txt` 才能识别 software/web 类型；仅推送源码或发布客户端不会更新已经部署的 Worker。

## 小工具本地维护

按维护者要求，具体小工具已从当前源码树与既有 Release 附件移除；历史 Git 提交未改写。已有后台指向被移除附件的条目需更换为独立资源托管直链与对应 SHA256，保留工具 ID。客户端的通用安装、独立窗口与软件下载能力仍在源码中维护。不要再次提交本地插件目录、经文数据、工具测试或构建包；`.gitignore` 与 CI/发布脚本均排除这些内容。
