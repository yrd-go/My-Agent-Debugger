# My-Agent-Debugger
An AI Agent debugging toolkit based on LLM

## 💡 项目背景
本项目为个人实战项目，旨在解决轻量级模型（如 GLM-4-Flash）在 Agent 调用链中的“代码幻觉”与“容错率低”问题。通过 Trace 追踪和自动重试机制，降低 Agent 在调用外部工具（如 API、数据库）时的报错重试率。

## 🔍 实战踩坑与解决案例
在搭建本地 Agent 工具链时，我遇到了以下真实痛点，并在此仓库记录了完整的排查思路与截图：

### 1. 沙箱环境限制与工具重构
* **现象**：Harness 沙箱拦截了 Python 解释器的外部调用，报错 `file access denied`。
* **解决**：顺应沙箱的安全规则，将工具重构为 Node.js 内置环境，兼顾了安全性与执行效率。
  ![沙箱拦截](01-sandbox-denied.png?raw=true)

### 2. 轻量级模型代码幻觉与自我纠错失败
* **现象**：智谱 GLM-4-Flash 生成 Node.js 脚本时，错误地对字符串使用了对象解构赋值（`const { id } = '1002'`），导致参数变成 `undefined`，最终任务失败并产生错误引导（如误报“Not found”）。
* **现象**：让该模型尝试自我纠错时，它既无法识别逻辑错误，又因端口占用和环境模块隔离（沙箱导致的 `Cannot find module`）彻底放弃。
* **解决**：通过查看 Trace（轨迹）日志精准定位到参数异常，手动替换为 DeepSeek 后一次跑通，确立了“核心链路调用强推理模型”的策略。
  ![模型幻觉](02-model-hallucination-trace.png?raw=true)
  ![自我纠错失败](03-self-correction-failure.png?raw=true)

### 3. 强推理模型修复与字符集编码排错
* **现象**：DeepSeek 接手后精准修复了代码，但首次请求时遇到 PowerShell 默认解码导致的乱码（`aæ...`）。
* **解决**：模型主动分析出是 `Content-Type: text/plain` 未声明字符集导致，随后添加 `charset=utf-8` 并重启服务，成功获取“李四”。完成后还主动执行 `job_kill` 清理了后台进程，展现了优秀的资源生命周期管理能力。
  ![DeepSeek成功](04-deepseek-success-trace.png?raw=true)
  ![修复编码](05-deepseek-fix-and-encoding.png?raw=true)
  ![字符集成功](06-charset-utf8-success.png?raw=true)

### 4. 外部 API 集成与合规审批
* **现象**：在开发 GitHub Issue 抓取脚本时，需要将英文标题翻译为中文。
* **解决**：Agent 严格遵守 `AGENTS.md` 规则，先输出计划并获得人工批准后才写入文件。同时，它识别到国内网络环境对 Google 翻译的限制，成功集成了 MyMemory 免费翻译 API。
  ![计划审批](07-github-issues-plan-approval.png?raw=true)
  ![外接翻译API](08-script-generation-summary.png?raw=true)

### 5. 数据库依赖风险与零依赖改造
* **现象**：创建 SQLite 数据库时，Agent 原本计划使用 `better-sqlite3`，但这在 Windows 下属于原生 C++ 模块，存在编译失败风险。
* **解决**：我果断介入，要求修改方案。Agent 随即检查 Node 版本（v24.21.0），改用内置的 `node:sqlite` 模块。最终成功建表、插入数据并查询出 `id=2` 的学生为“李四”，实现了零外部依赖部署。
  ![数据库选型](09-database-sqlite-setup.png?raw=true)
  ![执行验证](10-sqlite-execution-and-verification.png?raw=true)

### 6. 生成与执行解耦（混合工作流闭环）
* **现象**：由于 Harness 沙箱的安全隔离，Agent 无法直接调用本地 Python 解释器运行脚本；同时 VS Code 静态检查器因为未识别本地解释器，对内置库 `sqlite3` 和 `os` 误报红线。
* **解决**：我采用了“生成与执行解耦”的混合工作流。让 Agent 专注生成代码，我则手动拉取代码到本地 VS Code，并在终端执行 `python query_student.py`，成功输出 `id 为 2 的学生名字是：李四`。这验证了代码逻辑的正确性，同时保证了系统的安全边界。
  ![Agent 自动修复编码](11-agent-self-correction-trace.png?raw=true)
  ![本地执行验证成功](12-local-execution-verification.png?raw=true)

## 🚀 核心功能
1. 捕获终端报错日志。
2. 拼装 Prompt 模版，调用大模型 API 获取修复建议。
3. 反馈给主 Agent 进行自我纠错。

## 💻 技术栈
* Python (AI 辅助生成与调试)
* Node.js (内置模块开发)
* 大模型 API (DeepSeek / GLM-4-Flash)

## 📈 后续规划
- [ ] 支持自动捕获不同语言（Node.js、Python）的报错堆栈。
- [ ] 实现多模型路由：简单报错用轻量模型，复杂错误路由给强模型降本增效。
