# XSS 扫描器

面向 HTML 与页面内 JavaScript 的本地 XSS 分析工具。支持上传 HTML、输入网址或粘贴源码，用静态污点分析给出来源、汇点、行号和传播路径，也可以追加 DeepSeek 研判，辅助检查静态规则没有覆盖的写法。

![界面：拖文件、粘贴网址或直接贴源码](docs/screenshot-ui.png)

![报告：静态分析结果与模型疑似项分开展示](docs/screenshot-findings.png)

## 快速开始

需要 Python 3.10 或更新版本，CI 使用 Python 3.12。

```bash
pip install -r requirements.txt
python webapp.py
```

启动后会打开本地网页，地址里带本次启动的访问 token。选择输入方式并扫描，结果可在页面查看、复制为 JSON。默认只做静态分析；上传文件或粘贴源码时不联网，输入网址时会向目标站点发起 GET 请求。

要使用模型，在界面的密码框填入 DeepSeek Key，或在启动前设置环境变量 `DEEPSEEK_API_KEY`，再勾选“同时用大模型研判”。模型调用失败时仍保留静态分析结果。

**Windows 免安装版**：从 [Releases](https://github.com/Chen-zizheng/xss-taint-scanner/releases) 下载 `XSS-scanner-windows-x64.zip`，解压后双击 `XSS扫描器.exe`，保留同目录的 `_internal/`。如果浏览器未自动打开，可查看解压目录中的 `界面地址.txt`。

从源码打包：

```powershell
pip install pyinstaller
powershell -ExecutionPolicy Bypass -File build.ps1
```

产物为整个 `dist/XSS扫描器/` 目录，发布前按 [测试说明](TESTING.md) 做打包冒烟。

## 分析流程

```text
HTML ──► taint.analyze() ───────────────────► 静态发现：行号、片段、传播路径
  └────► extract_features() → build_prompt() → call_llm() ──► 模型疑似项（可选）
```

- **静态分析**：自写 JS 词法分析与污点传播规则，从 `location`、`cookie`、`localStorage`、`postMessage` 等来源追踪到 `innerHTML`、`document.write`、`eval`、`setAttribute` 等汇点。同一输入的结果确定，适合做回归基线。界面中的“已证实”表示规则识别到了来源到汇点的通路，不代表已经在浏览器中执行利用。
- **框架写法**：识别 Vue 的 `v-html`、Angular 的 `[innerHTML]`、React 的 `dangerouslySetInnerHTML`、Svelte 的 `{@html}` 等语法形态，以及部分路由参数、props、SSR state 来源；不是完整的框架语义分析。
- **模型研判**：将脱敏、截断后的页面特征交给 DeepSeek，结果单独列为“疑似”，供人工复核。界面在勾选后会追加调用，不以静态分析是否命中为条件。

## 评测结果

仓库有五套自建、人工标注语料，用关键词规则、污点分析、规则兜底、DeepSeek 和模型兜底做对照。前两轮留出集已用于指导规则修复，作为开发与回归数据；下面展示代码冻结后编写的第三轮留出集记录：**16 条样本，10 正、6 负**。

数据来自仓库保存的 [评测报告](eval/report.md)（2026-10-30），不是本次文档更新的新一轮模型测试。

| 后端 | 精确率 | 召回率 | F1 | TP / FP / TN / FN |
|---|---|---|---|---|
| 关键词规则 `heuristic` | 88.89% | 80.00% | 0.8421 | 8 / 1 / 5 / 2 |
| 污点分析 `taint` | 100% | 70.00% | 0.8235 | 7 / 0 / 6 / 3 |
| 污点 + 关键词兜底 `hybrid` | 90.00% | 90.00% | 0.9000 | 9 / 1 / 5 / 1 |
| DeepSeek 单独分析 `deepseek` | 100% | 70.00% | 0.8235 | 7 / 0 / 6 / 3 |
| 污点 + 模型兜底 `llm_union` | 100% | 90.00% | 0.9474 | 9 / 0 / 6 / 1 |

这次记录中，模型兜底补回了静态分析漏掉的 2 条，未新增误报；与污点分析相比的 McNemar 双侧精确检验 **p=0.5000**。样本量较小，这说明了本轮样本上的补充作用，还不能据此认定普遍优于静态分析。完整语料、置信区间和错误清单见 [基准测试](BENCHMARK.md)。

评测里的 `llm_union` 是**静态无发现才调用模型**，与界面“始终可追加模型研判”的流程不同，不能直接当作界面完整流程的指标。模型输出也会随运行变化。

```bash
python eval.py                                      # 三个离线后端 × 五套语料
python eval.py --backends deepseek,llm_union          # 使用模型缓存；缺缓存时需 Key 和网络
python eval.py --backends deepseek,llm_union --no-cache # 重新调用模型，会产生费用
```

评测会覆盖 `eval/report.md` 与 `eval/bench.json`，需要保留历史记录时请先备份。语料标注规则见 [eval/README.md](eval/README.md)。Semgrep 适配入口已提供，但当前没有可用的对照成绩。

## 数据与本地服务

模型请求发往 `https://api.deepseek.com/v1/chat/completions`。提示词包含来源标识、表单结构、脚本正文、内联事件和危险函数特征。处理逻辑在 `scanner_core.py`：

- `redact_secrets()` 替换形态匹配的密码、token、API Key 等；这是规则脱敏，不保证识别所有敏感内容。
- 单个脚本正文上限为 `MAX_SCRIPT_CHARS=1500`，超长会截断并标注。
- 提示词总长上限为 `MAX_PROMPT_CHARS=20000`，超预算直接拒绝调用。

可先用 dry-run 检查实际请求内容；仍需在界面勾选模型研判来触发这条路径：

```powershell
$env:SCANNER_DRY_RUN="1"; python webapp.py
```

此时不请求模型，payload 写入 `output/payload_*.json`。检查完后取消该环境变量再正常使用。Key 从界面传到本地服务后用于调用，不写入报告或 localStorage；也可通过环境变量提供。

本地服务只监听 `127.0.0.1`，启动时生成访问 token，并校验请求中的 Origin。网址抓取只允许 http/https 和 80/443/8080/8443 端口，拒绝解析到非公网 IP 的地址，最多跟随 3 次跳转并逐次检查，设置 10 秒超时与 2 MB 正文上限。

## 已知边界

- 分析不执行 JavaScript，不覆盖完整跨函数、跨脚本与框架运行时语义。第三轮静态漏报为 `ho3-003`、`ho3-006`、`ho3-007`；模型兜底仍漏掉 Svelte `$page` 来源的 `ho3-006`。
- 网址抓取拿到的是服务端返回的 HTML，不是浏览器执行后的 DOM；纯前端渲染、登录与反爬页面可能需要另存 HTML 后分析。
- 域名检查与实际连接之间存在再次解析的窗口，不能抵御恶意 DNS rebinding；只对有权访问的页面使用抓取功能。

## 测试与项目结构

```bash
python -m pytest tests -q -p no:cacheprovider
```

2026-11-15 本地验证：Python 3.13.13 / pytest 9.1.1，**129 passed**。用例覆盖特征提取、污点规则、模型请求与解析、指标计算、本地 HTTP 校验及启动参数；模型请求在测试中被替换，不需要真实 Key。

仓库历史覆盖率记录为 **80.8%**，仅统计 `scanner_core.py`、`taint.py`、`webui.py`、`webapp.py` 四个模块，不包含评测代码，也不是本次重新测得的覆盖率。分层设计与复跑命令见 [TESTING.md](TESTING.md)。CI 在 Python 3.12 上运行测试并输出这四个模块的覆盖率。

```text
scanner_core.py    # 特征提取、脱敏截断、提示词与模型调用
taint.py           # 静态污点分析与 CLI
webapp.py          # 本地服务启动入口
webui.py           # HTTP 处理、网址抓取、分析编排
page.html          # 网页界面
eval.py            # 多后端评测、指标与报告生成
eval/              # 五套语料、标注规范、保存的评测结果
tests/             # 129 条自动化用例
docs/              # 设计决策与截图
build.ps1          # Windows 打包脚本
```

MIT，见 [LICENSE](LICENSE)。
