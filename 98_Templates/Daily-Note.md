---
cssclasses:
  - daily-card
---

```dataviewjs
// ===== Hero：大字日期 · 周信息 · 心情/精力胶囊（阅读视图里点击即记录，写入 frontmatter）=====
const cur = dv.current();
const dayStr = (cur.file.name.match(/\d{4}-\d{2}-\d{2}/) || [])[0];
if (!dayStr) {
  dv.paragraph("⚠️ 日记文件名需要 YYYY-MM-DD 格式");
} else {
  const day = dv.luxon.DateTime.fromISO(dayStr);
  const WEEK = ["日", "一", "二", "三", "四", "五", "六"];
  const state = {
    mood: typeof cur.mood === "number" ? cur.mood : null,
    energy: typeof cur.energy === "number" ? cur.energy : null,
  };

  const hero = dv.container.createDiv({ cls: "daily-hero" });
  const left = hero.createDiv({ cls: "daily-hero-left" });
  left.createDiv({ cls: "daily-hero-date", text: `${day.month}月${day.day}日` });
  left.createDiv({ cls: "daily-hero-sub", text: `星期${WEEK[day.weekday % 7]} · ${day.month}月第${Math.ceil(day.day / 7)}周 · 今年第${day.ordinal}天` });

  const meters = hero.createDiv({ cls: "daily-hero-meters" });
  const moodBox = meters.createDiv({ cls: "daily-meter" });
  moodBox.createSpan({ cls: "daily-meter-label", text: "心情" });
  const moodRow = moodBox.createDiv({ cls: "daily-meter-row" });
  const MOODS = ["😩", "😕", "😐", "🙂", "😄"];
  MOODS.forEach((m, i) => {
    const chip = moodRow.createSpan({ cls: "daily-mood" + (state.mood === i + 1 ? " is-on" : ""), text: m });
    chip.addEventListener("click", () => {
      state.mood = state.mood === i + 1 ? null : i + 1;
      [...moodRow.children].forEach((c, k) => c.classList.toggle("is-on", state.mood != null && k === state.mood - 1));
      writeFM({ mood: state.mood });
    });
  });

  const energyBox = meters.createDiv({ cls: "daily-meter" });
  energyBox.createSpan({ cls: "daily-meter-label", text: "精力" });
  const energyRow = energyBox.createDiv({ cls: "daily-meter-row" });
  for (let i = 1; i <= 5; i++) {
    const seg = energyRow.createSpan({ cls: "daily-energy" + (state.energy >= i ? " is-on" : "") });
    seg.addEventListener("click", () => {
      state.energy = state.energy === i ? null : i;
      [...energyRow.children].forEach((c, k) => c.classList.toggle("is-on", state.energy != null && k < state.energy));
      writeFM({ energy: state.energy });
    });
  }

  function writeFM(patch) {
    try {
      const file = app.vault.getAbstractFileByPath(cur.file.path);
      if (!file) return;
      app.fileManager.processFrontMatter(file, (fm) => {
        for (const [k, v] of Object.entries(patch)) {
          if (v == null) delete fm[k]; else fm[k] = v;
        }
      });
    } catch (e) { console.warn("[daily] frontmatter 写入失败", e); }
  }
}
```

```dataviewjs
// ===== 今日日程：自动拉取当天 Full Calendar 日程（可自由编辑本代码块）=====
const dayStr = (dv.current().file.name.match(/\d{4}-\d{2}-\d{2}/) || [])[0];
if (!dayStr) {
  dv.paragraph("⚠️ 无法从文件名解析日期（需要 YYYY-MM-DD 格式）");
} else {
  const day = dv.luxon.DateTime.fromISO(dayStr);
  const ds = day.startOf("day");
  const de = day.endOf("day");
  const CAL = {
    Assignments: { name: "作业", color: "#ffb7c5" },
    Courses:     { name: "课程", color: "#a5cdff" },
    Exams:       { name: "考试", color: "#ffd1a5" },
    Habits:      { name: "习惯", color: "#cdb6f2" },
    Life:        { name: "生活", color: "#cbd5e1" },
    Meetings:    { name: "会议", color: "#a8e0b4" },
  };
  const rows = [];
  for (const p of dv.pages('"05_Calendar"')) {
    const folder = (p.file.folder || "").split("/").pop();
    const cal = CAL[folder];
    if (!cal) continue;
    let startDay = null;
    let endDay = null;
    try { startDay = p.date ? dv.date(p.date) : p.file.day; } catch (e) {}
    if (!startDay) continue;
    try { endDay = p.endDate ? dv.date(p.endDate) : startDay; } catch (e) { endDay = startDay; }
    if (startDay > de || endDay < ds) continue;
    rows.push({
      title: p.title ? String(p.title) : p.file.name,
      path: p.file.path,
      color: cal.color,
      name: cal.name,
      time: p.startTime
        ? String(p.startTime).slice(0, 5) + (p.endTime ? " – " + String(p.endTime).slice(0, 5) : "")
        : "全天",
      done: !!p.completed && p.completed !== "false",
      sort: p.startTime ? String(p.startTime) : "00:00",
    });
  }
  rows.sort((a, b) => a.sort.localeCompare(b.sort));
  const card = dv.container.createDiv({ cls: "daily-schedule" });
  card.createDiv({ cls: "daily-schedule-title", text: "今日日程" });
  if (!rows.length) {
    card.createDiv({ cls: "daily-empty", text: "☁️ 今天没有日程" });
  } else {
    const wrap = card.createDiv({ cls: "daily-events" });
    for (const r of rows) {
      const row = wrap.createDiv({ cls: "daily-ev" + (r.done ? " is-done" : "") });
      row.createSpan({ cls: "daily-ev-dot" }).style.background = r.color;
      const link = row.createEl("a", { cls: "daily-ev-title", text: r.title });
      link.addEventListener("click", (ev) => {
        ev.preventDefault();
        app.workspace.openLinkText(r.path, dv.current().file.path, false);
      });
      row.createSpan({ cls: "daily-ev-time", text: "· " + r.time });
      row.createSpan({ cls: "daily-ev-cal", text: r.name });
    }
  }
}
```

> [!todo] 待办
> - [ ] 

> [!good] 三件好事
> 1. 
> 2. 
> 3. 

> [!tip] 随手记
> 

> [!tomorrow] 明日最重要的一件事
> → 
