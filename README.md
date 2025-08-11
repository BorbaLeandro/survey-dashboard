# survey-dashboard
Manual entry dashboard for course survey results with weighted averages.
[README.md](https://github.com/user-attachments/files/21712771/README.md)
# Survey Dashboard — Google Sheets Edition

This is a React + Vite app that **reads survey entries from a public Google Sheet (CSV)** and renders a dashboard with **weighted monthly trends** and **per-question charts**.

## 1) Prepare your Google Sheet

Create a sheet with the following headers (order can vary, but these keys must exist):

```
course, year, month, participants, Q1, Q2, Q3, Q4, Q5
```

- `course`: string (e.g., "Course X")
- `year`: number (e.g., 2025)
- `month`: number 1–12
- `participants`: number (e.g., 16, 20, ...)
- `Q*`: question scores (1–10 by default). You can use your own labels instead of Q1..Q5 — the app will read labels from the CSV header.

Example rows:

```
Course X, 2025, 7, 16, 9.2, 9.0, 8.8, 9.6, 9.4
Course X, 2025, 8, 20, 9.1, 9.3, 9.0, 9.5, 9.2
Course Y, 2025, 8, 18, 8.7, 8.9, 8.5, 9.1, 8.8
```

### Publish the sheet as CSV
- In Google Sheets: **File → Share → Publish to web → Entire document → CSV**.
- Copy the CSV link. It will look like:
  `https://docs.google.com/spreadsheets/d/{SHEET_ID}/pub?output=csv`

Alternatively you can use the direct export format:
`https://docs.google.com/spreadsheets/d/{SHEET_ID}/export?format=csv&id={SHEET_ID}&gid={GID}`

## 2) Configure the app

Edit `.env` or set environment variables at deploy time:

```
VITE_SHEET_CSV_URL=<your_public_csv_url>
VITE_SCALE_MIN=1
VITE_SCALE_MAX=10
```

For local dev, create a file named `.env` in the project root with those values.

## 3) Run locally

```
npm install
npm run dev
```

## 4) Deploy

- **Vercel**: Import this repo and set `VITE_SHEET_CSV_URL` in Project → Settings → Environment Variables.
- **CodeSandbox/StackBlitz**: Import the repo/ZIP, set the env var in their UI, or hardcode a fallback in `src/config.js`.

---

If you don't have a sheet ready yet, the app lets you paste a CSV link on the page for quick testing.
import React, { useEffect, useMemo, useRef, useState } from "react";
import { motion } from "framer-motion";
import { LineChart, Line, XAxis, YAxis, Tooltip, CartesianGrid, ResponsiveContainer, BarChart, Bar, Legend } from "recharts";

/**
 * Survey Dashboard — Manual Entry (Prototype)
 * - Single‑file React app using Tailwind + Recharts + Framer Motion
 * - Manual entry of monthly survey results per course
 * - Variable participants per course/month (e.g., 16, 20, ...)
 * - Computes weighted averages, per‑question trends, and overall trend
 * - LocalStorage persistence, CSV import/export
 *
 * NOTE: Replace placeholder questions in SETTINGS to match your survey.
 */

// ---------- Helpers ----------
const fmtMonth = (m) => m.toString().padStart(2, "0");
const mid = () => Math.random().toString(36).slice(2, 10);
const clamp = (n, a, b) => Math.max(a, Math.min(b, n));

const SETTINGS_DEFAULT = {
  questions: [
    { id: "Q1", label: "Overall satisfaction" },
    { id: "Q2", label: "Faculty quality" },
    { id: "Q3", label: "Content relevance" },
    { id: "Q4", label: "Organization & logistics" },
    { id: "Q5", label: "Likelihood to recommend" },
  ],
  scaleMin: 1,
  scaleMax: 10,
};

function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try {
      const v = localStorage.getItem(key);
      return v ? JSON.parse(v) : initial;
    } catch (e) {
      return initial;
    }
  });
  useEffect(() => {
    try { localStorage.setItem(key, JSON.stringify(value)); } catch (e) {}
  }, [key, value]);
  return [value, setValue];
}

// ---------- Data Model ----------
// Entry = { id, course, year, month, participants, scores: {Q1: number, ...} }

// ---------- Components ----------
function Section({ title, children, right }) {
  return (
    <div className="bg-white/80 border border-stone-200 rounded-2xl p-5 shadow-sm">
      <div className="flex items-center justify-between gap-3">
        <h2 className="text-lg font-bold text-stone-900">{title}</h2>
        {right}
      </div>
      <div className="mt-4">{children}</div>
    </div>
  );
}

function numberOrEmpty(n) {
  if (n === null || n === undefined || n === "") return "";
  return Number(n);
}

function EntryForm({ settings, onSubmit, initial }) {
  const [course, setCourse] = useState(initial?.course || "Course X");
  const [year, setYear] = useState(initial?.year || new Date().getFullYear());
  const [month, setMonth] = useState(initial?.month || new Date().getMonth() + 1);
  const [participants, setParticipants] = useState(initial?.participants || 20);
  const [scores, setScores] = useState(() => {
    const base = {};
    settings.questions.forEach((q) => {
      base[q.id] = numberOrEmpty(initial?.scores?.[q.id]);
    });
    return base;
  });

  const valid = useMemo(() => {
    if (!course) return false;
    if (!participants || participants <= 0) return false;
    return true;
  }, [course, participants]);

  const handleSubmit = (e) => {
    e.preventDefault();
    if (!valid) return;
    const cleanScores = {};
    settings.questions.forEach((q) => {
      const v = Number(scores[q.id]);
      cleanScores[q.id] = isNaN(v) ? null : clamp(v, settings.scaleMin, settings.scaleMax);
    });
    onSubmit({
      id: initial?.id || mid(),
      course: course.trim(),
      year: Number(year),
      month: Number(month),
      participants: Number(participants),
      scores: cleanScores,
    });
  };

  return (
    <form onSubmit={handleSubmit} className="grid grid-cols-1 md:grid-cols-2 gap-4">
      <div>
        <label className="text-sm text-stone-600">Course</label>
        <input className="mt-1 w-full rounded-lg border p-2" value={course} onChange={(e) => setCourse(e.target.value)} />
      </div>
      <div className="grid grid-cols-2 gap-3">
        <div>
          <label className="text-sm text-stone-600">Year</label>
          <input type="number" className="mt-1 w-full rounded-lg border p-2" value={year} onChange={(e) => setYear(e.target.value)} />
        </div>
        <div>
          <label className="text-sm text-stone-600">Month</label>
          <select className="mt-1 w-full rounded-lg border p-2" value={month} onChange={(e) => setMonth(e.target.value)}>
            {Array.from({ length: 12 }).map((_, i) => (
              <option key={i + 1} value={i + 1}>{i + 1}</option>
            ))}
          </select>
        </div>
      </div>
      <div>
        <label className="text-sm text-stone-600">Participants</label>
        <input type="number" className="mt-1 w-full rounded-lg border p-2" value={participants} onChange={(e) => setParticipants(e.target.value)} />
      </div>

      <div className="md:col-span-2">
        <div className="text-sm font-medium text-stone-700">Scores ({settings.scaleMin}–{settings.scaleMax})</div>
        <div className="mt-2 grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-3">
          {settings.questions.map((q) => (
            <div key={q.id}>
              <label className="text-xs text-stone-600">{q.label}</label>
              <input
                type="number"
                step="0.1"
                placeholder="e.g. 9.2"
                className="mt-1 w-full rounded-lg border p-2"
                value={scores[q.id] ?? ""}
                onChange={(e) => setScores((s) => ({ ...s, [q.id]: e.target.value }))}
              />
            </div>
          ))}
        </div>
      </div>

      <div className="md:col-span-2 flex items-center justify-end gap-2">
        <button type="submit" className="px-4 py-2 rounded-xl bg-stone-900 text-white">{initial ? "Save changes" : "Add entry"}</button>
      </div>
    </form>
  );
}

function csvExport(entries, settings) {
  const head = ["id","course","year","month","participants",...settings.questions.map(q=>q.id)];
  const rows = entries.map(e => [e.id, e.course, e.year, e.month, e.participants, ...settings.questions.map(q => e.scores[q.id] ?? "")]);
  const csv = [head.join(","), ...rows.map(r => r.join(","))].join("\n");
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = `survey-entries.csv`; a.click();
  URL.revokeObjectURL(url);
}

function csvImport(text, settings) {
  const [headLine, ...lines] = text.split(/\r?\n/).filter(Boolean);
  const head = headLine.split(",");
  const qIds = settings.questions.map(q=>q.id);
  return lines.map((line) => {
    const cols = line.split(",");
    const obj = { id: cols[0] || mid(), course: cols[1], year: Number(cols[2]), month: Number(cols[3]), participants: Number(cols[4]), scores: {} };
    qIds.forEach((qid, i) => {
      const v = Number(cols[5 + i]);
      obj.scores[qid] = isNaN(v) ? null : v;
    });
    return obj;
  });
}

function dedupeByKey(arr, keyFn) {
  const map = new Map();
  arr.forEach((e) => { map.set(keyFn(e), e); });
  return Array.from(map.values());
}

export default function App() {
  const [settings, setSettings] = useLocalStorage("sd_settings_v1", SETTINGS_DEFAULT);
  const [entries, setEntries] = useLocalStorage("sd_entries_v1", []);
  const [editing, setEditing] = useState(null);
  const [filters, setFilters] = useState({ year: "all", course: "all" });

  const courses = useMemo(() => Array.from(new Set(entries.map(e => e.course))).sort(), [entries]);
  const years = useMemo(() => Array.from(new Set(entries.map(e => e.year))).sort(), [entries]);

  const filtered = useMemo(() => entries.filter(e => (filters.course === "all" || e.course === filters.course) && (filters.year === "all" || e.year === Number(filters.year))), [entries, filters]);

  const byMonth = useMemo(() => {
    // Aggregate weighted average per month for overall and per question
    const key = (e) => `${e.year}-${fmtMonth(e.month)}-${e.course}`;
    const groups = {};
    filtered.forEach((e) => {
      const k = `${e.year}-${fmtMonth(e.month)}`; // per month across courses
      if (!groups[k]) groups[k] = { y: e.year, m: e.month, totalP: 0, sumOverall: 0, q: {} };
      const qIds = settings.questions.map(q=>q.id);
      const avgQ = qIds.map((qid) => (e.scores[qid] ?? null)).filter((v) => v !== null);
      const overall = avgQ.length ? avgQ.reduce((a,b)=>a+b,0)/avgQ.length : null;
      const p = e.participants || 0;
      if (overall !== null) {
        groups[k].sumOverall += overall * p;
        groups[k].totalP += p;
      }
      qIds.forEach((qid) => {
        const v = e.scores[qid];
        if (v !== null && !isNaN(v)) {
          if (!groups[k].q[qid]) groups[k].q[qid] = { sum: 0, p: 0 };
          groups[k].q[qid].sum += v * p;
          groups[k].q[qid].p += p;
        }
      });
    });
    const rows = Object.values(groups).sort((a,b)=> a.y === b.y ? a.m - b.m : a.y - b.y).map((g) => {
      const obj = { monthLabel: `${g.y}-${fmtMonth(g.m)}` };
      obj.overall = g.totalP ? +(g.sumOverall / g.totalP).toFixed(2) : null;
      settings.questions.forEach((q) => {
        const s = g.q[q.id];
        obj[q.id] = s && s.p ? +(s.sum / s.p).toFixed(2) : null;
      });
      return obj;
    });
    return rows;
  }, [filtered, settings.questions]);

  const tableSorted = useMemo(() => entries.slice().sort((a,b)=> a.year === b.year ? (a.month === b.month ? a.course.localeCompare(b.course) : a.month - b.month) : a.year - b.year), [entries]);

  const addOrUpdate = (e) => {
    setEntries((arr) => {
      const next = arr.slice();
      const idx = next.findIndex((x) => x.id === e.id);
      if (idx >= 0) next[idx] = e; else next.push(e);
      return next;
    });
    setEditing(null);
  };

  const remove = (id) => setEntries((arr) => arr.filter((e) => e.id !== id));

  const exportCSV = () => csvExport(entries, settings);
  const importCSV = (file) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      const text = e.target.result;
      try {
        const rows = csvImport(text, settings);
        setEntries((prev) => dedupeByKey([...prev, ...rows], (x) => x.id));
      } catch (err) {
        alert("CSV inválido");
      }
    };
    reader.readAsText(file);
  };

  const updateQuestionLabel = (qid, label) => {
    setSettings((s) => ({ ...s, questions: s.questions.map((q) => q.id === qid ? { ...q, label } : q) }));
  };

  return (
    <div className="min-h-screen bg-emerald-50 text-stone-900">
      <div className="max-w-6xl mx-auto p-6">
        <header className="flex items-center justify-between gap-4">
          <div>
            <h1 className="text-2xl font-black">Survey Dashboard</h1>
            <p className="text-stone-600 text-sm">MVP para entrada manual e análise por mês/curso com médias ponderadas por participantes.</p>
          </div>
          <div className="flex items-center gap-2">
            <button onClick={exportCSV} className="px-3 py-2 rounded-lg border bg-white">Export CSV</button>
            <label className="px-3 py-2 rounded-lg border bg-white cursor-pointer">
              Import CSV
              <input type="file" accept=".csv" className="hidden" onChange={(e)=>e.target.files?.[0] && importCSV(e.target.files[0])} />
            </label>
          </div>
        </header>

        <div className="mt-6 grid grid-cols-1 lg:grid-cols-3 gap-5">
          <div className="lg:col-span-2 space-y-5">
            <Section title="Add / Edit Entry" right={editing && <button onClick={()=>setEditing(null)} className="text-sm underline">clear</button>}>
              <EntryForm settings={settings} onSubmit={addOrUpdate} initial={editing} />
            </Section>

            <Section title="Trend — Weighted Overall by Month">
              <div className="h-72 w-full">
                <ResponsiveContainer>
                  <LineChart data={byMonth} margin={{ top: 10, right: 10, bottom: 0, left: 0 }}>
                    <CartesianGrid strokeDasharray="3 3" />
                    <XAxis dataKey="monthLabel" tick={{ fontSize: 12 }} />
                    <YAxis domain={[settings.scaleMin, settings.scaleMax]} tick={{ fontSize: 12 }} />
                    <Tooltip />
                    <Line type="monotone" dataKey="overall" dot={true} strokeWidth={2} />
                  </LineChart>
                </ResponsiveContainer>
              </div>
            </Section>

            <Section title="Per‑Question (Weighted) — Latest 12 months">
              <div className="h-80 w-full">
                <ResponsiveContainer>
                  <BarChart data={byMonth.slice(-12)} margin={{ top: 10, right: 10, bottom: 0, left: 0 }}>
                    <CartesianGrid strokeDasharray="3 3" />
                    <XAxis dataKey="monthLabel" tick={{ fontSize: 12 }} />
                    <YAxis domain={[settings.scaleMin, settings.scaleMax]} tick={{ fontSize: 12 }} />
                    <Tooltip />
                    <Legend />
                    {settings.questions.map((q) => (
                      <Bar key={q.id} dataKey={q.id} stackId={undefined} />
                    ))}
                  </BarChart>
                </ResponsiveContainer>
              </div>
            </Section>
          </div>

          <div className="space-y-5">
            <Section title="Filters">
              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="text-xs text-stone-600">Year</label>
                  <select className="mt-1 w-full rounded-lg border p-2" value={filters.year} onChange={(e)=>setFilters(f=>({ ...f, year: e.target.value }))}>
                    <option value="all">All</option>
                    {years.map((y)=> <option key={y} value={y}>{y}</option>)}
                  </select>
                </div>
                <div>
                  <label className="text-xs text-stone-600">Course</label>
                  <select className="mt-1 w-full rounded-lg border p-2" value={filters.course} onChange={(e)=>setFilters(f=>({ ...f, course: e.target.value }))}>
                    <option value="all">All</option>
                    {courses.map((c)=> <option key={c} value={c}>{c}</option>)}
                  </select>
                </div>
              </div>
            </Section>

            <Section title="Questions">
              <div className="space-y-3">
                {settings.questions.map((q) => (
                  <div key={q.id} className="flex items-center gap-2">
                    <span className="text-xs px-2 py-1 rounded bg-stone-100 border">{q.id}</span>
                    <input className="flex-1 rounded-lg border p-2" value={q.label} onChange={(e)=>updateQuestionLabel(q.id, e.target.value)} />
                  </div>
                ))}
                <p className="text-xs text-stone-600">Edite os rótulos para combinar com seu questionário. Escala padrão {settings.scaleMin}–{settings.scaleMax}.</p>
              </div>
            </Section>

            <Section title="Quick Stats (filtered)">
              <Stats entries={filtered} settings={settings} />
            </Section>
          </div>
        </div>

        <div className="mt-6">
          <Section title="Data Table">
            <div className="overflow-auto rounded-lg border">
              <table className="min-w-full text-sm">
                <thead className="bg-stone-100">
                  <tr>
                    <th className="p-2 text-left">Course</th>
                    <th className="p-2 text-left">Year</th>
                    <th className="p-2 text-left">Month</th>
                    <th className="p-2 text-left">Participants</th>
                    {settings.questions.map((q)=> <th key={q.id} className="p-2 text-left">{q.id}</th>)}
                    <th className="p-2 text-left">Overall</th>
                    <th className="p-2"></th>
                  </tr>
                </thead>
                <tbody>
                  {tableSorted.map((e) => {
                    const qVals = settings.questions.map((q)=> e.scores[q.id]).filter(v=>v!==null && !isNaN(v));
                    const overall = qVals.length ? (qVals.reduce((a,b)=>a+b,0)/qVals.length).toFixed(2) : "—";
                    return (
                      <tr key={e.id} className="border-t">
                        <td className="p-2 whitespace-nowrap">{e.course}</td>
                        <td className="p-2">{e.year}</td>
                        <td className="p-2">{fmtMonth(e.month)}</td>
                        <td className="p-2">{e.participants}</td>
                        {settings.questions.map((q)=> <td key={q.id} className="p-2">{e.scores[q.id] ?? ""}</td>)}
                        <td className="p-2">{overall}</td>
                        <td className="p-2 text-right">
                          <div className="flex items-center gap-2 justify-end">
                            <button className="px-2 py-1 rounded border" onClick={()=>setEditing(e)}>Edit</button>
                            <button className="px-2 py-1 rounded border" onClick={()=>remove(e.id)}>Delete</button>
                          </div>
                        </td>
                      </tr>
                    );
                  })}
                </tbody>
              </table>
            </div>
          </Section>
        </div>
      </div>
    </div>
  );
}

function Stats({ entries, settings }) {
  const totalCourses = new Set(entries.map(e=>e.course)).size;
  const totalEvents = entries.length;
  const totalParticipants = entries.reduce((a,b)=> a + (b.participants||0), 0);

  const weightedOverall = useMemo(() => {
    let sum = 0, pTot = 0;
    entries.forEach((e) => {
      const qs = settings.questions.map((q) => e.scores[q.id]).filter((v) => v !== null && !isNaN(v));
      if (!qs.length) return;
      const avg = qs.reduce((a,b)=>a+b,0)/qs.length;
      sum += avg * (e.participants || 0);
      pTot += (e.participants || 0);
    });
    return pTot ? (sum / pTot).toFixed(2) : "—";
  }, [entries, settings.questions]);

  return (
    <div className="grid grid-cols-2 gap-4">
      <Stat label="Courses" value={totalCourses} />
      <Stat label="Entries" value={totalEvents} />
      <Stat label="Participants" value={totalParticipants} />
      <Stat label="Weighted Overall" value={weightedOverall} />
    </div>
  );
}

function Stat({ label, value }) {
  return (
    <div className="rounded-xl border p-4 bg-white">
      <div className="text-xs text-stone-500">{label}</div>
      <div className="text-2xl font-bold">{value}</div>
    </div>
  );
}

