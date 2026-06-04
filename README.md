# whu-cs-homework

武汉大学计算机学院专业课一体化平台 (cslabcg.whu.edu.cn) 自动写作业 Skill。

## 功能

- 自动操作 Chrome 浏览器，登录 WHU CS 平台
- 支持**编程题**：C/C++/Java/Python 代码自动编写、提交、根据判题日志优化
- 支持**简答题**：纯文本答案自动填写提交
- 严格遵循用户指令，绝不自动盲目提交

## 依赖

```bash
npm install --prefix ~/.claude/skills/browser ws
```

## 使用方式

将此 skill 安装到 Claude Code 的 skills 目录下：

```
~/.claude/skills/whu-cs-homework/
```

## 前置条件

1. 安装 Chrome 浏览器
2. 手动登录一次 https://cslabcg.whu.edu.cn/，让浏览器缓存 CAS 认证 session

## 支持的作业类型

| 类型 | 编辑器 | 提交方式 |
|------|--------|----------|
| 编程题 (programList_ce.jsp) | CodeMirror | `#cgSubmitBtn` → `cgsrcSubmit()` |
| 简答题 (briefAnswerList.jsp) | TinyMCE | `#brSubmitbtn` |

## 注意事项

- 简答题只允许纯文本 + `<br>` 换行，禁止 markdown
- 编程题代码通过 Base64 编码传递，避免 shell 转义
- 每次必须用 `--profile` 启动 Chrome 以保持登录态
