# Custom Visualizations in Apache Superset

A practical guide to how custom visuals are built in Superset, how far you can customize, and where the limits are. Written against **Superset 6.0**. Customization APIs change between major versions, so confirm details in the official docs (superset.apache.org) for your version.

---

## 1. The approach: escalate only when needed

Customization in Superset works like a ladder. Each step up gives more control but costs more effort and makes upgrades harder. The rule is **start at the lowest level that solves the problem**.

| Level | What you use | Effort | Needs code? | Survives upgrades? |
|---|---|---|---|---|
| 1 | Built-in chart settings (Explore) | Minutes | No | Yes |
| 2 | Dataset layer: virtual datasets, metrics, Jinja | Minutes–hours | SQL only | Yes |
| 3 | Dashboard styling: layout, CSS, colors, theming | Hours | CSS/JSON | Mostly (CSS can break) |
| 4 | Handlebars chart (HTML templates) | Hours | HTML/CSS | Yes |
| 5 | Server config (`superset_config.py`) | Hours | Python config | Mostly |
| 6 | Custom viz plugin (React + TypeScript) | Days | Yes | No, needs maintenance |

### Decision flow

```
Can a built-in chart show it with the right metrics/format?
 ├─ Yes → Level 1 (+ Level 2 if the data needs reshaping)
 └─ No → Is it mostly "styled cards / custom HTML layout" with no interactivity?
          ├─ Yes → Level 4 (Handlebars)
          └─ No → Is it only about look & feel (colors, fonts, spacing)?
                   ├─ Yes → Level 3 / Level 5
                   └─ No → Level 6 (custom plugin), only if it will be reused
```

A useful insight: **most "I need a custom chart" problems are really data-shaping problems.** Reshape the data in SQL (Level 2) and a built-in chart often works.

---

## 2. Level 1: Built-in chart configuration

Superset ships 40+ chart types (mostly Apache ECharts under the hood), such as Big Number, Time-series Line/Bar/Area, Pivot Table, Table, Funnel, Sankey, Heatmap, Treemap, Sunburst, Box Plot, Gauge, Radar, Graph, Word Cloud, Calendar Heatmap, and deck.gl geospatial maps.

What you can control in the Explore view:

- **Metrics:** simple aggregates, or **Custom SQL metrics** like `SUM(amount) FILTER (WHERE stage = 'disbursed')`.
- **Formatting:** D3 number formats (`,.0f`, `.1%`, `.3s`), date formats, currency, and axis bounds.
- **Advanced analytics** on time-series charts: rolling windows (7-day average), **time comparison** (vs. last week or year, as absolute, % change, or ratio), resampling, and cumulative sums.
- **Annotation layers:** mark events such as "April scheme launch" on time-series charts.
- **Color schemes**, legends, labels, tooltips, and sorting.
- **Contribution mode:** show each series as a percentage of the total.

**Example (PL journey):** conversion trend by channel, compared with the previous period.

- Chart: Line Chart, X-axis `created_at` (Week), Dimension `channel`
- Metric (Custom SQL): `SUM(CASE WHEN stage='disbursed' THEN 1 ELSE 0 END)*1.0/COUNT(*)`
- Advanced analytics → Time comparison: `1 week ago`, calculation type `Percentage change`

---

## 3. Level 2: The dataset layer

This is the most powerful no-plugin lever.

### Virtual datasets
A saved SQL query that behaves like a table. You can reshape data into exactly what a built-in chart expects.

**Example:** a true funnel counts applications that *reached at least* each stage, not just their current stage.

```sql
WITH s AS (
  SELECT *, CASE stage WHEN 'lead' THEN 1 WHEN 'prequalified' THEN 2
    WHEN 'applied' THEN 3 WHEN 'sanctioned' THEN 4 WHEN 'disbursed' THEN 5 END AS ord
  FROM loan_applications
)
SELECT st.stage_name, st.ord, COUNT(*) AS reached
FROM s
JOIN (VALUES ('lead',1),('prequalified',2),('applied',3),
             ('sanctioned',4),('disbursed',5)) AS st(stage_name, ord)
  ON s.ord >= st.ord
GROUP BY st.stage_name, st.ord
ORDER BY st.ord;
```

The built-in **Funnel** chart now renders a correct funnel, with no custom code.

### Calculated columns and saved metrics
Define reusable logic once on the dataset, such as `conversion_rate`, `amount_bucket`, or `ticket_size_band`, so every chart stays consistent.

### Jinja templating
Makes SQL dynamic:

```sql
SELECT * FROM loan_applications
WHERE 1=1
{% if filter_values('channel') %}
  AND channel IN {{ filter_values('channel') | where_in }}
{% endif %}
{% if from_dttm %} AND created_at >= '{{ from_dttm }}' {% endif %}
```

Useful macros include `filter_values()`, `get_filters()`, `current_username()`, `url_param()`, `from_dttm`/`to_dttm`, and `dataset()`.

Jinja must be enabled with the `ENABLE_TEMPLATE_PROCESSING` feature flag.

---

## 4. Level 3: Dashboard-level styling

- **Layout:** grid rows and columns, **tabs**, headers, dividers, and **Markdown** blocks (sanitized HTML is allowed).
- **Interactivity:** native filters, dependent filters, cross-filtering, drill by, and drill to detail.
- **Custom CSS** (Edit dashboard → ⋯ → Edit CSS), saved as reusable **CSS templates**:

```css
/* Tighter, branded dashboard */
.dashboard-header .header-title { font-weight: 700; color: #ED1C24; }
.dashboard-component-chart-holder { border-radius: 12px; box-shadow: 0 1px 4px rgba(0,0,0,.08); }
```

- **Consistent label colors:** in Edit properties → Advanced → JSON metadata:

```json
{
  "label_colors": {
    "Web": "#1f77b4",
    "App": "#2ca02c",
    "DSA": "#ff7f0e",
    "Branch": "#9467bd"
  }
}
```

- **Theming (6.0):** app-wide theming built on Ant Design design tokens (colors, fonts, border radius, light/dark). It's configured via `THEME_DEFAULT` / `THEME_DARK` in `superset_config.py`, and newer versions can also manage it through the UI. Check your version's docs for the exact keys.

> **Caution:** CSS depends on internal class names, which can change between releases. Keep dashboard CSS small and re-test after upgrades.

---

## 5. Level 4: Handlebars chart (custom HTML without a plugin)

The **Handlebars** chart runs your query and passes the rows to an HTML template. It's ideal for KPI cards, scorecards, styled leaderboards, and custom tables.

**Setup:** Chart type → Handlebars. Query mode: Aggregate or Raw. Group by `region`, metrics `COUNT(*)` and `conversion_rate`.

**Template:**

```handlebars
<div class="grid">
  {{#each data}}
    <div class="card">
      <div class="label">{{this.region}}</div>
      <div class="value">{{this.[COUNT(*)]}}</div>
      <div class="sub">Conversion: {{this.conversion_rate}}</div>
    </div>
  {{/each}}
</div>
```

**CSS (same chart, CSS box):**

```css
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(160px, 1fr)); gap: 12px; }
.card { padding: 16px; border-radius: 10px; background: #f7f7f9; }
.label { font-size: 12px; color: #666; text-transform: uppercase; }
.value { font-size: 28px; font-weight: 700; }
.sub { font-size: 12px; color: #2a7; }
```

Handlebars helpers such as date formatting and JSON stringify are available. The exact set varies by version.

**Limits:** no JavaScript, no click interactions or cross-filtering, and HTML is sanitized. It only handles presentation.

---

## 6. Level 5: Server configuration (`superset_config.py`)

Global customization, no frontend build required:

```python
# Brand color palette available in every chart's color picker
EXTRA_CATEGORICAL_COLOR_SCHEMES = [{
    "id": "kotak_brand",
    "description": "Brand palette",
    "label": "Brand",
    "isDefault": True,
    "colors": ["#ED1C24", "#003874", "#F7A600", "#6D6E71", "#00A3AD"],
}]

# Indian number/currency formatting defaults
D3_FORMAT = {
    "decimal": ".",
    "thousands": ",",
    "grouping": [3, 2, 2, 2, 2, 2, 2, 2, 2],   # 1,00,00,000 style
    "currency": ["₹", ""],
}

# Custom Jinja macros usable in any dataset SQL
def fiscal_year_start():
    return "2026-04-01"

JINJA_CONTEXT_ADDONS = {"fiscal_year_start": fiscal_year_start}

FEATURE_FLAGS = {
    "ENABLE_TEMPLATE_PROCESSING": True,
    "DASHBOARD_CROSS_FILTERS": True,
}
```

Other levers include custom time grains (`TIME_GRAIN_ADDONS`), extra D3 formats, a custom security manager, and HTML sanitization rules.

**deck.gl JavaScript controls:** map charts can accept a JavaScript "data interceptor" for tooltips and data transforms if `ENABLE_JAVASCRIPT_CONTROLS = True`. This lets chart authors run arbitrary JS in viewers' browsers, so **keep it off anywhere untrusted users can create charts.**

With the Docker Compose setup, local overrides go in `docker/pythonpath_dev/superset_config_docker.py`. Restart the containers after changes.

---

## 7. Level 6: Custom visualization plugin

This is the "real" custom chart. It's a React component that appears in Superset's chart picker, uses the Explore control panel, and receives query results from Superset.

### 7.1 How a plugin is structured

```
superset-plugin-chart-stage-funnel/
├── src/
│   ├── index.ts              # Registers the plugin (metadata + wiring)
│   ├── StageFunnel.tsx       # The React component that draws the chart
│   ├── plugin/
│   │   ├── buildQuery.ts     # Form controls → query sent to backend
│   │   ├── controlPanel.ts   # Controls shown in the Explore sidebar
│   │   └── transformProps.ts # Backend result → props for the component
│   └── images/thumbnail.png
└── package.json
```

The data flow:

```
Explore controls (controlPanel)
      → buildQuery → Superset backend → SQL on your DB
      → queriesData → transformProps → <StageFunnel /> renders
```

### 7.2 Scaffold it

Do this inside **WSL Ubuntu**, not PowerShell. The frontend toolchain is far smoother on Linux.

```bash
# Use the Node version pinned in superset-frontend/.nvmrc
cd superset/superset-frontend/packages/generator-superset
npm install && npm link
npm install -g yo

mkdir ~/superset-plugin-chart-stage-funnel && cd ~/superset-plugin-chart-stage-funnel
yo @superset-ui/superset
npm install
```

### 7.3 Example code: a stage-to-stage conversion funnel

**`src/plugin/controlPanel.ts`**

```ts
import { t } from '@superset-ui/core';
import { ControlPanelConfig, sharedControls } from '@superset-ui/chart-controls';

const config: ControlPanelConfig = {
  controlPanelSections: [
    {
      label: t('Query'),
      expanded: true,
      controlSetRows: [
        [{ name: 'groupby', config: { ...sharedControls.groupby, label: t('Stage column') } }],
        ['metric'],
        ['adhoc_filters'],
        ['row_limit'],
      ],
    },
    {
      label: t('Options'),
      expanded: true,
      controlSetRows: [
        [{
          name: 'show_step_conversion',
          config: {
            type: 'CheckboxControl',
            label: t('Show stage-to-stage conversion %'),
            default: true,
            renderTrigger: true, // re-renders without re-querying
          },
        }],
      ],
    },
  ],
};
export default config;
```

**`src/plugin/buildQuery.ts`**

```ts
import { buildQueryContext, QueryFormData } from '@superset-ui/core';

export default function buildQuery(formData: QueryFormData) {
  return buildQueryContext(formData, baseQueryObject => [{ ...baseQueryObject }]);
}
```

**`src/plugin/transformProps.ts`**

```ts
import { ChartProps, getMetricLabel } from '@superset-ui/core';

export default function transformProps(chartProps: ChartProps) {
  const { width, height, formData, queriesData } = chartProps;
  const rows = queriesData[0].data as Record<string, any>[];
  const metric = getMetricLabel(formData.metric);
  const stageCol = formData.groupby[0];

  return {
    width,
    height,
    showStepConversion: formData.showStepConversion,
    stages: rows.map(r => ({ stage: r[stageCol], value: Number(r[metric]) })),
  };
}
```

**`src/StageFunnel.tsx`**

```tsx
import React from 'react';

type Props = {
  width: number;
  height: number;
  showStepConversion: boolean;
  stages: { stage: string; value: number }[];
};

export default function StageFunnel({ width, height, stages, showStepConversion }: Props) {
  const max = Math.max(...stages.map(s => s.value), 1);
  const rowH = height / Math.max(stages.length, 1);

  return (
    <div style={{ width, height, fontFamily: 'inherit' }}>
      {stages.map((s, i) => {
        const pct = i > 0 && stages[i - 1].value ? (s.value / stages[i - 1].value) * 100 : null;
        return (
          <div key={s.stage} style={{ height: rowH, display: 'flex', alignItems: 'center', gap: 8 }}>
            <div style={{ width: 110, fontSize: 12 }}>{s.stage}</div>
            <div style={{ flex: 1 }}>
              <div style={{
                width: `${(s.value / max) * 100}%`, height: rowH * 0.6,
                background: '#003874', borderRadius: 6, margin: '0 auto',
              }} />
            </div>
            <div style={{ width: 130, fontSize: 12, textAlign: 'right' }}>
              {s.value.toLocaleString('en-IN')}
              {showStepConversion && pct !== null && (
                <span style={{ color: pct < 50 ? '#c0392b' : '#27ae60' }}> ({pct.toFixed(1)}%)</span>
              )}
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

**`src/index.ts`**

```ts
import { ChartMetadata, ChartPlugin } from '@superset-ui/core';
import buildQuery from './plugin/buildQuery';
import controlPanel from './plugin/controlPanel';
import transformProps from './plugin/transformProps';
import thumbnail from './images/thumbnail.png';

export default class StageFunnelChartPlugin extends ChartPlugin {
  constructor() {
    super({
      buildQuery,
      controlPanel,
      transformProps,
      loadChart: () => import('./StageFunnel'),
      metadata: new ChartMetadata({
        name: 'Stage Conversion Funnel',
        description: 'Funnel with stage-to-stage conversion percentages',
        thumbnail,
      }),
    });
  }
}
```

> Import paths and package names follow the generator's current template. If the generator output differs from the snippets above, follow the generator.

### 7.4 Register and run

1. Install the plugin into the frontend:
   ```bash
   cd superset/superset-frontend
   npm install -S ~/superset-plugin-chart-stage-funnel
   ```
2. Register it in `superset-frontend/src/visualizations/presets/MainPreset.js`:
   ```js
   import StageFunnelChartPlugin from 'superset-plugin-chart-stage-funnel';
   // inside plugins: [ ... ]
   new StageFunnelChartPlugin().configure({ key: 'ext-stage-funnel' }),
   ```
3. **Development:** run the frontend dev server (`npm run dev-server`) against the backend on port 8088, with hot reload at `localhost:9000`.
4. **Deployment:** build **your own Docker image** from the repo's `Dockerfile` so the compiled frontend includes the plugin, then point your compose file or Helm values at that image.

> **Important:** `docker-compose-image-tag.yml` (the setup you're running) uses a **prebuilt official image**. You **cannot** add a plugin to it. Plugins require building the frontend yourself; Section 8 walks through exactly that.

---

## 8. Worked example: building a chart inside Superset's source code

Section 7 treated the plugin as a separate npm package. This section does something more direct: **we add a new chart type straight into the Superset source tree, rebuild Superset, and use the chart on a real dashboard.** Every step is shown, from an empty folder to a clickable chart.

### 8.1 What we're building and why

**Chart:** *Target vs Actual (Bullet)*, a monthly disbursal attainment view per region.

Each row shows:

- a background track split into **red / amber / green bands** relative to that region's target,
- a **dark bar** for the actual disbursed amount,
- a **black tick** at the target,
- a label like `₹10.4 Cr / ₹12.0 Cr (87%)`, colored by band.

Clicking a row **cross-filters** the rest of the dashboard to that region.

**Why this needs a custom chart:** no built-in Superset chart combines *per-row targets*, *configurable attainment bands*, *crore formatting*, and *cross-filtering* in one compact view. Gauges show one number at a time, and bar charts can't draw per-row target ticks with bands.

### 8.2 How the pieces fit together

Before touching code, it helps to know what "adding a chart" actually changes.

```
┌────────────────────── superset-frontend (we change this) ──────────────────────┐
│ src/visualizations/BulletTarget/   ← new folder: our chart                      │
│ src/visualizations/presets/MainPreset  ← one line: register chart as 'bullet_target' │
└─────────────────────────────────────────────────────────────────────────────────┘
            │  webpack build → static JS bundled into the Docker image
            ▼
Browser: Explore shows controls → buildQuery → POST /api/v1/chart/data
            ▼
Superset backend (unchanged Python) → SQL on Postgres → rows
            ▼
transformProps → <BulletTarget/> draws SVG → click → setDataMask → cross-filter
```

Three things are worth understanding:

1. **No Python changes are needed.** Modern charts send a generic query through the chart data API, so the backend doesn't need to know the chart exists.
2. **The key `bullet_target` is permanent.** When you save a chart, Superset stores `viz_type: "bullet_target"` in its metadata database. If you rename the key later, every saved chart using it breaks.
3. **The frontend is compiled into the image.** That's why the prebuilt image you're running now can never show this chart. A saved `bullet_target` chart opened on the stock image fails with an error that the chart type isn't registered. You need a rebuilt image.

### 8.3 Step 1: Set up a workspace in WSL

Building the frontend from PowerShell on `C:\` is slow and error-prone. Do it in the Ubuntu you installed. Instead of re-downloading 1 GB, clone from your existing Windows copy:

```bash
# In Ubuntu (WSL)
git clone "/mnt/c/Users/shwet/Desktop/apache superset/superset" ~/superset
cd ~/superset
git checkout tags/6.0.0
git switch -c feature/bullet-target-chart     # your own branch for the change
```

Enable Docker inside Ubuntu: **Docker Desktop → Settings → Resources → WSL integration → turn on Ubuntu → Apply**. Check it with `docker ps` in Ubuntu.

Install the Node version Superset expects:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
cd ~/superset/superset-frontend
nvm install          # reads .nvmrc
nvm use
npm ci               # 5–15 minutes, installs the frontend dependencies
```

If `npm ci` or the build later crashes with "JavaScript heap out of memory", give WSL more RAM. Create `C:\Users\shwet\.wslconfig` with:

```ini
[wsl2]
memory=10GB
```

Then run `wsl --shutdown` in PowerShell and reopen Ubuntu.

### 8.4 Step 2: Give the chart real data (targets table)

Add a targets table to the demo database from the earlier guide:

```bash
docker exec -it pl-demo-db psql -U demo -d loans
```

```sql
CREATE TABLE disbursal_targets AS
SELECT r AS region,
       m::date AS month,
       round((80000000 + random() * 50000000)::numeric, -6) AS target_amount
FROM unnest(ARRAY['North','South','East','West']) AS r,
     generate_series(date_trunc('month', now()) - INTERVAL '3 months',
                     date_trunc('month', now()), INTERVAL '1 month') AS m;
```

In Superset's **SQL Lab**, save this as a virtual dataset named `region_target_vs_actual`:

```sql
SELECT
  a.region,
  DATE_TRUNC('month', a.created_at)::date                         AS month,
  SUM(a.amount) FILTER (WHERE a.stage = 'disbursed')              AS actual_amount,
  MAX(t.target_amount)                                            AS target_amount
FROM loan_applications a
JOIN disbursal_targets t
  ON t.region = a.region
 AND t.month  = DATE_TRUNC('month', a.created_at)::date
GROUP BY 1, 2
```

Open the dataset settings and mark `month` as **temporal** (Is temporal), so time-range filters work on it.

### 8.5 Step 3: Write the chart (the source-code change)

Create the folder:

```bash
cd ~/superset/superset-frontend
mkdir -p src/visualizations/BulletTarget/images
# Any existing thumbnail works as a placeholder
cp "$(find plugins -name thumbnail.png | head -1)" src/visualizations/BulletTarget/images/thumbnail.png
```

> Package names like `@superset-ui/core` are what 6.0 uses at the time of writing. Before you start, open a neighbouring file in `src/visualizations/` and mirror its import lines if they differ.

#### `types.ts`: the contract between data and drawing

```ts
export type BulletRow = {
  category: string;
  actual: number;
  target: number;
  attainment: number; // actual / target * 100
};

export type BulletTargetProps = {
  width: number;
  height: number;
  rows: BulletRow[];
  categoryCol: string;
  poorBelow: number;
  goodAbove: number;
  selected: string[];
  emitCrossFilters?: boolean;
  setDataMask?: (mask: any) => void;
};
```

#### `controlPanel.ts`: what the user sees in Explore

Two separate metric pickers, *Actual* and *Target*, instead of Superset's usual single "Metrics" box. This is what makes the chart feel purpose-built.

```ts
import { t, validateNonEmpty } from '@superset-ui/core';
import { ControlPanelConfig, sharedControls } from '@superset-ui/chart-controls';

const config: ControlPanelConfig = {
  controlPanelSections: [
    {
      label: t('Query'),
      expanded: true,
      controlSetRows: [
        [{
          name: 'groupby',
          config: {
            ...sharedControls.groupby,
            label: t('Category (e.g. region)'),
            multi: false,
            validators: [validateNonEmpty],
          },
        }],
        [{ name: 'actual_metric', config: { ...sharedControls.metric, label: t('Actual') } }],
        [{ name: 'target_metric', config: { ...sharedControls.metric, label: t('Target') } }],
        ['adhoc_filters'],
        ['row_limit'],
      ],
    },
    {
      label: t('Attainment bands'),
      expanded: true,
      controlSetRows: [
        [{
          name: 'poor_below',
          config: {
            type: 'TextControl', isInt: true, default: 70, renderTrigger: true,
            label: t('Red below (% of target)'),
          },
        }],
        [{
          name: 'good_above',
          config: {
            type: 'TextControl', isInt: true, default: 90, renderTrigger: true,
            label: t('Green from (% of target)'),
          },
        }],
        [{
          name: 'sort_by_attainment',
          config: {
            type: 'CheckboxControl', default: true, renderTrigger: true,
            label: t('Worst performers first'),
          },
        }],
      ],
    },
  ],
};

export default config;
```

`renderTrigger: true` means changing the band thresholds redraws the chart instantly **without re-running SQL**. Only the query section hits the database.

#### `buildQuery.ts`: the subtle part

Superset's default query builder only collects metrics from standard control names (`metric`, `metrics`, and so on). It **ignores** custom names like `actual_metric`. If you skip this file's logic, the chart runs a query with no metrics and gets back empty numbers. So we pass both metrics explicitly:

```ts
import { buildQueryContext, QueryFormData } from '@superset-ui/core';

export default function buildQuery(formData: QueryFormData) {
  const { actual_metric, target_metric, groupby = [] } = formData;

  return buildQueryContext(formData, baseQueryObject => [
    {
      ...baseQueryObject,
      columns: groupby,
      metrics: [actual_metric, target_metric],
    },
  ]);
}
```

#### `transformProps.ts`: turn raw rows into chart-ready rows

```ts
import { ChartProps, getColumnLabel, getMetricLabel } from '@superset-ui/core';
import { BulletRow, BulletTargetProps } from './types';

export default function transformProps(chartProps: ChartProps): BulletTargetProps {
  const { width, height, formData, queriesData, hooks, filterState, emitCrossFilters } = chartProps;
  // formData keys arrive camelCased: actual_metric → actualMetric
  const { groupby, actualMetric, targetMetric, poorBelow, goodAbove, sortByAttainment } = formData;

  const categoryCol = getColumnLabel(groupby[0]);
  const actualKey = getMetricLabel(actualMetric);
  const targetKey = getMetricLabel(targetMetric);

  const rows: BulletRow[] = (queriesData[0]?.data ?? []).map((r: Record<string, any>) => {
    const actual = Number(r[actualKey]) || 0;
    const target = Number(r[targetKey]) || 0;
    return {
      category: String(r[categoryCol]),
      actual,
      target,
      attainment: target > 0 ? (actual / target) * 100 : 0,
    };
  });

  if (sortByAttainment) rows.sort((a, b) => a.attainment - b.attainment);

  return {
    width,
    height,
    rows,
    categoryCol,
    poorBelow: Number(poorBelow),
    goodAbove: Number(goodAbove),
    selected: filterState?.selectedValues ?? [],
    emitCrossFilters,
    setDataMask: hooks?.setDataMask,
  };
}
```

#### `BulletTarget.tsx`: the drawing, in plain SVG

No charting library. SVG keeps the bundle small and gives full control.

```tsx
import React from 'react';
import { BulletTargetProps } from './types';

const LABEL_W = 80;
const VALUE_W = 190;
const BAND = { red: '#F5B7B1', amber: '#FAD7A0', green: '#ABEBC6' };
const TEXT = { red: '#B03A2E', amber: '#B9770E', green: '#1E8449' };

// Indian banking view: show crores, not millions
const crore = (v: number) => `₹${(v / 1e7).toFixed(1)} Cr`;

export default function BulletTarget({
  width, height, rows, categoryCol, poorBelow, goodAbove,
  selected, emitCrossFilters, setDataMask,
}: BulletTargetProps) {
  if (!rows.length) {
    return <div style={{ padding: 16 }}>No data for the selected filters.</div>;
  }

  const rowH = Math.min(52, height / rows.length);
  const trackW = Math.max(width - LABEL_W - VALUE_W, 60);
  const scaleMax = Math.max(...rows.map(r => Math.max(r.actual, r.target * 1.15)), 1);
  const x = (v: number) => Math.min((v / scaleMax) * trackW, trackW);

  const toggleFilter = (category: string) => {
    if (!emitCrossFilters || !setDataMask) return;
    const alreadySelected = selected.includes(category);
    setDataMask(
      alreadySelected
        ? { extraFormData: {}, filterState: { value: null, selectedValues: null } }
        : {
            extraFormData: { filters: [{ col: categoryCol, op: 'IN', val: [category] }] },
            filterState: { value: [category], selectedValues: [category] },
          },
    );
  };

  return (
    <svg width={width} height={rowH * rows.length} role="img" aria-label="Target vs actual">
      {rows.map((r, i) => {
        const y = i * rowH;
        const barH = rowH * 0.55;
        const top = y + (rowH - barH) / 2;
        const band = r.attainment < poorBelow ? 'red' : r.attainment < goodAbove ? 'amber' : 'green';
        const dimmed = selected.length > 0 && !selected.includes(r.category);

        const redEnd = x((r.target * poorBelow) / 100);
        const amberEnd = x((r.target * goodAbove) / 100);

        return (
          <g
            key={r.category}
            opacity={dimmed ? 0.35 : 1}
            style={{ cursor: emitCrossFilters ? 'pointer' : 'default' }}
            onClick={() => toggleFilter(r.category)}
          >
            <title>{`${r.category}: ${r.attainment.toFixed(1)}% of target`}</title>

            <text x={0} y={y + rowH / 2} dominantBaseline="middle" fontSize={12} fontWeight={600}>
              {r.category}
            </text>

            <g transform={`translate(${LABEL_W},0)`}>
              {/* Qualitative bands, relative to THIS row's target */}
              <rect x={0} y={top} width={redEnd} height={barH} fill={BAND.red} />
              <rect x={redEnd} y={top} width={amberEnd - redEnd} height={barH} fill={BAND.amber} />
              <rect x={amberEnd} y={top} width={trackW - amberEnd} height={barH} fill={BAND.green} />

              {/* Actual */}
              <rect x={0} y={top + barH * 0.3} width={x(r.actual)} height={barH * 0.4} fill="#1B2631" rx={2} />

              {/* Target tick */}
              <line x1={x(r.target)} x2={x(r.target)} y1={top - 4} y2={top + barH + 4}
                    stroke="#000" strokeWidth={2.5} />
            </g>

            <text x={width - 4} y={y + rowH / 2} textAnchor="end" dominantBaseline="middle"
                  fontSize={12} fill={TEXT[band]}>
              {`${crore(r.actual)} / ${crore(r.target)} (${r.attainment.toFixed(0)}%)`}
            </text>
          </g>
        );
      })}
    </svg>
  );
}
```

#### `index.ts`: package it as a Superset chart

```ts
import { Behavior, ChartMetadata, ChartPlugin, t } from '@superset-ui/core';
import buildQuery from './buildQuery';
import controlPanel from './controlPanel';
import transformProps from './transformProps';
import thumbnail from './images/thumbnail.png';

export default class BulletTargetChartPlugin extends ChartPlugin {
  constructor() {
    super({
      buildQuery,
      controlPanel,
      transformProps,
      loadChart: () => import('./BulletTarget'),   // lazy-loaded: no cost until used
      metadata: new ChartMetadata({
        name: t('Target vs Actual (Bullet)'),
        description: t('Attainment against per-category targets with red/amber/green bands.'),
        category: t('KPI'),
        tags: [t('Business'), t('Comparison')],
        thumbnail,
        behaviors: [Behavior.InteractiveChart],   // tells dashboards it can emit cross-filters
      }),
    });
  }
}
```

Final folder:

```
src/visualizations/BulletTarget/
├── BulletTarget.tsx
├── buildQuery.ts
├── controlPanel.ts
├── images/thumbnail.png
├── index.ts
├── transformProps.ts
└── types.ts
```

### 8.6 Step 4: Register the chart (the one-line core change)

Open `src/visualizations/presets/MainPreset` (`.js` or `.ts`, depending on version). Add the import at the top and an entry in the `plugins` array:

```js
import BulletTargetChartPlugin from '../BulletTarget';

// ...inside plugins: [
  new BulletTargetChartPlugin().configure({ key: 'bullet_target' }),
// ]
```

That's the entire change to Superset's own code. Everything else is new files. Keeping core edits this small matters when you upgrade later.

Type-check before running:

```bash
npx tsc --noEmit -p .
```

### 8.7 Step 5: Try it with the dev server (fast feedback loop)

Keep your existing Superset containers running (backend on port 8088). In Ubuntu:

```bash
cd ~/superset/superset-frontend
npm run dev-server
```

When webpack says it compiled, open **http://localhost:9000**. It's the same Superset and the same login, but served with *your* frontend, and it hot-reloads when you save a file.

Now **use the chart**:

1. **Charts → + Chart** → dataset `region_target_vs_actual` → **View all charts** → search **"Target vs Actual"** → Create.
2. Set **Category** to `region`.
3. Set **Actual** to `SUM(actual_amount)`, and **Target** to `SUM(target_amount)`.
4. Set **Time range** to *Current month* (with `month` as the time column).
5. Click **Update chart**. You'll see four rows, worst region first.
6. Change **Red below** from 70 to 85. The colors update instantly with no new query (that's `renderTrigger`).
7. **Save** → add to the *PL Funnel – Demo* dashboard.
8. On the dashboard, make sure cross-filtering is enabled, then **click the "North" row**. The other charts (weekly applications, region table) filter to North. Click it again to clear.

Try editing `BAND.green` in `BulletTarget.tsx` and save. The chart repaints within seconds. This loop is where most of the real development time goes.

### 8.8 Step 6: Build your own Superset image and switch to it

The dev server is only for development. To have the chart at `localhost:8088` permanently, bake it into an image:

```bash
cd ~/superset
docker build --target lean -t superset-custom:6.0.0-bullet .
```

The first build takes **20–40 minutes**, since it compiles the entire frontend.

Point Compose at your image. In `~/superset/docker-compose-image-tag.yml`, find the line near the top that defines the Superset image (it contains `apache/superset`) and replace the image reference with:

```yaml
superset-custom:6.0.0-bullet
```

Restart the stack:

```bash
# In PowerShell, stop the old stack (Ctrl+C in its window), then in Ubuntu:
cd ~/superset
docker compose -f docker-compose-image-tag.yml up
```

Both folders are named `superset`, so Compose reuses the same data volumes. Your dashboards, datasets, and the saved bullet chart are still there. Open **http://localhost:8088**, and the dashboard now renders the custom chart without the dev server.

### 8.9 Step 7: Keep it maintainable

```bash
cd ~/superset
git add superset-frontend/src/visualizations/BulletTarget superset-frontend/src/visualizations/presets
git commit -m "Add Target vs Actual bullet chart (bullet_target)"
```

When a new Superset version comes out:

```bash
git fetch --tags
git rebase --onto tags/<new-version> tags/6.0.0 feature/bullet-target-chart
npx tsc --noEmit -p superset-frontend    # catches API changes early
```

Then rebuild the image with a new tag and test the dashboard before switching.

**Checklist for a custom chart in production:**

- [ ] `viz_type` key (`bullet_target`) never renamed
- [ ] Core change limited to the preset registration line
- [ ] Type-check passes after each upgrade
- [ ] Image tag includes both Superset version and chart version
- [ ] Dashboard exported (ZIP) and stored in Git alongside the code
- [ ] Old stock image never used against a metadata DB containing `bullet_target` charts

### 8.10 Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Chart missing from the chart picker | Not registered, or old bundle cached | Check the preset entry; hard refresh (Ctrl+Shift+R) |
| Error that `bullet_target` is not registered | Running the stock image or old build | Use `localhost:9000` (dev) or rebuild the image |
| Chart renders but all values are 0 | `buildQuery` didn't pass custom metrics | Confirm `metrics: [actual_metric, target_metric]` |
| Values `undefined` in `transformProps` | Wrong key casing | Use camelCase in `formData` (`actualMetric`) |
| Clicking a row does nothing | Cross-filtering disabled, or missing `Behavior.InteractiveChart` | Enable in dashboard properties; check metadata |
| Build killed / heap out of memory | WSL RAM too low | Raise `memory` in `.wslconfig`; `export NODE_OPTIONS=--max-old-space-size=8192` |
| `npm ci` extremely slow | Repo is on `/mnt/c/...` | Work from the `~/superset` clone inside WSL |

---

## 9. What you can and cannot customize

### You can

- Shape any data into any chart via SQL, virtual datasets, and Jinja
- Define org-wide metrics, color palettes, number formats (including Indian grouping), and time grains
- Style dashboards with CSS, tabs, markdown, and theming
- Build HTML card layouts without code (Handlebars)
- Build any React/D3/ECharts visual as a plugin that integrates with Explore controls, filters, and caching
- Embed dashboards in your own React app with row-level security per user

### Limitations

| Area | Limitation | Workaround |
|---|---|---|
| Plugins | Require a frontend rebuild and custom Docker image; not a drop-in upload | Maintain a small fork/build pipeline |
| Upgrades | Plugin APIs and internal class names change across major versions (e.g. UI library migrations) | Pin versions, test plugins before upgrading |
| Handlebars | No JavaScript, no interactions, sanitized HTML | Move to a plugin if interaction is needed |
| CSS | Targets unstable internal selectors; no per-chart CSS in most chart types | Keep CSS minimal; prefer theming/config |
| ECharts options | Most raw ECharts options aren't exposed in the UI | Plugin, or reshape data |
| Cross-filtering | Only supported by some chart types; custom plugins must implement it themselves | Use native filters |
| Data volume | Charts render in the browser; row limits apply; huge result sets are slow | Aggregate in SQL, use caching |
| Printing/reports | Not pixel-perfect; PDF/PNG exports are screenshots | Use a reporting tool for statutory formats |
| Write-back | Read-only analytics; no forms, approvals, or data entry | Build that part in your own app |
| Custom logic | No arbitrary cross-chart JS workflows (e.g. "click → call an API") | Embed Superset in a custom app |
| Mobile | Dashboards work on mobile but layouts aren't truly responsive-designed | Separate simplified dashboard |
| External data | Charts query configured databases only; no calling REST APIs from charts | Load API data into a table first |
| JS controls | deck.gl JavaScript controls are a security risk if enabled | Enable only for trusted authors |

---

## 10. When Superset is the wrong tool

Superset works best for **internal analytics and monitoring**. Consider a custom React/FastAPI app (or keep your existing one) when you need:

- Highly bespoke UX or workflows (drill-through into case-level actions, approvals)
- Write-back or operational screens
- Pixel-perfect, branded, customer-facing views
- Complex client-side interactivity across many components

A common hybrid is **Superset for exploration and standard dashboards**, with dashboards **embedded** into a custom app that handles the bespoke parts.

---

## 11. Recommended learning path

1. Build the PL funnel dashboard using only Levels 1–2 (built-in charts and virtual datasets).
2. Add a brand color scheme and Indian number formatting (Level 5).
3. Build a regional KPI card grid with Handlebars (Level 4).
4. Add dashboard CSS and label colors (Level 3).
5. Scaffold the Stage Conversion Funnel plugin (Level 6) in WSL, run it with the dev server, and compare it with the built-in Funnel chart.
6. Build the Target vs Actual bullet chart inside the Superset source (Section 8), bake it into your own image, and use it with cross-filtering on the PL dashboard.
