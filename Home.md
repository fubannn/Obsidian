---
cssclasses:
  - home-page
---

```dataviewjs
// ============================================================
// Delphi Home Dashboard — 参考图式桌面布局
// 数据源：05_Calendar 下 Full Calendar Remastered 的事件笔记
// 样式：.obsidian/snippets/home-glass.css
// ============================================================

const DateTime = dv.luxon.DateTime;
const now = DateTime.now();
const today = now.startOf("day");
const todayEnd = today.endOf("day");
const M = window.moment;
const notify = (msg) => { try { if (typeof Notice !== "undefined") new Notice(msg); } catch (e) { console.log(msg); } };

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
    deadline: end,
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
const overdue = events.filter((e) => !e.done && e.end < today).sort((a, b) => a.start - b.start);
const upcoming = events.filter((e) => !e.done && e.start <= horizonEnd && e.end >= today).sort((a, b) => a.start - b.start);

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

// ---------------- 日期标签 ----------------
function dateLabel(e, short) {
  if (e.end < today) {
    return { text: short ? "逾期" : `已逾期 ${Math.round(today.diff(e.end.startOf("day"), "days").days)} 天`, cls: "is-overdue" };
  }
  if (e.start <= todayEnd && e.end >= today) return { text: "今天", cls: "is-today" };
  if (e.start <= today.plus({ days: 1 }).endOf("day")) return { text: "明天", cls: "" };
  return { text: `${e.start.month}月${e.start.day}日`, cls: "" };
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

// ---------------- 快速创建日程笔记 ----------------
async function createQuickEvent(title, folder, dateStr) {
  const safe = String(title).replace(/[\\/:*?"<>|#^[\]]/g, "-").trim();
  const base = `05_Calendar/${folder}/${dateStr} ${safe}`;
  let path = base + ".md";
  let i = 2;
  while (app.vault.getAbstractFileByPath(path)) { path = `${base} ${i}.md`; i++; }
  const calName = CAL[folder]?.name || folder;
  const content = `---\ntitle: "${safe.replace(/"/g, '\\"')}"\nallDay: true\ndate: "${dateStr}"\nendDate:\ncompleted: false\n---\n# ${safe}\n\n**时间**：${dateStr} All Day\n**地点**：\n**日历**：${calName}\n\n## Description\n\n\n## 要做的事\n- \n\n## 准备材料\n- \n\n## 记录\n- \n\n## 相关笔记\n- \n`;
  await app.vault.create(path, content);
  // 让 Dataview 立刻刷新视图（命令 + 索引兜底）
  try { app.plugins?.plugins?.dataview?.index?.touch?.(); } catch (e) {}
  app.commands.executeCommandById("dataview:dataview-force-refresh-views");
}

// ---------------- 侧栏自动收放 ----------------
// 进入 Home：快照侧栏状态并收起左右分栏，让主页铺满
// 离开 Home：恢复之前的状态
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
      savedLeft = ls ? ls.collapsed : null;
      savedRight = rs ? rs.collapsed : null;
      inHome = true;
      const collapseAll = () => {
        if (!inHome) return;
        const l = app.workspace.leftSplit;
        const r = app.workspace.rightSplit;
        if (l && !l.collapsed) l.collapse();
        if (r && !r.collapsed) r.collapse();
      };
      collapseAll();
      // 有些插件（如 Full Calendar 侧栏）会在启动稍晚时重新展开侧栏，补几次
      [350, 1200, 2500].forEach((ms) => setTimeout(collapseAll, ms));
    } else if (!nowHome && inHome) {
      if (ls && savedLeft === false) ls.expand();
      if (rs && savedRight === false) rs.expand();
      savedLeft = null;
      savedRight = null;
      inHome = false;
    }
  }

  app.workspace.on("active-leaf-change", sync);
  app.workspace.on("layout-change", sync);
  setTimeout(sync, 0);
}

// ---------------- 渲染 ----------------
const root = el(dv.container, "div", "home-dash");
const grid = el(root, "div", "home-grid");

/* ===== 1. 横幅：问候 + 时钟 + 统计块 ===== */
const hero = el(grid, "div", "home-hero");
const heroLeft = el(hero, "div", "home-hero-left");
const h = now.hour;
const greet = h < 5 ? "夜深了" : h < 11 ? "早上好" : h < 13 ? "中午好" : h < 18 ? "下午好" : "晚上好";
el(heroLeft, "div", "home-greeting", greet);

const clockRow = el(heroLeft, "div", "home-clockrow");
const clock = el(clockRow, "span", "home-clock-big");
const clockSS = el(clockRow, "span", "home-clock-ss");
el(clockRow, "span", "home-date", M().locale("zh-cn").format("YYYY年M月D日 dddd"));

const bits = [];
bits.push(todayTodo ? `今天 ${todayTodo} 件待办` : "今天没有待办");
if (overdue.length) bits.push(`逾期 ${overdue.length} 件`);
if (upcoming.length) bits.push(`最近：${upcoming[0].title}`);
bits.push(`本周已完成 ${weekDone}/${weekEvents.length}`);
el(heroLeft, "div", "home-summary", bits.join(" · "));

let timer = null;
function tick() {
  if (!document.body.contains(clock)) {
    if (timer) clearInterval(timer);
    timer = null;
    return;
  }
  const m = M();
  clock.textContent = m.format("HH:mm");
  clockSS.textContent = m.format("ss");
}
tick();
timer = setInterval(tick, 1000);

const heroStats = el(hero, "div", "home-stats");
function statTile(label, value, sub, ratio) {
  const t = el(heroStats, "div", "home-stat");
  el(t, "div", "home-stat-label", label);
  el(t, "div", "home-stat-value", value);
  el(t, "div", "home-stat-sub", sub);
  if (ratio !== null && ratio !== undefined) {
    const track = el(t, "div", "home-stat-bar");
    const fill = el(track, "div", "home-stat-fill");
    fill.style.width = Math.round(Math.max(0, Math.min(1, ratio)) * 100) + "%";
  }
}

const todayDoneRatio = todayEvents.length ? (todayEvents.length - todayTodo) / todayEvents.length : 0;
statTile("今日待办", String(todayTodo), todayEvents.length ? `今天共 ${todayEvents.length} 件` : "今天没有日程", todayDoneRatio);
statTile("本周完成", `${weekDone}/${weekEvents.length}`, "本周日程", weekEvents.length ? weekDone / weekEvents.length : 0);
statTile("按时完成率", onTimeRate === null ? "—" : onTimeRate + "%", onTimeCand.length ? `按时 ${onTime}/${onTimeCand.length}` : "本周暂无记录", onTimeRate === null ? 0 : onTimeRate / 100);
statTile("连续记录", `${streak} 天`, "连续写日记", null);

/* ===== 2. 分类卡片墙 ===== */
const board = el(grid, "div", "home-board");

const focus = el(board, "div", "home-focus");
el(focus, "div", "home-focus-label", "今日焦点");
el(focus, "div", "home-focus-num", String(todayTodo));
el(focus, "div", "home-focus-sub", "件待办");
const focusList = el(focus, "div", "home-focus-list");
for (const e of todayEvents.slice(0, 3)) {
  const i = el(focusList, "div", "home-focus-item", e.title);
  i.title = e.title;
  i.addEventListener("click", () => app.workspace.openLinkText(e.page.file.path, "", false));
}

for (const [folder, cal] of Object.entries(CAL)) {
  const card = el(board, "div", "home-cat");
  const head = el(card, "div", "home-cat-head");
  const dot = el(head, "span", "home-cat-dot");
  dot.style.background = cal.color;
  el(head, "span", "home-cat-name", cal.name);

  const evs = upcoming.filter((e) => e.folder === folder);
  el(head, "span", "home-cat-count", String(evs.length));

  const listEl = el(card, "div", "home-cat-list");
  if (!evs.length) {
    el(listEl, "div", "home-cat-empty", "暂无任务");
  } else {
    for (const e of evs.slice(0, 8)) {
      const item = el(listEl, "div", "home-cat-item");
      item.title = `${e.title} · ${e.timeText}`;
      const row1 = el(item, "div", "home-cat-item-top");
      const d = dateLabel(e, true);
      const tag = el(row1, "span", "home-cat-item-date " + d.cls, d.text);
      el(row1, "span", "home-cat-item-title", e.title);
      item.addEventListener("click", () => app.workspace.openLinkText(e.page.file.path, "", false));
    }
  }
}

/* ===== 3. 快速添加 / 本周总览 / 近期日程 ===== */
const mid = el(grid, "div", "home-mid");

// 快速添加
const qaCard = el(mid, "div", "home-card home-quickadd");
const qaHead = el(qaCard, "div", "home-card-head");
el(qaHead, "div", "home-card-title", "快速添加日程");
const qaForm = el(qaCard, "div", "home-qa-form");
const qaTitle = document.createElement("input");
qaTitle.type = "text";
qaTitle.className = "home-input";
qaTitle.placeholder = "输入日程标题，回车创建…";
qaForm.appendChild(qaTitle);

const qaRow = el(qaForm, "div", "home-qa-row");
const qaCal = document.createElement("select");
qaCal.className = "home-input home-select";
for (const [folder, cal] of Object.entries(CAL)) {
  const opt = document.createElement("option");
  opt.value = folder;
  opt.textContent = cal.name;
  qaCal.appendChild(opt);
}
const qaDate = document.createElement("input");
qaDate.type = "date";
qaDate.className = "home-input";
qaDate.value = today.toISODate();
const qaBtn = el(qaRow, "button", "home-btn-primary", "创建");
qaRow.appendChild(qaCal);
qaRow.appendChild(qaDate);
qaRow.appendChild(qaBtn);

async function doQuickAdd() {
  const title = qaTitle.value.trim();
  if (!title) { notify("请输入日程标题"); return; }
  const dateStr = qaDate.value || today.toISODate();
  await createQuickEvent(title, qaCal.value, dateStr);
  qaTitle.value = "";
  notify(`已创建：${dateStr} ${title}`);
}
qaBtn.addEventListener("click", doQuickAdd);
qaTitle.addEventListener("keydown", (ev) => { if (ev.key === "Enter") doQuickAdd(); });

// 本周总览
const weekCard = el(mid, "div", "home-card home-week");
const weekHead = el(weekCard, "div", "home-card-head");
el(weekHead, "div", "home-card-title", "本周总览");
el(weekHead, "div", "home-mini-note", `完成 ${weekDone}/${weekEvents.length}`);

const weekBars = el(weekCard, "div", "home-week-bars");
const weekDays = [];
for (let i = 0; i < 7; i++) weekDays.push(weekStart.plus({ days: i }));
const weekCounts = weekDays.map((d) => events.filter((e) => covers(e, d.startOf("day"), d.endOf("day"))).length);
const weekMax = Math.max(1, ...weekCounts);
for (let i = 0; i < 7; i++) {
  const d = weekDays[i];
  const col = el(weekBars, "div", "home-week-col" + (d.hasSame(today, "day") ? " is-today" : ""));
  const wrap = el(col, "div", "home-week-barwrap");
  const bar = el(wrap, "div", "home-week-bar");
  bar.style.height = Math.round((weekCounts[i] / weekMax) * 100) + "%";
  el(col, "div", "home-week-count", String(weekCounts[i]));
  el(col, "div", "home-week-day", WEEK_ZH[d.weekday % 7]);
}

const weekFoot = el(weekCard, "div", "home-week-foot");
el(weekFoot, "span", null, `完成率 ${weekEvents.length ? Math.round((weekDone / weekEvents.length) * 100) : 0}%`);
if (onTimeRate !== null) el(weekFoot, "span", null, ` · 按时 ${onTimeRate}%`);

// 近期日程
const agendaCard = el(mid, "div", "home-card home-agenda");
const agendaHead = el(agendaCard, "div", "home-card-head");
el(agendaHead, "div", "home-card-title", "近期日程");
const agBtns = el(agendaHead, "div", "home-head-btns");
const openCalBtn = el(agBtns, "button", "home-pill-btn", "打开日历");
openCalBtn.addEventListener("click", () => app.commands.executeCommandById("full-calendar-remastered:full-calendar-open"));
const newBtn = el(agBtns, "button", "home-pill-btn accent", "+ 新建");
newBtn.addEventListener("click", () => app.commands.executeCommandById("full-calendar-remastered:full-calendar-new-event"));

const agList = el(agendaCard, "div", "home-agenda-list");
const allRows = [...overdue, ...upcoming];
function renderRow(parent, e) {
  const row = el(parent, "div", "home-row");
  row.title = e.page.file.path;
  const dot = el(row, "span", "home-row-dot");
  dot.style.background = e.cal.color;
  const mid2 = el(row, "div", "home-row-main");
  el(mid2, "div", "home-row-title", e.title);
  el(mid2, "div", "home-row-meta", `${e.cal.name} · ${e.timeText}`);
  const d = dateLabel(e, false);
  el(row, "div", "home-row-date " + d.cls, d.text);
  row.addEventListener("click", () => app.workspace.openLinkText(e.page.file.path, "", false));
}
if (!allRows.length) {
  el(agList, "div", "home-empty", "未来 7 天没有待办日程 🎉");
} else {
  for (const e of allRows) renderRow(agList, e);
}

/* ===== 4. 分类进度 ===== */
const dirRow = el(grid, "div", "home-dir");
for (const [folder, cal] of Object.entries(CAL)) {
  const evs = weekEvents.filter((e) => e.folder === folder);
  const done = evs.filter((e) => e.done).length;
  const card = el(dirRow, "div", "home-dir-card");
  const head = el(card, "div", "home-dir-head");
  const dot = el(head, "span", "home-cat-dot");
  dot.style.background = cal.color;
  el(head, "span", "home-dir-name", cal.name);
  el(head, "span", "home-dir-count", String(evs.length));

  const track = el(card, "div", "home-dir-track");
  const fill = el(track, "div", "home-dir-fill");
  fill.style.background = cal.color;
  fill.style.width = evs.length ? Math.round((done / evs.length) * 100) + "%" : "0%";

  el(card, "div", "home-dir-sub", `本周 ${evs.length} 件 · 已完成 ${done}`);
}

/* ===== 5. 热力图 + 日历 ===== */
const foot = el(grid, "div", "home-foot");

// 热力图
const heatCard = el(foot, "div", "home-card home-heat");
const heatHead = el(heatCard, "div", "home-card-head");
el(heatHead, "div", "home-card-title", "本月热力图");
el(heatHead, "div", "home-mini-note", today.toFormat("yyyy年M月"));

const heatGrid = el(heatCard, "div", "home-heat-grid");
for (const w of ["一", "二", "三", "四", "五", "六", "日"]) {
  el(heatGrid, "div", "home-heat-week", w);
}
const heatOffset = today.startOf("month").weekday - 1;
for (let i = 0; i < heatOffset; i++) el(heatGrid, "div", "home-heat-cell is-empty");
const heatDays = today.daysInMonth;
for (let d = 1; d <= heatDays; d++) {
  const day = today.set({ day: d });
  const count = events.filter((e) => covers(e, day.startOf("day"), day.endOf("day"))).length;
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

// 日历
const calCard = el(foot, "div", "home-card home-cal");
const calHead = el(calCard, "div", "home-card-head");
el(calHead, "div", "home-card-title", "日历");
const calBtns = el(calHead, "div", "home-head-btns");
const calOpenBtn = el(calBtns, "button", "home-pill-btn", "打开日历");
calOpenBtn.addEventListener("click", () => app.commands.executeCommandById("full-calendar-remastered:full-calendar-open"));

const calNav = el(calCard, "div", "home-cal-nav");
const prevBtn = el(calNav, "button", "home-icon-btn", "‹");
const calLabel = el(calNav, "div", "home-cal-label");
const nextBtn = el(calNav, "button", "home-icon-btn", "›");
const todayBtn = el(calNav, "button", "home-pill-btn", "今天");
const calGrid = el(calCard, "div", "home-cal-grid");

let viewMonth = today.startOf("month");
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
    const dayEvs = events.filter((e) => covers(e, day.startOf("day"), day.endOf("day")));
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

prevBtn.addEventListener("click", () => { viewMonth = viewMonth.minus({ months: 1 }); renderMonth(); });
nextBtn.addEventListener("click", () => { viewMonth = viewMonth.plus({ months: 1 }); renderMonth(); });
todayBtn.addEventListener("click", () => { viewMonth = today.startOf("month"); renderMonth(); });
el(calCard, "div", "home-hint", "点击日期打开 / 创建当天日记");
```