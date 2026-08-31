# TypeScript Learning Plan — React Dashboard (Vite)

Built for someone starting from zero in JavaScript, working toward a segmented dashboard (MIS / Insights / Actionable) in React.

**The one rule:** don't learn TypeScript before JavaScript. TypeScript is JavaScript plus a type layer. If the JS underneath is shaky, every type error will feel like magic instead of a message. Phase 0 is not optional.

---

## Phase 0 — JavaScript foundations (~1 week)

Learn these in JS first, with no types at all. Open the browser console and try each one.

| Topic | Why it matters for your dashboard |
|---|---|
| `let` / `const`, primitives | Every variable you write |
| Objects and arrays | An API row is an object; a table is an array of them |
| Functions, arrow functions | Components are functions |
| `map`, `filter`, `reduce`, `find`, `sort` | This *is* dashboard work — grouping, totals, filtering rows |
| Destructuring, spread (`...`) | React props and state updates depend on it |
| Template literals | Building labels and display strings |
| Optional chaining `?.` and `??` | API fields that might be missing |
| Promises, `async` / `await` | Fetching data |
| ES modules (`import` / `export`) | How files connect |
| `this` — just enough to avoid it | Modern React barely needs it |

**Milestone:** write a plain `.js` file that takes a hardcoded array of 20 loan records and prints total outstanding, count by status, and the 5 largest.

---

## Phase 1 — TypeScript core (~1 week)

### Basic annotations
```ts
const branchName: string = "Andheri";
const overdueCount: number = 42;
const isActive: boolean = true;
const tags: string[] = ["retail", "secured"];
```

### `type` and `interface`
Your most-used feature. Model one API row:
```ts
type LoanAccount = {
  accountId: string;
  customerName: string;
  outstanding: number;
  dpd: number;               // days past due
  branch: string;
  lastPaymentDate: string | null;
};
```
Use `type` by default. `interface` is nearly identical — pick one and stay consistent.

### Union types and literal types
```ts
type Bucket = "0-30" | "31-60" | "61-90" | "90+";
type Segment = "MIS" | "Insights" | "Actionable";
```
This is where TypeScript starts paying you back — typos become errors.

### Optional and readonly
```ts
type Filters = {
  branch?: string;           // may be absent
  readonly reportDate: string;
};
```

### Function types
```ts
function totalOutstanding(rows: LoanAccount[]): number {
  return rows.reduce((sum, r) => sum + r.outstanding, 0);
}
```

### Type inference — learn what NOT to annotate
```ts
const count = rows.length;   // already number, don't write : number
```
Annotate function parameters and API boundaries. Let TypeScript infer the rest.

### `any` vs `unknown`
`any` turns type checking off. Treat it as a last resort with a `// TODO` next to it. `unknown` is the safe version — you must check the shape before using it.

### Narrowing
```ts
if (row.lastPaymentDate !== null) {
  // TypeScript now knows it's a string here
}
```

### Generics — the light version
You only need to *read* them at first:
```ts
Array<LoanAccount>          // same as LoanAccount[]
Promise<LoanAccount[]>      // what an async fetch returns
useState<Filters>({ ... })  // React state
```
Writing your own generics can wait months.

### Utility types worth knowing early
`Partial<T>`, `Pick<T, K>`, `Omit<T, K>`, `Record<K, V>`

```ts
type FilterDraft = Partial<Filters>;
type CountsByBucket = Record<Bucket, number>;
```

**Milestone:** convert your Phase 0 file to `.ts` with zero `any`.

---

## Phase 2 — TypeScript + React (~1–2 weeks)

### Setup
```bash
npm create vite@latest my-dashboard -- --template react-ts
```
Files are `.tsx` when they contain JSX, `.ts` when they don't.

### Typing props
```tsx
type KpiCardProps = {
  label: string;
  value: number;
  delta?: number;
  onClick?: () => void;
};

function KpiCard({ label, value, delta }: KpiCardProps) {
  return <div>{label}: {value}</div>;
}
```

### Typing state
```tsx
const [filters, setFilters] = useState<Filters>({ reportDate: "2026-08-31" });
const [rows, setRows] = useState<LoanAccount[]>([]);
const [status, setStatus] = useState<"idle" | "loading" | "error">("idle");
```
That last one — a union for loading state — will save you from four separate booleans.

### Children and composition
```tsx
type PanelProps = { title: string; children: React.ReactNode };
```

### Event handlers
```tsx
function onChange(e: React.ChangeEvent<HTMLInputElement>) {
  setFilters({ ...filters, branch: e.target.value });
}
```

### Hooks with types
- `useEffect` — no types needed, just understand the dependency array
- `useMemo<T>` — for expensive aggregations over rows
- `useRef<HTMLDivElement>(null)` — for chart containers

**Milestone:** a single KPI card + a filter dropdown, both fully typed, no `any`.

---

## Phase 3 — Data and API typing (dashboard-specific)

This is where most real bugs live.

1. **Type the API response, not just the component.** Write a `types/api.ts` file mirroring what your backend returns.
2. **Fetch typing is a lie by default.** `await res.json()` returns `any`. Cast it deliberately:
   ```ts
   const data = (await res.json()) as LoanAccount[];
   ```
   Later, upgrade to runtime validation with **Zod** — it checks the shape at runtime *and* generates the type. Worth learning once you're comfortable.
3. **Discriminated unions** for your three segments — this is the type-level version of your stock-vs-flow distinction:
   ```ts
   type Widget =
     | { kind: "mis"; period: string; frozenAt: string }
     | { kind: "insights"; dimension: string }
     | { kind: "actionable"; assignedTo: string; asOf: string };
   ```
   TypeScript will force you to handle each case, and it makes overlap between segments structurally impossible.
4. **Dates.** Decide now: strings (ISO) everywhere in your types, converted to `Date` only at display time. Mixing the two is a common mess.
5. **Nulls from the database.** If a column is nullable, type it `| null` and handle it. Don't paper over it.

### Libraries you'll touch
- **Charting** — Recharts or Chart.js; both ship types
- **Tables** — TanStack Table (heavily generic; read the docs, don't guess)
- **Data fetching** — TanStack Query, once manual `useEffect` fetching gets painful
- **Validation** — Zod

---

## Phase 4 — Tooling and config (as needed)

- `tsconfig.json` — you mainly care about `strict: true`. Leave it on. It's harder now and much easier in month three.
- Reading compiler errors — the first line is usually the real problem; the rest is detail
- ESLint + Prettier
- VS Code: hover over anything to see its inferred type. Use this constantly — it's the fastest way to learn.
- `npm run build` catches type errors that the dev server may not surface

---

## Deliberately skip for now

These come up in tutorials and will slow you down with no near-term payoff:

- Decorators
- `namespace`
- Abstract classes and heavy OOP (React is function-first)
- Conditional types, mapped types, template literal types
- Declaration merging
- Writing your own `.d.ts` files
- `enum` — prefer union types of string literals

---

## Suggested sequence

| Week | Focus | Output |
|---|---|---|
| 1 | JS fundamentals | Aggregation script in plain JS |
| 2 | TS core | Same script in TS, strict, no `any` |
| 3 | React + TS | Typed KPI card and filter control |
| 4 | Data layer | One real API call, typed end to end |
| 5+ | Build one page | Ship the simplest Actionable page — it's row-level and concrete |

Start with an Actionable page rather than an Insights page. It's a list of rows with a filter, which is the smallest complete slice of the app, and the stakeholder demand is already there.

---

## Resources

- **typescript-lang.org → "TS for JavaScript Programmers"** — short, official, read it twice
- **Total TypeScript (free beginner tutorials)** — exercise-driven, matches how you'll actually use it
- **react.dev** — the current React docs; ignore anything teaching class components
- **Matt Pocock on YouTube** — short clips for specific confusions

Avoid older courses that predate hooks. They'll teach you patterns you then have to unlearn.
