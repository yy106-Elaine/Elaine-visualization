---
theme: dashboard
title: Comedy Performances
toc: true
---

# Comedy Performance Dates 

The following graphs show number of comedy performances per year.

```js
import { filterComedyPerformances } from "./components/comedyCharts.js"; // [Added]

const danish_data = FileAttachment("data/danish-performances.csv").csv({
  typed: true,
});
const french_data = FileAttachment("data/french-performances.json").json();
```

```js
// Yearly Performance Chart Function
function yearChart(data) {
  return Plot.plot({
    title: "Performances by year",
    x: { label: "Year" },
    y: { grid: true, label: "Performances" },
    width: 1000,
    marks: [
      Plot.barY(data, Plot.groupX({ y: "count" }, { x: "year" })),
      Plot.ruleY([0]),
    ],
  });
}
```

# Danish Comedy Performances, 1748-1778

```js
const danish_comedy = danish_data.filter(
  (d) =>
    d.genre &&
    (d.genre.toLowerCase().includes("comed") ||
      d.genre.toLowerCase().includes("coméd"))
);
display(yearChart(danish_comedy));
```

# French Comedy Performances, 1748-1778

```js
const french_comedy = filterComedyPerformances(french_data);
display(yearChart(french_comedy));
```

# Comparative Comedy Performances, 1748-1778
```js
const combined_comedy_data = danish_comedy
  .map((d) => ({ ...d, origin: "danish" }))
  .concat(french_comedy.map((d) => ({ ...d, origin: "french" })))
  .map((d) => ({ ...d, year: String(d.year) }));
```

```js
const start_date = view(Inputs.date({label: "Start", value: "1748-01-01"}));
const end_date = view(Inputs.date({label: "End", value: "1778-12-31"}));
const origins = view(Inputs.checkbox(["danish", "french"], {label: "Origin", value: ["danish",  "french"]}));
```

```js
const formatted_data = combined_comedy_data.filter(d => (new Date(d.date) > start_date) && (new Date(d.date) <= end_date) && origins.includes(d.origin));
```
```js
function compareYearsChart(data) {
  return Plot.plot({
    title: "Compare performances per year",
    fx: { padding: 0, label: null },
    x: { axis: null, paddingOuter: 0.2 },
    y: { grid: true, label: "Performances" },
    color: { legend: true },
    width: 1000,
    marks: [
      Plot.barY(
        data,
        Plot.groupX(
          { y2: "count" },
          { x: "origin", fx: "year", fill: "origin" }
        )
      ),
      Plot.ruleY([0]),
    ],
  });
}

display(formatted_data.length > 0 ? compareYearsChart(formatted_data) : html`<i>No data.</i>`);
```

```js
// 处理函数：将原始表演数据按年份分组，统计喜剧和非喜剧数量
function processPerformanceData(fullData, comedyData, origin) {
  // 先把所有年份提取出来并建立 Map
  const allYears = d3.rollup(
    fullData,
    v => v.length,
    d => d.year
  );

  const comedyYears = d3.rollup(
    comedyData,
    v => v.length,
    d => d.year
  );

  // 将每年数据格式化，标记是 danish 还是 french
  return Array.from(allYears, ([year, total]) => {
    const comedy = comedyYears.get(year) || 0;
    const nonComedy = total - comedy;
    return {
      year: +year,
      comedy,
      nonComedy,
      origin,
      percent: comedy / total,
    };
  });
}

// 生成统计数据
const danish_summary = processPerformanceData(danish_data, danish_comedy, "danish");
const french_summary = processPerformanceData(french_data, french_comedy, "french");

// 合并两个国家的数据
const summary_data = danish_summary.concat(french_summary);

// 可视化：绘制左右分布的堆叠柱状图
display(Plot.legend({
  color: {
    domain: ["Danish Comedy", "Danish Non-Comedy", "French Comedy", "French Non-Comedy"],
    range: ["#fca5a5", "#991b1b", "#93c5fd", "#1e3a8a"]
  },
  title: "Legend",
  columns: 2
}))

display(
  Plot.plot({
  title: "Diverging Comedy Performance Chart (1748–1778)",
  width: 1000,
  height: 800,
  x: {
    label: "Number of Performances",
    tickFormat: Math.abs,
  },
  y: {
    label: "Year",
    reverse: true, // 从上到下显示年份
  },

  color: {
  domain: ["danish-comedy", "danish-nonComedy", "french-comedy", "french-nonComedy"],
  range: ["#fca5a5", "#991b1b", "#93c5fd", "#1e3a8a"],
},

  marks: [
    // 丹麦数据：显示在左侧（值为负数）
Plot.barX(
  danish_summary.flatMap(d => [
    { year: d.year, type: "nonComedy", value: -d.nonComedy, origin: "danish" },
    { year: d.year, type: "comedy", value: -d.comedy, origin: "danish", percent: d.percent }
  ]),
  {
    x: "value",
    y: "year",
    fill: d => `${d.origin}-${d.type}`, // 用组合键区分颜色
  }
),

    // 法国数据：显示在右侧（值为正数）
Plot.barX(
  french_summary.flatMap(d => [
    { year: d.year, type: "nonComedy", value: d.nonComedy, origin: "french" },
    { year: d.year, type: "comedy", value: d.comedy, origin: "french", percent: d.percent }
  ]),
  {
    x: "value",
    y: "year",
    fill: d => `${d.origin}-${d.type}`, // 同样用组合键
  }
),

    // 中心线
    Plot.ruleX([0]),

    // 丹麦百分比标签（左侧）
    Plot.text(
      danish_summary,
      {
        x: d => -(d.comedy + d.nonComedy) - 2, // 微调文字位置
        y: "year",
        text: d => `${Math.round(d.percent * 100)}%`,
        fill: "black",
        textAnchor: "end"
      }
    ),

    // 法国百分比标签（右侧）
    Plot.text(
      french_summary,
      {
        x: d => d.comedy + d.nonComedy + 2,
        y: "year",
        text: d => `${Math.round(d.percent * 100)}%`,
        fill: "black",
        textAnchor: "start"
      }
    )
  ]
})
)
```


```js
// filter out french tragedy, ballet and drama genres
const french_tragedy = french_data.filter(
  (d) =>
    d.genre &&
    (d.genre.toLowerCase().includes("tragédie"))
);

const french_ballet = french_data.filter(
  (d) =>
    d.genre &&
    (d.genre.toLowerCase().includes("ballet"))
);

const french_drama = french_data.filter(
  (d) =>
    d.genre &&
    (d.genre.toLowerCase().includes("drame")))

// filter out danish tragedy, ballet and drama genres
const danish_tragedy = danish_data.filter(
  (d) =>
    d.genre &&
    (d.genre.toLowerCase().includes("tragedia per musica") ||
      d.genre.toLowerCase().includes("tragedy"))
);

const danish_ballet = danish_data.filter(
  (d) =>
    d.genre &&
    (d.genre.toLowerCase().includes("ballet") ||
      d.genre.toLowerCase().includes("ballet,ballet")||
      d.genre.toLowerCase().includes("ballet,ballet,ballet"))
);

const danish_drama = danish_data.filter(
  (d) =>
    d.genre &&
    (d.genre.toLowerCase().includes("drama") ||
      d.genre.toLowerCase().includes("dramma giocoso per musica") ||
      d.genre.toLowerCase().includes("dramma pastorale")||
      d.genre.toLowerCase().includes("dramma per musica"))
);
```

```js
// 处理函数：将原始表演数据按年份分组，统计4种类型
function processPerformanceGenres(fullData, comedyData, dramaData, tragedyData, balletData, origin) {
  const allYears = d3.rollup(fullData, v => v.length, d => d.year);
  const comedyYears = d3.rollup(comedyData, v => v.length, d => d.year);
  const dramaYears = d3.rollup(dramaData.concat(tragedyData), v => v.length, d => d.year);
  const balletYears = d3.rollup(balletData, v => v.length, d => d.year);

  return Array.from(allYears, ([year, total]) => {
    const comedy = comedyYears.get(year) || 0;
    const drama = dramaYears.get(year) || 0;
    const ballet = balletYears.get(year) || 0;
    const other = total - comedy - drama - ballet;
    return {
      year: +year,
      origin,
      comedy,
      drama,
      ballet,
      other,
      percent: {
        comedy: comedy / total,
        drama: drama / total,
        ballet: ballet / total,
        other: other / total
      }
    };
  });
}

const danish_summary = processPerformanceGenres(danish_data, danish_comedy, danish_drama, danish_tragedy, danish_ballet, "danish");
const french_summary = processPerformanceGenres(french_data, french_comedy, french_drama, french_tragedy, french_ballet, "french");

display(Plot.legend({
  color: {
    domain: [
      "danish-comedy", "danish-drama", "danish-ballet", "danish-other",
      "french-comedy", "french-drama", "french-ballet", "french-other"
    ],
    range: ["#fca5a5", "#fb7185", "#ef4444", "#a3a3a3", "#93c5fd", "#60a5fa", "#3b82f6", "#6b7280"]
  },
  title: "Legend",
  columns: 2
}));

display(
  Plot.plot({
    title: "Diverging Genre Performance Chart (1748–1778)",
    width: 1000,
    height: 800,
    x: {
      label: "Number of Performances",
      tickFormat: Math.abs
    },
    y: {
      label: "Year",
      reverse: true
    },
    color: {
      domain: [
        "danish-comedy", "danish-drama", "danish-ballet", "danish-other",
        "french-comedy", "french-drama", "french-ballet", "french-other"
      ],
      range: ["#fca5a5", "#fb7185", "#ef4444", "#a3a3a3", "#93c5fd", "#60a5fa", "#3b82f6", "#6b7280"]
    },
  
    marks: [
      // 左侧（丹麦）：堆叠柱状图（负数）
      Plot.barX(
        danish_summary.flatMap(d => {
          const parts = [];
          let x = 0;
          for (const type of ["comedy", "drama", "ballet", "other"]) {
            const value = d[type];
            parts.push({
              year: d.year,
              x1: -x,
              x2: -(x + value),
              type,
              origin: "danish",
              percent: `${Math.round(d.percent[type] * 100)}%`
            });
            x += value;
          }
          return parts;
        }),
        {
          x1: "x1",
          x2: "x2",
          y: "year",
          fill: d => `${d.origin}-${d.type}`
        }
      ),

      // 右侧（法国）：堆叠柱状图（正数）
      Plot.barX(
        french_summary.flatMap(d => {
          const parts = [];
          let x = 0;
          for (const type of ["comedy", "drama", "ballet", "other"]) {
            const value = d[type];
            parts.push({
              year: d.year,
              x1: x,
              x2: x + value,
              type,
              origin: "french",
              percent: `${Math.round(d.percent[type] * 100)}%`
            });
            x += value;
          }
          return parts;
        }),
        {
          x1: "x1",
          x2: "x2",
          y: "year",
          fill: d => `${d.origin}-${d.type}`
        }
      ),

      // 中心线
      Plot.ruleX([0]),

      // 百分比文字标签（丹麦）
      Plot.text(
        danish_summary.flatMap(d => {
          const labels = [];
          let x = 0;
          for (const type of ["comedy", "drama", "ballet", "other"]) {
            const value = d[type];
            if (value > 0) {
              labels.push({
                year: d.year,
                x: -(x + value / 2),
                text: `${Math.round(d.percent[type] * 100)}%`
              });
            }
            x += value;
          }
          return labels;
        }),
        {
          x: "x",
          y: "year",
          text: "text",
          fill: "black",
          textAnchor: "middle"
        }
      ),

      // 百分比文字标签（法国）
      Plot.text(
        french_summary.flatMap(d => {
          const labels = [];
          let x = 0;
          for (const type of ["comedy", "drama", "ballet", "other"]) {
            const value = d[type];
            if (value > 0) {
              labels.push({
                year: d.year,
                x: x + value / 2,
                text: `${Math.round(d.percent[type] * 100)}%`
              });
            }
            x += value;
          }
          return labels;
        }),
        {
          x: "x",
          y: "year",
          text: "text",
          fill: "black",
          textAnchor: "middle"
        }
      )
    ]
  })
);
```
