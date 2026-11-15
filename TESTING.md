# 测试设计说明

这份文档记录自动化测试的分层、用例设计、历史覆盖率与发布冒烟步骤。
2026-11-15 使用 Python 3.13.13 / pytest 9.1.1 运行现有测试，结果为 129 passed（7.69 秒）；覆盖率与模型评测沿用仓库历史记录，本次未重新测量。命令见第 7 节。

## 1. 测试分层

| 层 | 对象 | 证明什么 | 规模 |
|---|---|---|---|
| 单元 | `scanner_core` / `taint` | 每个判断规则在边界处是否正确 | 77 例 |
| 接口 | `webui` 的 HTTP 处理、`webapp` 的启动参数 | 鉴权、限额、异常路径是否按约定拒绝 | 25 例 |
| 评测 | `eval.py` 的指标、区间与检验 | 数字本身算得对不对 | 27 例 |
| 数据集回归 | 五套语料 × 五个后端 | 对比检测能力与逐条误报、漏报 | 完整组合 25 行指标；默认三个离线后端为 15 行 |
| 发布冒烟 | 打包后的 exe | 别人下载下来双击能不能用 | 手动脚本，见第 6 节 |

前三层在 `pytest` 里（本次 129 passed），第四层由 `eval.py` 输出指标表。
第五层不在自动化里，需要对实际打包产物验证启动、地址可发现性和扫描流程。

## 2. 历史覆盖率记录

以下为旧文档保存的四模块覆盖率快照，不是本次测试的新覆盖率，也不代表整个仓库的覆盖率。当前代码的结果请用第 7 节命令重新生成。

```
scanner_core.py    95 语句    0 未覆盖   100.0%
webapp.py          16 语句    1 未覆盖    93.8%
taint.py          439 语句   76 未覆盖    82.7%
webui.py          242 语句   75 未覆盖    69.0%
合计              792 语句  152 未覆盖    80.8%
```

怎么读这几个数字：

- `webui.py` 缺的 75 行集中在抓网址的异常分支（跳转超限、非 HTML、空壳页面）、浏览器打开失败
  与 HTTP 服务的收尾。界面交互没有纳入当前自动化测试，发布时按第 6 节手动验证启动、浏览器访问与扫描结果。
- `webapp.py` 少的那 1 行是 `__main__` 里的 `raise SystemExit`。
- `eval/` 与 `eval.py` 不在这次历史覆盖率的统计范围中；指标计算、bootstrap、McNemar 等逻辑另由 `tests/test_eval.py` 的 27 条用例检查。输出报告本身不能独立证明评测脚本正确。

当前没有设置覆盖率门槛。行覆盖用于定位未执行的分支，不能替代对断言质量和真实使用流程的检查。

## 3. 用例怎么设计的

| 方法 | 在这个仓库里的落点 |
|---|---|
| 等价类划分 | 语料按「有 sink 有 source / 有 sink 无 source / 净化后 / 框架语法」分类，每类都有正负样本 |
| 边界值 | 2 MB 输入（正好 2 MB / 超 1 字节）、单脚本 1500 字符、提示词 20000 字符、跳转 3 次、key 200 字符、POST 4 MB |
| 判定表 | `webui` 的请求校验：Origin 有无 × token 对错 × Content-Length 是否合法 × JSON 是否可解析 |
| 状态迁移 | 扫描流程：无 key → 勾选禁用 → 填 key → 可勾选 → 勾选后调用失败 → 静态结论仍在 |
| 错误推测 | 模型返回围栏 JSON / 裸 JSON / 散文 / 空对象 / 429；报告字段里塞换行试图伪造标题 |
| 组合 | 五个后端 × 五套语料，指标表就是组合测试的结果 |
| 回归 | 每轮语料冻结后不再改；发现漏报 → 修规则 → 用**新一轮**语料验证（旧轮次降级为开发证据） |

## 4. 语料即测试数据

`eval/*.jsonl` 每条形如：

```json
{"id": "ho3-001", "html": "<div v-html=\"$route.params.slug\"></div>",
 "label": {"has_xss_risk": true, "risk_level": "HIGH"}, "note": "Vue v-html 绑定路由参数"}
```

- 标注规则写在 [eval/README.md](eval/README.md)：什么算 XSS 风险、什么算净化、如何标记风险等级。
- 每套语料有用途，不混用：`dataset` 开发、`corpus-hard` 难例、`holdout/2/3` 三轮留出。
- **留出集的规矩**：写完之后冻结；改代码不能回头改语料；哪一轮被用来指导修复，哪一轮就不再是留出集。
  当前主要参考第三轮的 16 条自建样本（10 正、6 负），前两轮作为开发与回归证据；小样本结果不代表真实站点上的总体检测率。
- 评测 `llm_union` 只在静态无发现时调用模型；界面勾选后会追加模型研判，两者不是同一条完整流程。

## 5. 缺陷记录

按「现象 → 根因 → 修复 → 回归用例」记录。每条修复都挂一个用例；把修复回退，对应用例变红。

| ID | 严重级 | 现象 | 根因 | 修复 | 回归用例 |
|---|---|---|---|---|---|
| D-1 | 中 | 页面里写死的密码跟着脚本正文进了第三方提示词 | `prepare_features` 只截断不脱敏 | `redact_secrets()` + 三道闸 | `tests/test_llm_pipeline.py` 脱敏两组、`prepare_features` 不改入参 |
| D-2 | 低 | 8 MB 页面能撑出 233 万 token 的提示词 | 输入与提示词都没有上限 | `MAX_INPUT_BYTES=2MB`、`MAX_PROMPT_CHARS=20000` | 超大 HTML 返回 413、超预算抛 `ValueError` |
| D-3 | 待核实 | 旧记录称打包 exe 双击退出，未保留可核对的异常栈 | 旧文档归因于 stdout 为空；Python 3.12 中 `sys.stdout=None` 时 `print()` 本身不必报错，该归因不足以成立 | 当前冻结打包分支写入 exe 同目录 `界面地址.txt`，便于无控制台时找到地址 | 第 6 节为待执行的打包冒烟步骤，不作为旧故障已复现的证据 |
| D-4 | 中 | 目标站点返回 403 或纯前端空壳时，工具给出「0 条」的假结论 | 只判 `status_code >= 400`，未处理空正文 | `_status_hint()` 按状态码给处置建议；空正文直接拒绝 | `test_fetch_403_suggests_saving_page`、`test_fetch_empty_body_rejected` |
| D-5 | 低 | 界面粘贴超长 key 时行为未定义 | 未校验请求体字段 | 非字符串或超 200 字符返回 400 | `test_key_too_long_rejected` |
| D-6 | 高 | `DOMPurify.sanitize(a) + location.hash` 这类「净化函数 + 真实污点」被判安全，真实绕过漏报 | `_taint_of` 只要区间里出现净化函数名就整段放行 | `_sanitized_spans()` 只豁免净化函数自己的实参 | `test_sanitizer_only_covers_its_own_argument`、`test_sanitizer_does_not_mask_other_taint_in_eval`、`test_expression_with_only_sanitizers_is_quiet` |
| D-7 | 中 | 网页界面的代码框永远是空的 | `_finding()` 不产出 `snippet` 字段，前端却在渲染它 | 补 `snippet` | `test_finding_carries_snippet_for_the_ui` |

按仓库历史记录，D-1、D-2 来自发布前安全检查，D-4、D-5 来自实际使用，D-6、D-7 来自评测复核。本次未重做这些历史实验。D-3 的旧根因说明已撤回，需以具体打包产物和异常日志重新排查；源码中的同类注释也不能代替复现证据。

## 6. 发布冒烟（不进 CI）

打包后手工验证启动、浏览器和地址文件行为。下面是验证步骤，本次文档更新未执行打包冒烟：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File build.ps1
cmd /c start "" "dist\XSS扫描器\XSS扫描器.exe" --no-browser --port 8943
Start-Sleep -Seconds 5
Get-Content "dist\XSS扫描器\界面地址.txt"          # 应当出现 http://127.0.0.1:8943/?t=...
```

拿到地址后：页面返回 200、页面里有 `id="key"`、POST 一个含 `location.hash → innerHTML` 的片段能出结论、
进程保持运行。四项都通过才算发布成功。

## 7. 怎么跑

```bash
python -m pytest tests -q -p no:cacheprovider      # 本次 129 passed，7.69 秒
pip install pytest-cov coverage
python -m pytest tests -q -p no:cacheprovider --cov=scanner_core --cov=taint --cov=webui --cov=webapp --cov-report=term-missing
```

CI 在 Python 3.12 上跑同一套用例，每次 push 到 main 与每个 PR 都会执行。

CI 配置可在 `.github/workflows/test.yml` 核对；本地测试结果不替代远端具体一次构建的结果。

## 8. 没测的东西，以及为什么

- **真实 DeepSeek 接口的集成测试**：不在自动化单测范围中。`SCANNER_DRY_RUN=1` 可供人工检查 payload，但不能验证真实鉴权、网络或模型行为；历史真实调用结果见评测报告。
- **网页界面的真实交互**：无头环境不稳，用第 6 节的打包冒烟兜底。
- **并发与性能**：本地单用户工具，没有并发场景；抓取有 10 秒超时与 2 MB 截断，但没有吞吐基线。
- **兼容性**：只在 Windows 11 + Python 3.13 与本机 CI（Ubuntu + 3.12）上验证过，没有覆盖更老的 Python 与其它发行版。
- **真实恶意站点**：抓网址功能只针对公开页面做过验证，没有对反爬/蜜罐站点做对抗测试。