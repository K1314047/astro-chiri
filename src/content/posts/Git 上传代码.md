---
title: Git 上传代码
pubDate: '2025-08-18'
---

## 如果你想确认当前状态，可以执行：

```
git status
```

- 如果显示 `working tree clean`，说明没有未提交的更改。
- 如果有修改但没提交，会提示你先 `git add` 和 `git commit`。

------

## 如果你想推送新的代码，步骤是：

1. 修改或新增文件
2. 执行 `git add .`
3. 执行 `git commit -m "你的提交信息"`
4. 执行 `git push`