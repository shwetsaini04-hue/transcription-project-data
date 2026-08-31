# JavaScript Learning Plan — Foundations for a React Dashboard

The companion to the TypeScript plan. This is the expanded version of its Phase 0 — do this one first.

**How to practice:** open Chrome DevTools (F12) → Console tab, and type things. No project setup, no build step. Every example below can be pasted straight in. Once you're comfortable there, install Node and run `node script.js` from the terminal.

Learn modern JavaScript only. If a tutorial uses `var`, `function()` callbacks everywhere, or jQuery, close it.

---

## 1. Values and variables

```js
const rate = 8.5;          // can't be reassigned — your default
let total = 0;             // reassignable — use when it changes
// var — never
```

**The seven things a value can be:** string, number, boolean, `null`, `undefined`, object, symbol. Arrays and functions are both objects under the hood.

**`null` vs `undefined`** — `undefined` means "nobody set this yet." `null` means "explicitly set to nothing." Your database nulls will arrive as `null`; a missing object key gives `undefined`.

**Truthiness.** These are all falsy: `false`, `0`, `""`, `null`, `undefined`, `NaN`. Everything else is truthy — including `[]` and `{}`.

```js
if (rows.length) { }        // right
if (rows) { }               // wrong — an empty array is truthy
```

**Always use `===`, never `==`.** `"0" == 0` is `true`, which is not what you want.

---

## 2. Numbers, strings, dates

### Numbers
```js
0.1 + 0.2                   // 0.30000000000000004
```
Not a bug — it's how binary floating point works. **For a lending dashboard this matters.** Never compare money with `===` after arithmetic. Either work in paise/cents as integers, or round at display time:

```js
const display = amount.toFixed(2);              // "1234.50" — a string
new Intl.NumberFormat("en-IN").format(amount);  // "1,23,450"
```

`NaN` appears when a conversion fails. `NaN === NaN` is `false` — use `Number.isNaN(x)`.

### Strings
```js
const label = `${branch}: ${count} accounts`;   // template literal
name.trim().toLowerCase()
id.startsWith("LN")
csv.split(",")
parts.join(" | ")
```

### Dates
The built-in `Date` is awkward. Learn just enough:
```js
new Date("2026-08-31")          // ISO strings parse reliably
d.toISOString().slice(0, 10)    // back to "2026-08-31"
```
Two rules that prevent most date bugs: store and pass dates as ISO strings, and convert to `Date` only when formatting or doing math. When date handling gets real, add `date-fns`.

---

## 3. Arrays — the core of dashboard work

Almost everything you build is a transformation of an array of rows.

```js
const rows = [
  { id: "LN01", branch: "Andheri", amount: 50000, dpd: 12,  status: "overdue" },
  { id: "LN02", branch: "Andheri", amount: 12000, dpd: 0,   status: "current" },
  { id: "LN03", branch: "Bandra",  amount: 98000, dpd: 45,  status: "overdue" },
];
```

| Method | Returns | Use for |
|---|---|---|
| `map` | new array, same length | reshaping rows for a table or chart |
| `filter` | new array, fewer items | applying dashboard filters |
| `reduce` | one value | totals, grouping |
| `find` | one item or `undefined` | looking up a row by id |
| `some` / `every` | boolean | "any overdue?" / "all settled?" |
| `sort` | **mutates!** | ranking — copy first |
| `slice` | copy of a range | top 10 |
| `includes` | boolean | membership check |

```js
const overdue = rows.filter(r => r.status === "overdue");
const total   = rows.reduce((sum, r) => sum + r.amount, 0);
const names   = rows.map(r => r.id);
const worst   = [...rows].sort((a, b) => b.dpd - a.dpd).slice(0, 5);
```

### Two traps

**`sort` without a comparator sorts as text.**
```js
[10, 9, 100].sort()              // [10, 100, 9]  ← wrong
[10, 9, 100].sort((a, b) => a - b)  // [9, 10, 100]
```

**`sort` and `reverse` change the original array.** In React that causes bugs that look like the UI "not updating." Copy with `[...rows]` first.

### Grouping — write this one until it's automatic
```js
const byBranch = rows.reduce((acc, r) => {
  acc[r.branch] = (acc[r.branch] ?? 0) + r.amount;
  return acc;
}, {});
// { Andheri: 62000, Bandra: 98000 }
```

`Object.entries(byBranch)` turns that back into an array you can `map` over for a chart.

---

## 4. Objects

```js
const row = rows[0];

const { branch, amount } = row;                    // destructuring
const { dpd: daysPastDue } = row;                  // rename
const updated = { ...row, status: "closed" };      // copy + change
const { id, ...rest } = row;                       // remove a key
```

**Optional chaining and nullish coalescing** — you'll use these constantly against real API data:
```js
customer?.address?.city              // undefined instead of a crash
const branch = filters.branch ?? "All";   // only falls back on null/undefined
```
Note `??` differs from `||`: `0 || 10` gives `10`, but `0 ?? 10` gives `0`. For counts and amounts, `??` is what you want.

**Objects are compared by reference,** not contents. `{a:1} === {a:1}` is `false`. This is why React needs a *new* object to detect a change — mutating the old one changes nothing it can see.

---

## 5. Functions

```js
function total(rows) { return rows.length; }         // declaration
const total = (rows) => rows.length;                 // arrow — your default
const byDpd = (a, b) => b.dpd - a.dpd;
```

Concepts worth understanding, in order of payoff:

1. **Functions as values** — passing `r => r.amount` into `map` is the whole idea
2. **Default parameters** — `function fmt(n, digits = 2) {}`
3. **Scope** — a `const` inside `{}` doesn't exist outside it
4. **Closures** — a function remembers the variables around where it was defined. This is what makes React hooks work, and also the source of the "stale value" bug you'll hit eventually
5. **Pure functions** — same input, same output, nothing mutated. Aim for this in every calculation function you write

You can skip `this`, `call`/`apply`/`bind`, and prototypes entirely for now.

---

## 6. Asynchronous JavaScript

The concept: JavaScript doesn't wait. A fetch takes 400ms and your code keeps running.

```js
async function loadAccounts(branch) {
  try {
    const res = await fetch(`/api/accounts?branch=${branch}`);
    if (!res.ok) throw new Error(`Request failed: ${res.status}`);
    const data = await res.json();
    return data;
  } catch (err) {
    console.error(err);
    return [];
  }
}
```

Points that trip people up:
- `fetch` **does not throw on a 404 or 500.** You must check `res.ok` yourself.
- `res.json()` is itself async — hence the second `await`.
- `await` only works inside an `async` function.
- An `async` function always returns a Promise, even when you `return 5`.
- Parallel requests: `const [a, b] = await Promise.all([fetchA(), fetchB()]);`

Understand Promises (`.then` / `.catch`) well enough to read them, then write `async`/`await` in your own code.

---

## 7. Modules and npm

```js
// utils/format.js
export function formatINR(n) { /* ... */ }
export default function Dashboard() { /* ... */ }

// elsewhere
import Dashboard, { formatINR } from "./utils/format.js";
```

Named exports for utilities, default export for a component — that's the React convention.

Then: what `package.json` is, `npm install`, `npm run dev`, and why `node_modules` is never committed. That's enough npm for months.

---

## 8. The DOM — a light touch

React will handle almost all of this, but you should know what it's doing underneath:

- The DOM is a tree of objects representing the page
- `document.querySelector`, `.textContent`, `.classList`
- `addEventListener("click", handler)` and the event object
- Why direct DOM manipulation inside React is a mistake

An hour on this is plenty. Don't spend a week learning DOM APIs you'll never call.

---

## 9. Debugging

This is a skill, not a topic — and it's the difference between an hour and a day.

- `console.log` with a label: `console.log("rows after filter:", rows)`
- `console.table(rows)` — genuinely great for row data
- DevTools **Network** tab — see the actual request and response
- DevTools **Sources** → breakpoints, or drop `debugger;` in your code
- Read the *first* line of an error. `Cannot read properties of undefined (reading 'name')` means something upstream was `undefined` — go find where.

---

## 10. Common beginner traps, collected

| Trap | Fix |
|---|---|
| `==` surprises | always `===` |
| `sort()` on numbers | pass a comparator |
| `sort`/`reverse`/`push` mutating state | copy with `[...arr]` first |
| `0.1 + 0.2` on money | integers or `toFixed` at display |
| Comparing objects with `===` | compare a field, or the ids |
| `if (rows)` on an empty array | `if (rows.length)` |
| Forgetting `await` | you get a `Promise`, not the data |
| `res.json()` on a failed request | check `res.ok` first |
| `\|\|` swallowing valid `0` | use `??` |

---

## Skip for now

Classes and inheritance, `this`, prototypes, generators, `Proxy`, `Symbol`, IIFEs, callback-style Node APIs, and anything jQuery. Some become useful later; none block you.

---

## Milestones

Work in one plain `.js` file with hardcoded data before touching React.

1. Print every branch name from an array of 20 loan rows
2. Total outstanding, and count by status
3. Group amounts by branch into an object, then into a chart-shaped array
4. Top 5 accounts by DPD, without mutating the original array
5. Format all amounts as Indian currency strings
6. Fetch real data from any public API and log it, with error handling
7. Split your helpers into a `format.js` module and import them

Finish #7 and you're ready for Phase 1 of the TypeScript plan.

---

## Resources

- **javascript.info** — the best free JS course there is. Parts 1 and 2 only; skip the rest for now.
- **MDN Web Docs** — your reference for any single method. Search "MDN array reduce".
- **freeCodeCamp JavaScript Algorithms** — if you want structured exercises with feedback.

Skip anything published before 2020, and skip full-stack "build 20 projects" courses until the fundamentals here feel dull.
