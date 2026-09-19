# WebBrain Windows 兼容性分析

> 分析日期：2026-08-16 · 依据：windows-compat-checker 脚本扫描（`out/05`）+ 源码核实
> 总体结论：**扩展本体在 Windows 上完全可用（官方支持 Chrome 与 Edge 商店安装）；开发/构建工具链存在少量小问题，均不影响核心功能。**

## 1. 总体评级

| 组件 | Windows 兼容性 | 说明 |
|---|---|---|
| 扩展运行时（Chrome/Edge） | ✅ 完全兼容 | 纯浏览器 JS，官方 Edge Add-ons 在架 |
| 扩展运行时（Firefox） | ✅ 完全兼容 | 纯浏览器 JS |
| `npm test` 主套件 | ✅ 兼容 | 纯 Node，`&&` 在 cmd.exe 有效，spawnSync 用 `process.execPath` |
| MCP 服务器（npx） | ✅ 兼容 | 纯 Node ≥20 + stdio + ws，无平台分支 |
| LM Studio 插件 | ✅ 兼容 | 纯 Node ≥20 |
| Playwright e2e（`ci:e2e`） | ✅ 预期兼容 | Playwright 原生支持 Windows（本机未运行验证） |
| `npm run sync:logo` | ⚠️ 有摩擦 | 调用 `python3`，Windows 上命令名通常为 `python`/`py` |
| 子包 `npm run clean` | ❌ cmd.exe 下失败 | `rm -rf dist`（mcp-server、lmstudio-plugin） |
| `.sh` 辅助脚本 | ⚠️ 需 Git Bash/WSL | 见第 5 节 |
| CI 验证 | ➖ 无 Windows 覆盖 | 全部 GitHub Actions 工作流仅 ubuntu |

## 2. 扩展本体（核心产品）：无问题

- 代码只使用 WebExtensions API（`chrome.*` / `browser.*`）与标准 Web API，不含 Node API、child_process、平台分支——浏览器扩展天然跨平台。
- manifest 的 `host_permissions` 含 `http://localhost/*`、`http://127.0.0.1/*`：连接 Windows 本地模型服务（Ollama、LM Studio、llama.cpp 等）与在 Linux/macOS 上完全一致。
- `agent.js` 中的本地地址判断（如 `_isLocalIpv6Host`，`agent.js:1650`）基于 host 字符串解析 `::1`/ULA/链路本地段，与操作系统无关。
- 商店渠道本身即含 **Edge Add-ons**（Windows 主要浏览器），开发者加载（`chrome://extensions` → Load unpacked → `src/chrome`）在 Windows Chrome/Edge 上即开即用，无需构建。

## 3. npm scripts 检查（脚本扫描 + 核实）

### 3.1 `&&` 链——无问题

脚本扫描标记了 `test`、`test:ci`、`build:web`、`sync:logo` 中的 `&&`。**这是误报类风险**：npm 在 Windows 上默认经 cmd.exe 执行，而 cmd.exe 原生支持 `&&`。这些脚本可正常运行，无需修改。

### 3.2 真实问题 ①：`python3` 命令名

```json
"sync:logo": "python3 scripts/sync-logo-assets.py && python3 scripts/gen-store-promos.py && node ..."
```

Windows 的 Python 安装通常提供 `python` 或 `py` 启动器；`python3` 仅在启用 Microsoft Store 应用执行别名或手动配置后可用。**影响范围**：仅品牌资产再生成（logo 同步、商店宣传图），普通开发/测试/发布 zip 不受影响。

修复建议（任选其一）：

```jsonc
// 方案 A：跨平台写法（python3 优先，回退 python）
"sync:logo": "python3 scripts/sync-logo-assets.py || python scripts/sync-logo-assets.py && ..."

// 方案 B：文档说明（CONTRIBUTING.md 注明 Windows 用户先 py -3 -m venv 或设置别名）
```

### 3.3 真实问题 ②：`rm -rf`（子包 clean 脚本）

`mcp-server/package.json:24` 与 `lmstudio-plugin/package.json:19`：

```json
"clean": "rm -rf dist"
```

cmd.exe 无 `rm`，`npm run clean` 直接失败（对发布无影响——`prepack`/`prepare` 只跑 `tsc`）。

修复建议：改用跨平台等价物，例如 `node -e "fs.rmSync('dist',{recursive:true,force:true})"`，或引入 `rimraf`（会新增依赖，与项目零依赖风格冲突，前者更合适）。

## 4. PATH 分隔符标记——全部为误报（已逐一核实）

脚本扫描标记了 10 处 `split(':')` / `join(':')`。逐一核实源码后确认**没有一处是 PATH 分隔符**：

| 位置 | 实际用途 |
|---|---|
| `agent.js:1641-1651` | IPv6 地址段解析（`mapped.split(':')` 十六进制段 → 数值；`_isLocalIpv6Host`） |
| `agent.js:23752-23758` | 剥离尾部 `:line:col` 形式的源码位置后缀（`parts.slice(0,-2).join(':')`） |
| `agent.js:24539/24552`、`cdp-client.js:3347`、`content.js:5020`、`rich-text-toolbar-heuristic.js:158`、`apocalypse-mode.js:1285` | 构建复合缓存/观测键（如 `[tag, role, label].join(':')`、`[id, generation, updatedAt].join(':')`） |

这些字符串键与文件系统无关，在 Windows 上行为完全一致。**无需任何修改。**

## 5. Shell 脚本

| 脚本 | 用途 | Windows 处置 |
|---|---|---|
| `docs/vision-models/sources/deepseek-v4-flash/scripts/*.sh`（8 个，zsh/bash） | RunPod/Linux GPU 云端模型实验环境 | 与本仓库核心产品无关（文档附带资料），不需要在 Windows 运行 |
| `test/smd-tests/start-chrome-debug.sh` | 以调试端口启动 Chrome 的测试辅助 | 需 Git Bash 或 WSL；或手动用 `chrome.exe --remote-debugging-port=...` 等价执行 |

仓库无 `.ps1`/`.cmd` 辅助脚本；`scripts/` 下的构建/发布脚本全部是 Node `.mjs`（用 `node:path`、`path.join`），跨平台安全。

## 6. 平台守卫与 child_process

- 扫描未发现 `process.platform` 守卫——经核实**无需**：扩展代码不接触 OS；Node 工具链（`test/run.js`、`ci/run.mjs`、`scripts/*.mjs`）全部使用 `node:path`/`pathToFileURL` 等跨平台 API，`spawnSync` 仅以 `process.execPath`（Node 自身）为目标。
- katex.min.js 中的 "exec" 匹配为压缩代码字面量误报，非 child_process 调用。
- 无 Unix 信号处理（`SIGTERM` 等）、无符号链接操作、无硬编码 `/usr` 路径、无 `HOME` 环境变量依赖（扩展数据全部走 `chrome.storage` / IndexedDB / OPFS）。
- 本地模型探测（llama.cpp/Ollama/LM Studio 元数据）基于 HTTP 端口，Windows 上的 LM Studio/Ollama 原生 Windows 版均可被自动发现。

## 7. CI/CD

全部 6 个 GitHub Actions 工作流（`main.yml`、`cloud-e2e.yml`、`webmcp.yml`、`minor-release.yml`、`webbrain-cloud-smoke.yml`、`coupon-domain-refresh.yml`）仅 `ubuntu-latest/24.04`，**无 Windows runner**。鉴于代码库为纯浏览器 + 纯 Node，跨平台风险本来就低，但若要正式保障 Windows 开发体验，可在 `main.yml` 加一个 `runs-on: windows-latest` 的 `npm test` 矩阵项（成本：CI 时长约增一档）。

## 8. Windows 用户实操建议

1. **只想用扩展**：直接从 Chrome Web Store 或 Edge Add-ons 安装，无任何 Windows 特有步骤；本地模型推荐 LM Studio 或 Ollama 的 Windows 版。
2. **想跑测试**：`npm install` → `npm test`——在 cmd/PowerShell/Git Bash 均可。
3. **想用 MCP 服务器**：`claude mcp add --transport stdio webbrain -- npx -y @webbrain/mcp-server`，Node ≥ 20；桥接 URL `ws://127.0.0.1:17374/extension` 在 Windows 上同样有效。
4. **要跑 e2e**：`npm run ci:e2e` 需要 Playwright 下载浏览器，Windows 原生支持。
5. **要再生成品牌资产**：先确保 `python3` 可用（`py -0` 检查；必要时 `Set-Alias python3 python` 或用 `py -3` 手动执行两个 Python 脚本），或跳过该命令。
6. **子包清理**：在 Git Bash 中运行 `npm run clean`，或直接手动删除 `dist/` 目录。

## 9. 结论

WebBrain 对 Windows 的兼容状态**良好**：产品本体（浏览器扩展 + MCP 服务器 + LM Studio 插件）为零依赖纯浏览器/纯 Node 实现，在 Windows 上无功能缺失。仅存的两处真实摩擦（`python3` 命令名、子包 `rm -rf` clean 脚本）都在低频辅助路径上，修复成本低（各一行）。自动化扫描报告中的其余标记（`&&` 链、`:` 连接、katex "exec"）经源码核实均为误报。
