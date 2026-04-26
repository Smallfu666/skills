# skills

放置你自己的 Copilot skills。

## 結構

每個 skill 放在自己的資料夾，且資料夾內至少要有一個 `SKILL.md`。

```text
skills/
├── README.md
└── my-skill/
    └── SKILL.md
```

```text
skills/
├── cluster-info/
│   └── SKILL.md
```

## 安裝方式

之後可以用這種方式安裝單一 skill：

```bash
npx skills@latest add <你的 GitHub 帳號>/skills/my-skill
```

## 新增 skill

1. 建立新的資料夾。
2. 在裡面放 `SKILL.md`。
3. 在 `SKILL.md` 內寫好 frontmatter：

```md
---
name: my-skill
description: 這個 skill 做什麼。Use when ...。
---

# My Skill
```
