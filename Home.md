---
cssclasses:
  - home-page
---

<div class="home-top">
  <div class="home-greeting">Delphi</div>
  <div class="home-date">今天 · 打开库第一眼，先看这三块</div>
</div>

<div class="home-grid">

<div class="glass-card">

### 📅 今天的日程

```dataview
LIST WITHOUT ID
  link(file.link, title) + " · " + date
FROM "05_Calendar"
WHERE date = date(today)
SORT date ASC
```

> 没显示？先确认 Full Calendar 里建的日程在 `05_Calendar`，且笔记 frontmatter 里有 `date`。

[打开完整日历](obsidian://open?vault=Delphi)

</div>

<div class="glass-card">

### ✅ 今天的任务

```tasks
not done
due today
```

```tasks
not done
no due date
limit 5
```

</div>

<div class="glass-card">

### 🚀 正在做的事

- [[03_Projects/README|项目]]
- [[04_Research/Paper-Drafts|论文草稿]]
- [[04_Research/Papers|文献]]

> 这里只链 3 个以内，换项目时改这三行就行。

</div>

<div class="glass-card glass-wide">

### 🕒 最近的日程

```dataview
TABLE WITHOUT ID
  link(file.link, title) AS "日程",
  date AS "日期"
FROM "05_Calendar"
WHERE date >= date(today) AND date <= date(today) + dur(7 days)
SORT date ASC
LIMIT 7
```

</div>

<div class="glass-card">

### 📚 课程

- [[02_Courses/Complex-Analysis|复变函数]]
- [[02_Courses/Data-Structures-and-Algorithms-I|数据结构与算法 I]]
- [[02_Courses/Computer-Networks|计网]]

</div>

<div class="glass-card">

### 📝 日记与记录

- [[01_Diary|今天日记]]
- [[04_Research/Experiments|实验记录]]
- [[08_Life|生活]]

</div>

</div>

<div class="home-dock">

[[02_Courses|📚 课程]] · [[04_Research/Papers|📄 文献]] · [[05_Calendar|📅 日历]] · [[01_Diary|📝 日记]] · [[03_Projects|🚀 项目]] · [[08_Resources|⚙️ 资源]]

</div>
