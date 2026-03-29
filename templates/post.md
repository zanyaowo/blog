---
title: 文章範例與格式說明
published: 2024-01-01
updated: 2024-06-01
description: "這是一篇文章範例，說明所有可用的 front-matter 欄位與 Markdown 語法。"
cover: "./cover.jpg"
coverInContent: false
pinned: false
tags: [範例, 參考]
category:
    - 參考:
        - 格式說明
draft: true
---

# Front-matter 欄位說明

每篇文章的最上方需填寫 front-matter，格式如下：

```yaml
---
title: 文章標題
published: 2024-01-01         # 發布日期
updated: 2024-06-01           # 最後更新日期（選填）
description: "文章摘要"        # 顯示在文章列表的描述（選填）
cover: ./cover.jpg            # 封面圖片路徑（選填）
                              #   ./cover.jpg → 相對於 .md 檔案的路徑
                              #   /assets/images/cover.jpg → 相對於 public/
                              #   https://... → 外部圖片網址
coverInContent: false         # 是否在文章內容中也顯示封面圖（預設 false）
pinned: false                 # 是否置頂（預設 false）
tags: [標籤1, 標籤2]          # 標籤（選填）
category: 分類名稱            # 單一分類寫法
# category: [分類, 子分類]    # 多層分類寫法
# category:                  # 巢狀分類寫法
#     - 父分類:
#         - 子分類
comment: true                 # 是否啟用留言（需在 twilight.config.yaml 開啟留言功能）
draft: false                  # 草稿模式，true 時不會公開顯示
author: "作者名稱"             # 作者（選填，預設使用 profile.name）
licenseName: "CC BY-NC-SA 4.0" # 授權條款名稱（選填）
sourceLink: "https://..."     # 原始來源連結（選填）
---
```

---

# 文章存放位置

所有文章放在 `src/content/posts/` 目錄下，支援子資料夾：

```
src/content/posts/
├── 單一文章.md
└── 帶封面的文章/
    ├── cover.jpg
    └── index.md
```

---

# 常用 Markdown 語法

## 標題

```markdown
# H1 標題
## H2 標題
### H3 標題
```

## 粗體與斜體

```markdown
**粗體文字**
*斜體文字*
~~刪除線~~
```

## 清單

```markdown
- 無序清單項目
- 項目二

1. 有序清單
2. 項目二
```

## 連結與圖片

```markdown
[連結文字](https://example.com)
![圖片描述](./image.jpg)
```

## 引言

```markdown
> 這是引言文字
```

## 程式碼

行內程式碼：\`console.log("hello")\`

程式碼區塊（支援語法高亮）：

```rust
fn main() {
    println!("Hello, world!");
}
```

---

# 特殊元件

## GitHub 卡片

```markdown
::github{repo="帳號/repo名稱"}
```

## 提示框

```markdown
:::note
這是一個備註提示框
:::

:::tip
這是一個技巧提示框
:::

:::warning
這是一個警告提示框
:::

:::caution
這是一個注意提示框
:::

:::important
這是一個重要提示框
:::
```

## 音樂卡片

```markdown
:::music{server="netease" type="song" id="歌曲ID"}
:::
```
