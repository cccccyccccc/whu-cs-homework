---
name: whu-cs-homework
description: Automate homework completion and submission on Wuhan University CS Platform (cslabcg.whu.edu.cn). Use this skill whenever the user asks to complete, write, or submit homework, assignments, or problems on the WHU CS platform — including programming problems (CodeMirror) and short-answer questions (TinyMCE). Also use when the user mentions specific course assignments, asks to check homework status, or wants to switch courses and work on assignments. This skill should be triggered even if the user only mentions a course name or assignment name from WHU CS.
---

# WHU CS Homework Automation

Automates completing and submitting homework on the Wuhan University Computer Science platform (cslabcg.whu.edu.cn) using local Chrome browser automation via CDP (Chrome DevTools Protocol). No external MCP server required.

## Prerequisites

Before first use, install the WebSocket dependency:

```bash
npm install --prefix ../browser ws
```

All browser scripts are at:
```
../browser/scripts/
```

Available scripts:
- `start.cjs` — Launch Chrome with remote debugging on port 9222
- `nav.cjs <url>` — Navigate to URL in current tab
- `eval.cjs <js-code>` — Execute JavaScript in the browser page context
- `screenshot.cjs` — Capture a screenshot of the current page

## Login & Session Management

The platform uses WHU CAS (Central Authentication Service) for login. The user MUST manually log in once — this skill does NOT handle credentials.

**To start a session:**
```bash
node "../browser/scripts/start.cjs" --profile
```
The `--profile` flag uses a persistent Chrome profile at `~/.chrome-debug-profile`, which preserves cookies and session tokens across restarts.

**Login flow:**
1. Navigate to `https://cslabcg.whu.edu.cn/`
2. Click the "校园统一认证登录" button: `document.querySelector("a#cgstuloginbtn").click()`
3. The user manually enters credentials on the CAS page and completes any captcha
4. After successful CAS login, the browser is redirected back to the course platform
5. The session is cached — subsequent visits with `--profile` skip login entirely

**IMPORTANT:** If Chrome was closed, always re-launch with `--profile` before navigating. If navigation returns empty errors, Chrome needs to be restarted.

## Platform Navigation

### Switch Course
Navigate directly to the course by its ID:
```
https://cslabcg.whu.edu.cn/courselist.jsp?courseID=<ID>
```

### Open Online Homework
```
https://cslabcg.whu.edu.cn/includes/redirect.jsp?tab=-2
```

### View an Assignment
```
https://cslabcg.whu.edu.cn/assignment/index.jsp?courseID=<courseID>&assignID=<assignID>
```

The assignment page shows:
- Left sidebar: "当前作业" (current) and "历史作业" (historical) with due dates and status
- Main area: problem list with table headers "#", "题目 点击题目标题，进入答题", "分值", "提交/评阅状态"
- Each problem row shows: problem number, name (linked to the answering page), score value, and submission status

### Understanding Assignment Status
- "未提交答案" — not yet submitted
- "还未提交代码" — code not submitted
- Shows submission time and score if already submitted
- "补交中" — late submission (may have score penalty)

## Two Types of Assignments

### Type 1: Programming Problems (编程题)

The problem links point to `programList.jsp` which redirects to `programList_ce.jsp`.

**Editor:** CodeMirror (NOT a plain textarea)
- The underlying textarea is `#cgsoucecode` (name="cgsoucecode")
- The CodeMirror instance is accessible via `document.querySelector(".CodeMirror").CodeMirror`
- Language selector: `#language` with options `c`, `c++`, `java`, `python`

**Submission form:** `#uploadFORM` (POST to `showProcessMsg.jsp`)
- Hidden fields: `doSubmit=true`, `byCE=true`, `progLanguage`, `problemID`, `assignID`, `wtime`
- Submit button: `#cgSubmitBtn` with onclick `cgsrcSubmit()`

**Judging results:** Displayed in the `#showmessageFRAME` iframe on the same page after submission. Shows compile errors, test case results, memory/CPU usage.

**Code submission gotcha — shell escaping:**
When passing C++ code through shell commands, backslashes in escape sequences (`\n`) get mangled. ALWAYS use Base64 encoding:

1. Encode the code as base64 first (use a temporary approach — write to file or pipe)
2. Set via eval.cjs using `atob()` decode:
```javascript
const code = atob("<base64-encoded-code>");
document.getElementById("cgsoucecode").value = code;
document.querySelector(".CodeMirror").CodeMirror.setValue(code);
```

**Reading judge results after submission:**
Wait a few seconds, then check the iframe:
```javascript
document.getElementById("showmessageFRAME").contentWindow.document.body.innerText
```
If there are compile errors or test failures, analyze the feedback, fix the code, and resubmit. Continue iterating until all test cases pass and the score is full marks.

### Type 2: Short Answer Questions (简答题)

The problem links point to `briefAnswerList.jsp`.

**Editor:** TinyMCE (rich text editor)
- The underlying textarea is `#tinyContent` (name="answer")
- Use TinyMCE API: `tinyMCE.get("tinyContent").setContent(content)`
- Submit button: `#brSubmitbtn`

**Formatting rules for short answers:**
- Use **plain text only** — no Markdown rendering, no bold, no italics, no lists
- The ONLY exception is **tables**, which are allowed
- For line breaks, use `<br>` HTML tags (NOT `\n`, NOT `<p>` tags)
- Do NOT use `<ul>`, `<ol>`, `<li>`, `<strong>`, `<em>`, `<h1>`-`<h6>`, or any other HTML formatting tags

**Example of correct formatting:**
```
TCP的拥塞控制包括以下几个阶段：<br><br>1. 慢启动阶段：cwnd从1个MSS开始，每经过一个RTT翻倍。<br>2. 拥塞避免阶段：cwnd线性增长，每RTT增加1个MSS。<br>3. 快速重传：收到3个重复ACK时立即重传丢失的段。<br>4. 快速恢复：将ssthresh设为cwnd的一半，cwnd减半后进入拥塞避免。
```

**Example of WRONG formatting:**
```
<p><strong>TCP的拥塞控制</strong></p><ul><li>慢启动</li><li>拥塞避免</li></ul>
```

## Core Workflow

### Step 1: Ensure Chrome is running
```bash
node "../browser/scripts/start.cjs" --profile
```

### Step 2: Navigate to platform and verify session
```bash
node "../browser/scripts/nav.cjs" "https://cslabcg.whu.edu.cn/"
node "../browser/scripts/eval.cjs" "document.title"
```
If the page shows the course list, the session is active and you can proceed directly to Step 3.

If the page shows an empty title or the login page (not the course list), the cached session has expired. Trigger the CAS unified auth flow:
```bash
node "../browser/scripts/eval.cjs" 'document.querySelector("a#cgstuloginbtn").click()'
```
This redirects to `cas.whu.edu.cn`. If the CAS session is also cached, it auto-redirects back to the course platform. Otherwise, tell the user they need to manually complete the CAS login (enter credentials and captcha) — the skill cannot handle credentials.

### Step 3: Discover available courses and assignments
ALWAYS inspect the page before acting. Ask the user which course and which assignment they want to work on rather than guessing.

To get the course list:
```bash
node "../browser/scripts/eval.cjs" '(()=>{const links=document.querySelectorAll("a");return Array.from(links).filter(l=>l.href.includes("courseID")).map(l=>({text:l.textContent.trim(),courseID:new URLSearchParams(l.href.split("?")[1]).get("courseID")}));})()'
```

### Step 4: Navigate to the specified course and homework
Switch course → go to homework tab → enter assignment → read problems.

### Step 5: For each problem, read the question first
Never submit without understanding the problem. Read the problem description carefully.

### Step 6: Write the answer/code
For programming: write correct code, handle edge cases, match I/O format exactly.
For short answer: use plain text with `<br>` line breaks.

### Step 7: Submit only when explicitly instructed
**CRITICAL RULE:** Never click submit without the user's explicit instruction for each problem. Always present the answer/code to the user first if there's any uncertainty. The user must explicitly say "submit" or equivalent.

### Step 8: Check results and iterate
After submission, check the results. For programming problems:
- Compile errors → fix the code and resubmit
- Wrong answer on some test cases → analyze edge cases, fix, resubmit
- All correct → move to next problem

## Constraints & Safety Rules

1. **NO automatic submission.** The user must explicitly instruct submission for each assignment or problem. When in doubt, ask.
2. **NO guessing courses.** Always list available courses/assignments and let the user specify which one to work on.
3. **Only plain text for short answers.** No HTML formatting tags except `<br>` and tables. Explain this constraint to the user if they ask for formatted text.
4. **Read before writing.** Always read the problem description before attempting to answer.
5. **Iterate on programming problems.** Use the judge feedback (in showmessageFRAME) to fix and improve code until full marks.
6. **Preserve the browser session.** Always use `--profile` flag when starting Chrome. The session at `~/.chrome-debug-profile` persists cookies and auth tokens.

## Common Errors & Fixes

**"Error: Cannot find module 'ws'"**
→ Run `npm install --prefix ../browser ws`

**Navigation returns empty error**
→ Chrome was closed. Restart with `start.cjs --profile`

**Code compilation fails with escape sequence errors**
→ Shell escaping issue. Use Base64 encoding for code with backslash escapes.

**TinyMCE content not updating**
→ Use `tinyMCE.get("tinyContent").setContent(content)` instead of setting textarea value directly.

**CodeMirror content not updating**
→ Use `document.querySelector(".CodeMirror").CodeMirror.setValue(code)` to sync with the editor instance.
