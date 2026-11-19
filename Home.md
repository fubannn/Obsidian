---
cssclasses:
  - home-page
---

```dataviewjs
// ============================================================
// Delphi Home Dashboard
// 数据源：05_Calendar 下 Full Calendar Remastered 的事件笔记
// 样式：.obsidian/snippets/home-glass.css
// ============================================================

const DateTime = dv.luxon.DateTime;
const now = DateTime.now();
const today = now.startOf("day");
const todayEnd = today.endOf("day");
const M = window.moment;

// 日历目录 → 名称与颜色（与 Full Calendar 中的日历一一对应）
const CAL = {
  Assignments: { name: "作业", color: "#ffb7c5" },
  Courses:     { name: "课程", color: "#a5cdff" },
  Exams:       { name: "考试", color: "#ffd1a5" },
  Habits:      { name: "习惯", color: "#cdb6f2" },
  Life:        { name: "生活", color: "#cbd5e1" },
  Meetings:    { name: "会议", color: "#a8e0b4" },
};

// ---------------- helpers ----------------
function el(parent, tag, cls, text) {
  const e = document.createElement(tag);
  if (cls) e.className = cls;
  if (text != null) e.textContent = text;
  parent.appendChild(e);
  return e;
}

// 任意日期值 → luxon DateTime
function toLS(v) {
  if (!v) return null;
  if (DateTime.isDateTime(v)) return v;
  if (typeof v.toMillis === "function") {
    const m = v.toMillis();
    return Number.isFinite(m) ? DateTime.fromMillis(m) : null;
  }
  if (v instanceof Date) return DateTime.fromJSDate(v);
  if (typeof v === "string") {
    const r = DateTime.fromISO(v) || DateTime.fromISO(v.replace(" ", "T"));
    return r.isValid ? r : null;
  }
  return null;
}

function withTime(dt, hhmm) {
  const m = /^(\d{1,2}):(\d{2})/.exec(String(hhmm || ""));
  if (!m) return null;
  return dt.set({ hour: +m[1], minute: +m[2], second: 0, millisecond: 0 });
}

const WEEK_ZH = ["日", "一", "二", "三", "四", "五", "六"];
const weekdayZh = (d) => "周" + WEEK_ZH[d.weekday % 7];

// ---------------- 收集事件 ----------------
const events = [];
for (const p of dv.pages('"05_Calendar"')) {
  const folder = (p.file.folder || "").split("/").pop();
  const cal = CAL[folder];
  if (!cal) continue;
  const startDay = toLS(p.date) || toLS(p.file.day);
  if (!startDay) continue;

  const endDay = toLS(p.endDate) || startDay;
  const startTime = p.startTime != null ? String(p.startTime).slice(0, 5) : null;
  const endTime = p.endTime != null ? String(p.endTime).slice(0, 5) : null;
  const timed = !!startTime && p.allDay !== true;

  const start = timed ? (withTime(startDay, startTime) || startDay) : startDay.startOf("day");
  const end = timed ? (withTime(endDay, endTime || startTime) || endDay.endOf("day")) : endDay.endOf("day");
  const deadline = end;

  const doneVal = p.completed;
  const done = !!doneVal && doneVal !== "false";
  const doneAt = done ? toLS(doneVal) : null;

  events.push({
    page: p,
    folder,
    cal,
    title: p.title ? String(p.title) : p.file.name,
    start,
    end,
    deadline,
    timed,
    startTime,
    endTime,
    timeText: !timed ? "全天" : (endTime ? `${startTime} – ${endTime}` : startTime),
    done,
    doneAt,
  });
}

// ---------------- 指标 ----------------
const covers = (e, a, b) => e.start <= b && e.end >= a;
const todayEvents = events.filter((e) => covers(e, today, todayEnd));
const todayTodo = todayEvents.filter((e) => !e.done).length;

const weekStart = today.minus({ days: today.weekday - 1 });
const weekEnd = weekStart.plus({ days: 6 }).endOf("day");
const weekEvents = events.filter((e) => e.start >= weekStart && e.start <= weekEnd);
const weekDone = weekEvents.filter((e) => e.done).length;
const onTimeCand = weekEvents.filter((e) => e.done && e.doneAt);
const onTime = onTimeCand.filter((e) => e.doneAt <= e.deadline).length;
const onTimeRate = onTimeCand.length ? Math.round((onTime / onTimeCand.length) * 100) : null;

const horizonEnd = today.plus({ days: 7 }).endOf("day");
const overdue = events
  .filter((e) => !e.done && e.end < today)
  .sort((a, b) => a.start - b.start);
const upcoming = events
  .filter((e) => !e.done && e.start <= horizonEnd && e.end >= today)
  .sort((a, b) => a.start - b.start);

// 连续写日记天数
const dnCfg = (app.internalPlugins?.plugins?.["daily-notes"]?.instance?.options) || {};
const diaryFolder = String(dnCfg.folder || "01_Diary").replace(/\/+$/, "");
const diarySet = new Set(
  app.vault.getMarkdownFiles()
    .filter((f) => f.path.startsWith(diaryFolder + "/"))
    .map((f) => f.basename)
);
let streak = 0;
let cursor = today;
if (!diarySet.has(cursor.toFormat("yyyy-MM-dd"))) cursor = cursor.minus({ days: 1 });
while (diarySet.has(cursor.toFormat("yyyy-MM-dd"))) {
  streak++;
  cursor = cursor.minus({ days: 1 });
}

// ---------------- 打开 / 创建当天日记 ----------------
async function openDaily(day) {
  const cfg = (app.internalPlugins?.plugins?.["daily-notes"]?.instance?.options) || {};
  const folder = String(cfg.folder || "01_Diary").replace(/\/+$/, "");
  const format = cfg.format || "YYYY-MM-DD";
  const m = M(day.toISODate(), "YYYY-MM-DD", true);
  const fileName = m.format(format);
  const path = (folder ? folder + "/" : "") + fileName + ".md";

  let file = app.vault.getAbstractFileByPath(path);
  if (!file) {
    let content = "";
    const tpl = cfg.template;
    if (tpl) {
      const tplPath = tpl.endsWith(".md") ? tpl : tpl + ".md";
      const tf = app.vault.getAbstractFileByPath(tplPath);
      if (tf) content = await app.vault.read(tf);
    }
    content = content
      .replace(/\{\{\s*date(?::([^}]+))?\s*\}\}/g, (_, f) => m.format(f || "YYYY-MM-DD"))
      .replace(/\{\{\s*time(?::([^}]+))?\s*\}\}/g, (_, f) => M().format(f || "HH:mm"))
      .replace(/\{\{\s*title\s*\}\}/g, fileName);
    file = await app.vault.create(path, content);
  }
  await app.workspace.getLeaf(false).openFile(file);
}

// ---------------- 侧栏自动收放 ----------------
// 进入 Home：快照侧栏状态并收起左右分栏，让主页铺满
// 离开 Home：恢复之前的状态
// 用 window 标志保证多处渲染（dataviewjs 重跑）只注册一次
if (!window.__homeSidebarHook) {
  window.__homeSidebarHook = true;

  let savedLeft = null;
  let savedRight = null;
  let inHome = false;

  function isHomeFile(path) {
    return !!path && path.endsWith("Home.md");
  }

  function sync() {
    const f = app.workspace.getActiveFile();
    const nowHome = !!f && isHomeFile(f.path);
    const ls = app.workspace.leftSplit;
    const rs = app.workspace.rightSplit;

    if (nowHome && !inHome) {
      // 进入 Home：先快照，再收起
      savedLeft = ls ? ls.collapsed : null;
      savedRight = rs ? rs.collapsed : null;
      if (ls && !ls.collapsed) ls.collapse();
      if (rs && !rs.collapsed) rs.collapse();
      inHome = true;
    } else if (!nowHome && inHome) {
      // 离开 Home：恢复
      if (ls && savedLeft === false) ls.expand();
      if (rs && savedRight === false) rs.expand();
      savedLeft = null;
      savedRight = null;
      inHome = false;
    }
  }

  app.workspace.on("active-leaf-change", sync);
  app.workspace.on("layout-change", sync);

  // 启动时同步一次
  setTimeout(sync, 0);
}

// ---------------- 渲染 ----------------
const root = el(dv.container, "div", "home-dash");

// ===== Hero：问候 + 动态摘要 + 时钟 =====
const hero = el(root, "div", "glass-card home-hero");
const heroLeft = el(hero, "div", "home-hero-left");
const h = now.hour;
const greet = h < 5 ? "夜深了" : h < 11 ? "早上好" : h < 13 ? "中午好" : h < 18 ? "下午好" : "晚上好";
el(heroLeft, "div", "home-greeting", greet);

const bits = [];
bits.push(todayTodo ? `今天 ${todayTodo} 件待办` : "今天没有待办");
if (overdue.length) bits.push(`逾期 ${overdue.length} 件`);
if (upcoming.length) bits.push(`最近：${upcoming[0].title}`);
bits.push(`本周已完成 ${weekDone}/${weekEvents.length}`);
el(heroLeft, "div", "home-summary", bits.join(" · "));

const heroRight = el(hero, "div", "home-hero-right");
const clock = el(heroRight, "div", "home-clock");
const clockHM = el(clock, "span", "", "");
const clockSS = el(clock, "span", "home-clock-ss", "");
el(heroRight, "div", "home-today-date", M().locale("zh-cn").format("YYYY年M月D日 dddd"));

function tick() {
  if (!document.body.contains(clock)) { clearInterval(timer); return; }
  const m = M();
  clockHM.textContent = m.format("HH:mm");
  clockSS.textContent = m.format("ss");
}
tick();
const timer = setInterval(tick, 1000);

// ===== 统计卡 =====
const stats = el(root, "div", "home-stats");

function statCard(label, value, sub, fillRatio) {
  const card = el(stats, "div", "glass-card home-stat");
  el(card, "div", "home-stat-label", label);
  el(card, "div", "home-stat-value", value);
  el(card, "div", "home-stat-sub", sub);
  if (fillRatio !== null && fillRatio !== undefined) {
    const track = el(card, "div", "home-stat-bar");
    const fill = el(track, "div", "home-stat-fill");
    fill.style.width = Math.round(Math.max(0, Math.min(1, fillRatio)) * 100) + "%";
  }
}

const todayDoneRatio = todayEvents.length
  ? (todayEvents.length - todayTodo) / todayEvents.length
  : 0;
statCard(
  "今日待办",
  String(todayTodo),
  todayEvents.length ? `今天共 ${todayEvents.length} 件日程` : "今天没有日程",
  todayDoneRatio
);
statCard(
  "本周完成",
  `${weekDone}/${weekEvents.length}`,
  "本周日程完成情况",
  weekEvents.length ? weekDone / weekEvents.length : 0
);
statCard(
  "按时完成率",
  onTimeRate === null ? "—" : onTimeRate + "%",
  onTimeCand.length ? `按时完成 ${onTime}/${onTimeCand.length}` : "本周暂无完成记录",
  onTimeRate === null ? 0 : onTimeRate / 100
);
statCard("连续记录", `${streak} 天`, "连续写日记", null);

// ===== 主区：月历 + 近期日程 =====
const mainGrid = el(root, "div", "home-main");

// --- 月历卡 ---
const calCard = el(mainGrid, "div", "glass-card");
const calHead = el(calCard, "div", "home-card-head");
el(calHead, "div", "home-card-title", "日历");
const calHeadBtns = el(calHead, "div");
const todayBtn = el(calHeadBtns, "button", "home-pill-btn", "回到今天");

const calNav = el(calCard, "div", "home-cal-nav");
const prevBtn = el(calNav, "button", "home-icon-btn", "‹");
const calLabel = el(calNav, "div", "home-cal-label");
const nextBtn = el(calNav, "button", "home-icon-btn", "›");
const calGrid = el(calCard, "div", "home-cal-grid");
el(calCard, "div", "home-hint", "点击日期打开 / 创建当天日记");

let viewMonth = today.startOf("month");

function dayEventsOn(day) {
  const ds = day.startOf("day");
  const de = day.endOf("day");
  return events.filter((e) => e.start <= de && e.end >= ds).sort((a, b) => a.start - b.start);
}

function renderMonth() {
  calLabel.textContent = viewMonth.toFormat("yyyy年M月");
  calGrid.textContent = "";
  for (const w of ["一", "二", "三", "四", "五", "六", "日"]) {
    el(calGrid, "div", "home-cal-week", w);
  }
  const offset = viewMonth.startOf("month").weekday - 1;
  for (let i = 0; i < offset; i++) el(calGrid, "div", "home-day is-empty");

  const days = viewMonth.daysInMonth;
  for (let d = 1; d <= days; d++) {
    const day = viewMonth.set({ day: d });
    const dayEvs = dayEventsOn(day);
    const cell = el(calGrid, "div", "home-day");
    if (day.hasSame(today, "day")) cell.classList.add("is-today");
    el(cell, "span", "home-day-num", String(d));
    if (dayEvs.length) {
      const dots = el(cell, "span", "home-day-dots");
      for (const ev of dayEvs.slice(0, 3)) {
        const dot = el(dots, "span", "home-dot");
        dot.style.background = ev.cal.color;
      }
    }
    cell.title = day.toFormat("M月d日") + (dayEvs.length ? ` · ${dayEvs.length} 件日程` : "");
    cell.addEventListener("click", () => { openDaily(day); });
  }

  const rest = (7 - ((offset + days) % 7)) % 7;
  for (let i = 0; i < rest; i++) el(calGrid, "div", "home-day is-empty");
}

renderMonth();

prevBtn.addEventListener("click", () => {
  viewMonth = viewMonth.minus({ months: 1 });
  renderMonth();
});
nextBtn.addEventListener("click", () => {
  viewMonth = viewMonth.plus({ months: 1 });
  renderMonth();
});
todayBtn.addEventListener("click", () => {
  viewMonth = today.startOf("month");
  renderMonth();
});

// --- 近期日程卡 ---
const upCard = el(mainGrid, "div", "glass-card home-upcoming");
const upHead = el(upCard, "div", "home-card-head");
const upTitle = el(upHead, "div", "home-card-title");
upTitle.textContent = overdue.length
  ? `近期日程 · ${upcoming.length} 件待办 + ${overdue.length} 件逾期`
  : `近期日程 · ${upcoming.length} 件待办`;
const newBtn = el(upHead, "button", "home-pill-btn accent", "+ 新建");
newBtn.addEventListener("click", () => {
  app.commands.executeCommandById("full-calendar-remastered:full-calendar-new-event");
});

const list = el(upCard, "div", "home-list");

function renderRow(parent, e) {
  const row = el(parent, "div", "home-row");
  row.title = e.page.file.path;

  const dot = el(row, "span", "home-row-dot");
  dot.style.background = e.cal.color;

  const mid = el(row, "div", "home-row-main");
  el(mid, "div", "home-row-title", e.title);
  el(mid, "div", "home-row-meta", `${e.cal.name} · ${e.timeText}`);

  let label;
  let cls = "";
  if (e.end < today) {
    label = `已逾期 ${Math.round(today.diff(e.end.startOf("day"), "days").days)} 天`;
    cls = "is-overdue";
  } else if (e.start <= todayEnd && e.end >= today) {
    label = "今天";
    cls = "is-today";
  } else if (e.start <= today.plus({ days: 1 }).endOf("day")) {
    label = "明天";
  } else {
    label = `${e.start.month}月${e.start.day}日 ${weekdayZh(e.start)}`;
  }
  el(row, "div", "home-row-date " + cls, label);

  row.addEventListener("click", () => {
    app.workspace.openLinkText(e.page.file.path, "", false);
  });
}

const allRows = [...overdue, ...upcoming];
if (!allRows.length) {
  el(list, "div", "home-empty", "未来 7 天没有待办日程 🎉");
} else {
  for (const e of allRows) renderRow(list, e);
}

// ===== 底部：分类分布 + 热力图 + 快捷操作 =====
const bottom = el(root, "div", "home-bottom");

// --- 本周分类分布 ---
const distCard = el(bottom, "div", "glass-card");
const distHead = el(distCard, "div", "home-card-head");
el(distHead, "div", "home-card-title", "本周分类分布");
el(distHead, "div", "home-stat-sub", `共 ${weekEvents.length} 件`);

const distBody = el(distCard, "div", "home-dist");
const distData = Object.entries(CAL).map(([folder, cal]) => {
  const evs = weekEvents.filter((e) => e.folder === folder);
  return { cal, total: evs.length, done: evs.filter((e) => e.done).length };
});
const distMax = Math.max(1, ...distData.map((x) => x.total));

for (const x of distData) {
  const row = el(distBody, "div", "home-dist-row");
  el(row, "span", "home-dist-name", x.cal.name);
  const track = el(row, "span", "home-dist-track");
  const fill = el(track, "span", "home-dist-fill");
  fill.style.width = Math.round((x.total / distMax) * 100) + "%";
  fill.style.background = x.cal.color;
  const num = el(row, "span", "home-dist-num", String(x.total));
  if (x.total) num.title = `已完成 ${x.done}/${x.total}`;
}

// --- 本月热力图 ---
const heatCard = el(bottom, "div", "glass-card");
const heatHead = el(heatCard, "div", "home-card-head");
el(heatHead, "div", "home-card-title", "本月热力图");
el(heatHead, "div", "home-stat-sub", today.toFormat("yyyy年M月"));

const heatGrid = el(heatCard, "div", "home-heat-grid");
for (const w of ["一", "二", "三", "四", "五", "六", "日"]) {
  el(heatGrid, "div", "home-heat-week", w);
}
const heatOffset = today.startOf("month").weekday - 1;
for (let i = 0; i < heatOffset; i++) el(heatGrid, "div", "home-heat-cell is-empty");
const heatDays = today.daysInMonth;
for (let d = 1; d <= heatDays; d++) {
  const day = today.set({ day: d });
  const count = dayEventsOn(day).length;
  const level = count === 0 ? 0 : count <= 2 ? 1 : count <= 4 ? 2 : count <= 6 ? 3 : 4;
  const cell = el(heatGrid, "div", `home-heat-cell home-heat-${level}`);
  if (day.hasSame(today, "day")) cell.classList.add("is-today");
  el(cell, "span", null, String(d));
  cell.title = `${day.toFormat("M月d日")} · ${count} 件日程`;
  cell.addEventListener("click", () => { openDaily(day); });
}
for (let i = 0; i < (7 - ((heatOffset + heatDays) % 7)) % 7; i++) {
  el(heatGrid, "div", "home-heat-cell is-empty");
}

const legend = el(heatCard, "div", "home-heat-legend");
el(legend, "span", null, "少");
for (let i = 0; i <= 4; i++) el(legend, "span", `home-heat-cell home-heat-${i}`);
el(legend, "span", null, "多");

// --- 快捷操作 ---
const actCard = el(bottom, "div", "glass-card home-actions");
const actHead = el(actCard, "div", "home-card-head");
el(actHead, "div", "home-card-title", "快捷操作");

const btnNew = el(actCard, "button", "home-action-btn primary");
el(btnNew, "span", "home-action-icon", "✚");
const btnNewText = el(btnNew, "span");
el(btnNewText, "div", "home-action-title", "新建日程");
el(btnNewText, "div", "home-action-sub", "在 Full Calendar 中创建事件");
btnNew.addEventListener("click", () => {
  app.commands.executeCommandById("full-calendar-remastered:full-calendar-new-event");
});

const btnCal = el(actCard, "button", "home-action-btn secondary");
el(btnCal, "span", "home-action-icon", "📅");
const btnCalText = el(btnCal, "span");
el(btnCalText, "div", "home-action-title", "打开完整日历");
el(btnCalText, "div", "home-action-sub", "Full Calendar · 周视图");
btnCal.addEventListener("click", () => {
  app.commands.executeCommandById("full-calendar-remastered:full-calendar-open");
});
```