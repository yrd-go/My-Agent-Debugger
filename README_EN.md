# My-Agent-Debugger (Agent Toolchain Troubleshooting & Hybrid Workflow Field Notes)

## 💡 Background

This project is a field record of a software engineering student putting an **AI agent into real production use**. While assembling a local toolchain on top of DeepSeek Harness, I ran into genuine engineering friction: sandbox isolation, LLM code hallucination, and Windows encoding conflicts.

By analyzing agent traces, running multi-model A/B comparisons, and adopting a **"generation/execution decoupling"** hybrid workflow, I eventually closed the loop from Node.js all the way to a working Python database query.

This repository documents the complete diagnostic reasoning, comparison screenshots, and code evolution. The goal is to share a practical pattern for the AI era: **AI-assisted generation combined with human engineering review**.

> **TL;DR** — Use a strong reasoning model on the critical path. Let the agent write code, but keep execution behind a human-verified boundary. When an agent and a database fail to talk, suspect the character set before you suspect the logic.

## 🚀 Quick Start & Reproduction Guide

### Prerequisites

* Node.js >= 22.5.0 (native `node:sqlite` support)
* Python >= 3.10
* API keys for the relevant models (DeepSeek / Zhipu GLM, etc.)
* The `test.db` database in this repository is generated automatically by `init_db.js` — run the initialization step first.

### Running the Demo Scripts

1. **Node.js version (database initialization and query):**

   ```bash
   node init_db.js
   node query_student.js 2
   ```

2. **Python version (argument validation and exception handling):**

   ```bash
   python query_student.py 2
   python query_student.py abc
   python query_student.py
   ```

*Note: the repository root contains mixed Node.js and Python verification scripts, corresponding to different stages of the troubleshooting experiments.*

## 🔍 Field Notes: Pitfalls and Resolutions

### 1. Sandbox Environment Restrictions and Tool Refactoring

* **Symptom**: The Harness sandbox blocked external invocation of the Python interpreter, failing with `file access denied`.
* **Resolution**: Rather than fighting the sandbox, I worked with its security model and refactored the tool onto Node.js built-in modules — preserving both safety and execution efficiency.

  ![Sandbox denial](01-sandbox-denied.png?raw=true)

### 2. Lightweight-Model Code Hallucination and Failed Self-Correction

* **Symptom**: When generating a Node.js script, Zhipu GLM-4-Flash incorrectly applied object destructuring to a string (`const { id } = '1002'`). This silently turned the parameter into `undefined`, failed the task, and produced misleading output (a false "Not found").
* **Symptom**: When asked to self-correct, the model could neither identify the logical error nor recover from a port conflict and sandbox-induced module isolation (`Cannot find module`) — it abandoned the task entirely.
* **Resolution**: Reading the **trace log** pinpointed the corrupted parameter precisely. Swapping in DeepSeek ran the task successfully on the first attempt, establishing the policy that **a strong reasoning model is mandatory on the critical path**.

  ![Model hallucination](02-model-hallucination-trace.png?raw=true)
  ![Self-correction failure](03-self-correction-failure.png?raw=true)

### 3. Strong-Reasoning-Model Repair and Character-Set Debugging (IPC Encoding Troubleshooting)

* **Symptom**: DeepSeek fixed the code precisely, but the first request returned mojibake (`aæ...`) caused by PowerShell's default decoding.
* **Resolution**: The model independently diagnosed the root cause — the `Content-Type: text/plain` header did not declare a character set. It then added `charset=utf-8` and restarted the service, successfully returning "李四" (Li Si).
* **Bonus**: On completion, the agent proactively issued `job_kill` to clean up the background process — demonstrating sound resource lifecycle management.

  ![DeepSeek success](04-deepseek-success-trace.png?raw=true)
  ![Encoding fix](05-deepseek-fix-and-encoding.png?raw=true)
  ![Charset success](06-charset-utf8-success.png?raw=true)

**Why this class of bug is worth calling out:** an inter-process communication (IPC) boundary has *two* independent decoders — the producer's declared content type and the consumer's console codepage. The bytes on the wire were always valid UTF-8; the failure was entirely in metadata. Declaring `charset=utf-8` on the response is the durable fix, because it makes the contract explicit instead of relying on the client to guess.

### 4. External API Integration and Compliance Approval

* **Symptom**: Building a GitHub issue-scraping script required translating English issue titles into Chinese.
* **Resolution**: The agent strictly followed the rules in `AGENTS.md` — it output a plan and only wrote files after receiving human approval. It also recognized that Google Translate is unreliable behind the domestic network, and successfully integrated the free MyMemory translation API instead.

  ![Plan approval](07-github-issues-plan-approval.png?raw=true)
  ![External translation API](08-script-generation-summary.png?raw=true)

### 5. Database Dependency Risk and a Zero-Dependency Rewrite

* **Symptom**: To create the SQLite database, the agent initially planned to use `better-sqlite3` — a native C++ module with a real risk of compilation failure on Windows.
* **Resolution**: I intervened decisively and asked for a different approach. The agent immediately checked the Node version (v24.21.0) and switched to the built-in `node:sqlite` module. It went on to create the table, insert data, and query the student with `id=2` as "李四" — achieving a **zero-external-dependency deployment**.

  ![Database selection](09-database-sqlite-setup.png?raw=true)
  ![Execution and verification](10-sqlite-execution-and-verification.png?raw=true)

### 6. Generation/Execution Decoupling (Closing the Hybrid Workflow Loop)

* **Symptom**: Because of Harness sandbox isolation, the agent could not directly invoke the local Python interpreter to run scripts. At the same time, the VS Code static checker flagged the built-in `sqlite3` and `os` modules with red squiggles, since it had not detected the local interpreter.
* **Resolution**: I adopted a hybrid workflow that **decouples generation from execution**. The agent focused purely on generating code; I manually pulled that code into local VS Code and ran `python query_student.py` in the terminal, successfully printing `the student name for id 2 is: 李四`. This verified the correctness of the code logic while preserving the system's security boundary.

  ![Agent self-correction](11-agent-self-correction-trace.png?raw=true)
  ![Local execution verification](12-local-execution-verification.png?raw=true)

**The trade-off, stated plainly:** decoupling costs one manual copy-and-run step, and in exchange the agent never gains arbitrary code execution on my machine. For a toolchain that an LLM is actively writing, that is the right side of the trade.

### 7. Boundary Testing and a Closed Argument-Validation Loop

* **Symptom**: To make the script genuinely usable, I asked the agent to add command-line argument validation to `query_student.py`, including friendly error messages.
* **Resolution**: The agent independently introduced `sys.argv` and a `parse_id` function, then proactively ran four boundary tests (invalid ID, normal query, non-numeric input, no argument). All four passed — confirming that the engineering standards I set in `AGENTS.md` (approval before execution, parameterized queries against SQL injection) were fully enforced.

  ![Argument validation loop](13-param-validation-success.png?raw=true)

### 8. Context Engineering Retrospective (Decoupling Meta-Rules from Task-Rules)

* **Retrospective**: I originally wrote code-specific constraints directly into `AGENTS.md` (for example, "accept parameters via `sys.argv`"). This caused context pollution and hallucination when the AI later moved on to cross-domain tasks.
* **Resolution**: I restructured the system into a **"meta-rules + task-rules"** separation. `AGENTS.md` now holds only global safety and interaction baselines (such as "ask before acting"), while concrete technical constraints live in each conversation's user prompt. This keeps the system context clean and substantially improves the model's instruction-following accuracy.

**Rule of thumb:** anything that is true for *every* task belongs in `AGENTS.md`; anything true for *this* task belongs in the prompt. Mixing the two is how a global rule file turns into a hallucination source.

### 9. Multi-Language Adaptive Translation Routing

* **Symptom**: The original translation router hard-coded `hello` and supported only Chinese-English translation, leaving long-tail needs such as Japanese, Korean, and Russian unhandled.
* **Resolution**:
  1. Replaced `re.search` with `re.finditer` to strip suffixes like "to Japanese" / "to Russian" precisely, avoiding false positives on complex phrasing (for example, "the content translated to Japanese").
  2. Built a `LANG_NAME_TO_CODE` mapping table that tolerates synonyms such as "Japanese / Japanese-language / Chinese".
  3. Implemented **adaptive bidirectional translation** (Chinese ↔ English) plus **directed multi-language routing** (Chinese → Japanese / Russian / Korean).
  4. Added **same-language protection** (short-circuiting when source language equals target language, so no useless request is sent) and empty-content boundary handling.
  5. Ran seven real test cases in local VS Code (including empty input, mixed Chinese-English, and multi-language cases) — all passed.

  ![Multi-language router verification](14-multi-language-router-final.png?raw=true)

### 10. Smooth Database Evolution (SQLite → MongoDB)

* **Symptom**: As the agent toolchain grew more complex, it needed to store dynamic JSON structures and unstructured logs. The rigid table schemas of relational databases (SQLite / MySQL) became cumbersome.
* **Resolution**: I led a database selection change, migrating from SQLite to the document-oriented **MongoDB**:
  1. Rewrote `init_db.py` and `query_student.py` using the `pymongo` library.
  2. Read connection details strictly from the `MONGO_URI` environment variable — no hard-coding.
  3. Stored data as JSON documents (`{"id": 1, "name": "张三"}`), matching the mutable nature of AI-domain data structures.
  4. Ran it for real in local VS Code: created the database, inserted three records, and precisely queried "李四" for `id=2`.

  ![MongoDB migration success](15-mongodb-migration-success.png?raw=true)

## 🚀 Key Takeaways

1. Gained a working method for locating AI tool-call failures through **trace analysis** rather than guessing at prompts.
2. Experienced and understood why local sandbox isolation and the file-system observation policy (`FS_NOT_OBSERVED`) matter for agent safety.
3. Mastered the **"AI generation + local IDE verification"** hybrid workflow, achieving safe decoupling of generation from execution.
4. Developed the practical ability to perform **engineering review of AI-generated code** — SQL injection prevention, resource release, and exception handling.

## 🧭 Design Principles Behind the Pitfalls

| Principle | Concrete manifestation in this repo |
| --- | --- |
| **Multi-Model Routing** | GLM-4-Flash failed on the critical path (string destructuring hallucination); DeepSeek fixed it on the first attempt. Lightweight models can handle peripheral work, but the critical path requires a strong reasoning model. |
| **Environment Decoupling** | The sandbox denies local interpreter invocation. The agent generates; the human executes in local VS Code. Capability gained without granting arbitrary code execution. |
| **IPC Encoding Troubleshooting** | Mojibake traced to an undeclared charset on `Content-Type: text/plain`, fixed with `charset=utf-8` plus a service restart. Always make the encoding contract explicit at the boundary. |
| **Meta-Rules vs. Task-Rules** | `AGENTS.md` holds only global safety and interaction baselines; task-specific constraints stay in the prompt. Separation keeps context clean and raises instruction-following accuracy. |
| **Zero-Dependency Bias** | Choosing `node:sqlite` over `better-sqlite3` removed a native C++ compilation risk on Windows in one decision. |

## 💻 Tech Stack

* Python (AI-assisted generation and debugging)
* Node.js (built-in module development)
* LLM APIs (DeepSeek / GLM-4-Flash)
* MongoDB / SQLite (`node:sqlite`, `pymongo`)

## 📈 Roadmap

- [ ] Automatically capture error stacks across languages (Node.js, Python).
- [ ] Implement **multi-model routing**: route simple errors to a lightweight model and complex failures to a strong model, optimizing cost without sacrificing success rate.
- [ ] Extend the generation/execution decoupling boundary with a reviewer step that diffs agent output before it reaches local execution.
