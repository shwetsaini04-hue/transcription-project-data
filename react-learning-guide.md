# React — End to End

*A learning guide written for someone who knows C++, has just picked up JavaScript basics, and is building a loan journey analytics dashboard.*

---

## How to read this

Ten sections, meant to be worked through in order. Each one has the same shape:

- **The idea** — what the concept is and why it exists
- **The code** — a minimal example
- **In your dashboard** — how it applies to the loan journey project
- **Checkpoint** — something you should be able to build before moving on

Do not read this end to end and then start coding. Read one section, write the checkpoint code, then read the next. React is a thing you learn by typing, not by reading. The checkpoints matter more than the prose.

Estimated time: **3 to 4 weeks** at an hour or two a day. Sections 1–5 are the core; if you only get through those you can already build a working page.

**One note on your C++ background.** You will find React's syntax easy and its *model* strange. In C++ you tell the machine what to do step by step. In React you describe what the screen should look like for a given set of data, and React figures out the steps. That inversion is the whole learning curve. Everything else is detail.

---

## Table of contents

1. [Setup and mental model](#1-setup-and-mental-model)
2. [Components and JSX](#2-components-and-jsx)
3. [Props](#3-props)
4. [Rendering lists](#4-rendering-lists)
5. [State](#5-state)
6. [Events and forms](#6-events-and-forms)
7. [Lifting state up](#7-lifting-state-up)
8. [Effects and data fetching](#8-effects-and-data-fetching)
9. [Context — global filters](#9-context--global-filters)
10. [Routing — multiple pages](#10-routing--multiple-pages)
11. [Performance and custom hooks](#11-performance-and-custom-hooks)
12. [Common mistakes coming from C++](#12-common-mistakes-coming-from-c)
13. [What to skip](#13-what-to-skip)
14. [Putting it together — the dashboard build order](#14-putting-it-together--the-dashboard-build-order)

---

## 1. Setup and mental model

### The idea

React is a library for building user interfaces out of **components** — reusable, self-contained pieces of screen. A component is a JavaScript function that returns markup.

The core principle, and the one thing to internalise before anything else:

> **UI is a function of state.**

You never write "when the user clicks this, find that div and change its text." You write "the screen looks like *this* when the data is *this*." When the data changes, React re-runs your function and updates the screen for you.

In C++ terms: you are writing a pure function from data to a description of the screen. You are not writing a sequence of mutations.

### The code

Create the project:

```bash
npm create vite@latest loan-dashboard -- --template react
cd loan-dashboard
npm install
npm run dev
```

Open `http://localhost:5173`. You now have a running React app.

The files that matter:

```
src/
  main.jsx      ← entry point, mounts your app into the page. Rarely touched.
  App.jsx       ← your root component. Start here.
  components/   ← you will create this
  pages/        ← you will create this
```

`main.jsx` looks roughly like this and you can mostly ignore it:

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)
```

`StrictMode` deliberately runs some of your code twice in development to surface bugs. If you see a `console.log` appear twice, that is why. It does not happen in production.

### In your dashboard

Delete everything in `App.jsx` and replace it with:

```jsx
function App() {
  return <h1>Loan Journey Dashboard</h1>
}

export default App
```

Everything you build from here hangs off that component.

### Checkpoint

Get the Vite project running and change the heading text. Confirm the browser updates without you refreshing. That instant feedback loop is what you will be living in.

---

## 2. Components and JSX

### The idea

A component is a function that returns JSX. JSX is HTML-like syntax inside JavaScript. It is not HTML and it is not a string — it compiles down to function calls.

Two hard rules:

1. **Component names must start with a capital letter.** `JourneyCard` is a component. `journeyCard` is treated as a plain HTML tag and will silently fail.
2. **A component returns exactly one root element.** Wrap siblings in a `<div>` or in `<>...</>` (an empty "fragment" that renders nothing itself).

### The code

```jsx
function JourneyCard() {
  return (
    <div className="card">
      <h3>Pre-Approved</h3>
      <p>1,240 entries today</p>
    </div>
  )
}
```

Curly braces drop you back into JavaScript from inside JSX:

```jsx
function JourneyCard() {
  const journeyName = "Pre-Approved"
  const entries = 1240

  return (
    <div className="card">
      <h3>{journeyName}</h3>
      <p>{entries.toLocaleString('en-IN')} entries today</p>
      <p>Doubled: {entries * 2}</p>
    </div>
  )
}
```

Anything between `{ }` is evaluated as an expression. Note *expression*, not statement — you can put `entries * 2` or a ternary there, but not an `if` block or a `for` loop.

### JSX gotchas

These trip up everyone at the start:

| HTML | JSX | Why |
|---|---|---|
| `class="card"` | `className="card"` | `class` is a reserved word in JavaScript |
| `for="name"` | `htmlFor="name"` | same reason |
| `onclick="..."` | `onClick={handleClick}` | camelCase, and takes a function not a string |
| `<br>` | `<br />` | every tag must close |
| `style="color: red"` | `style={{ color: 'red' }}` | takes an object, hence the double braces |

The double braces in `style` confuse people. The outer pair means "JavaScript expression," the inner pair is an object literal. So it is `{` + `{ color: 'red' }` + `}`.

### In your dashboard

Make a `src/components/` folder and create `JourneyCard.jsx`:

```jsx
function JourneyCard() {
  return (
    <div>
      <h3>Pre-Approved</h3>
      <p>Entries today: 1,240</p>
      <p>Offers generated: 967</p>
      <p>Offer rate: 78%</p>
    </div>
  )
}

export default JourneyCard
```

Then use it in `App.jsx`:

```jsx
import JourneyCard from './components/JourneyCard'

function App() {
  return (
    <div>
      <h1>Loan Journey Dashboard</h1>
      <JourneyCard />
      <JourneyCard />
    </div>
  )
}
```

Two identical cards. Which is useless — every card shows the same numbers. That problem is what props solve.

### Checkpoint

Build a `StatBox` component that displays a hardcoded label and number. Render three of them inside `App`.

---

## 3. Props

### The idea

Props pass data from a parent component into a child. They are the component's parameters.

The critical rule: **props are read-only.** A component may never modify its own props. Think of them as `const` references — the child reads them, the parent owns them. Attempting to write to a prop is a bug, and in your case a C++ instinct worth actively unlearning.

### The code

```jsx
function JourneyCard(props) {
  return (
    <div>
      <h3>{props.name}</h3>
      <p>Entries: {props.entries}</p>
      <p>Offer rate: {props.offerRate}%</p>
    </div>
  )
}
```

Used as:

```jsx
<JourneyCard name="Pre-Approved" entries={1240} offerRate={78} />
```

Strings go in quotes. Everything else — numbers, booleans, arrays, objects, functions — goes in curly braces.

### Destructuring — how you will actually write it

Nobody writes `props.name` repeatedly. Destructure in the parameter list:

```jsx
function JourneyCard({ name, entries, offerRate }) {
  return (
    <div>
      <h3>{name}</h3>
      <p>Entries: {entries}</p>
      <p>Offer rate: {offerRate}%</p>
    </div>
  )
}
```

Same thing, less noise. This is the standard form and what you will see in every codebase.

Default values, for when a prop is not passed:

```jsx
function JourneyCard({ name, entries = 0, offerRate = 0 }) {
```

### The children prop

A special one. Whatever you put *between* a component's tags arrives as `children`:

```jsx
function Card({ title, children }) {
  return (
    <div className="card">
      <h3>{title}</h3>
      {children}
    </div>
  )
}
```

```jsx
<Card title="Offer Funnel">
  <p>Anything here becomes children.</p>
  <Chart />
</Card>
```

This is how you build layout wrappers — panels, modals, page shells. You will use it constantly.

### In your dashboard

```jsx
function App() {
  return (
    <div>
      <h1>Loan Journey Dashboard</h1>
      <JourneyCard name="Pre-Approved" entries={1240} offerRate={78} />
      <JourneyCard name="Top Up PA" entries={856} offerRate={71} />
      <JourneyCard name="PQ ReKYC" entries={432} offerRate={52} />
      <JourneyCard name="PQ NACH" entries={689} offerRate={44} />
      <JourneyCard name="PQ Standalone Asset" entries={301} offerRate={37} />
    </div>
  )
}
```

Working overview page. Still repetitive — five near-identical lines. Section 4 fixes that.

### Checkpoint

Rewrite your `StatBox` to take `label`, `value`, and `change` as props. Render four with different values.

---

## 4. Rendering lists

### The idea

You do not write a loop that appends elements. You take an array of data and `map` it into an array of elements. React renders arrays of elements natively.

This is the biggest departure from imperative UI code, and once it clicks, most of React clicks.

### The code

```jsx
const journeys = [
  { id: 'pa',        name: 'Pre-Approved',        entries: 1240, offerRate: 78 },
  { id: 'topup',     name: 'Top Up PA',           entries: 856,  offerRate: 71 },
  { id: 'rekyc',     name: 'PQ ReKYC',            entries: 432,  offerRate: 52 },
  { id: 'nach',      name: 'PQ NACH',             entries: 689,  offerRate: 44 },
  { id: 'standalone',name: 'PQ Standalone Asset', entries: 301,  offerRate: 37 },
]

function App() {
  return (
    <div>
      <h1>Loan Journey Dashboard</h1>
      {journeys.map(journey => (
        <JourneyCard
          key={journey.id}
          name={journey.name}
          entries={journey.entries}
          offerRate={journey.offerRate}
        />
      ))}
    </div>
  )
}
```

Five cards, one block of code. Add a sixth journey to the array and a sixth card appears with no other change. That is the payoff.

### The key prop

Every item in a mapped list needs a unique `key`. React uses it to track which item is which across re-renders. Without it you get a console warning and, eventually, real bugs where the wrong row updates.

**Use a stable id from your data.** Do not use the array index:

```jsx
// Wrong — breaks when the list is sorted or filtered
{journeys.map((j, index) => <JourneyCard key={index} ... />)}

// Right
{journeys.map(j => <JourneyCard key={j.id} ... />)}
```

The index is fine only if the list never reorders, never filters, and never has items inserted. On a dashboard with sortable tables, that is never true.

### Spreading props

When prop names match your object keys, this shorthand helps:

```jsx
{journeys.map(journey => (
  <JourneyCard key={journey.id} {...journey} />
))}
```

`{...journey}` spreads every key of the object into props. Convenient, but it hides which props the child actually receives. Use it for small, well-understood objects; write them out otherwise.

### In your dashboard

Move that `journeys` array into `src/data/journeys.js`:

```js
export const journeys = [
  { id: 'pa', name: 'Pre-Approved', entries: 1240, offerRate: 78 },
  // ...
]
```

```jsx
import { journeys } from './data/journeys'
```

Keeping mock data in its own file means that when you swap it for a real API call in section 8, only that one file changes.

### Checkpoint

Build a stage funnel component. Given an array of stage objects `{ stage, count }` for the PA journey — `offer_generated`, `offer_reviewed`, `offer_selection`, `offer_accepted`, `disb_initiated`, `disb_completed` — render one row per stage showing the name and count.

---

## 5. State

### The idea

Props come from outside and cannot change. **State is data a component owns and can change**, and changing it makes React re-render.

This is the beating heart of React. Everything interactive runs through it.

```jsx
import { useState } from 'react'

function Counter() {
  const [count, setCount] = useState(0)

  return (
    <button onClick={() => setCount(count + 1)}>
      Clicked {count} times
    </button>
  )
}
```

`useState(0)` returns a pair: the current value, and a function to change it. The array destructuring syntax `const [a, b] = ...` is just JavaScript.

### The mental model that matters

When you call `setCount`, three things happen:

1. React records the new value
2. React **calls your component function again** from the top
3. `useState` returns the *new* value this time, and the returned JSX reflects it

Your component function runs many times over the life of the app. Once per render. Sit with that for a minute — it explains almost every confusing React behaviour you will hit.

A consequence that catches everyone: **state updates are not immediate.**

```jsx
function handleClick() {
  setCount(count + 1)
  console.log(count)   // prints the OLD value
}
```

`count` is a const captured during this render. It will not change mid-function. The new value appears on the next render. This is not a bug and there is no way around it, nor do you need one.

If the new value depends on the previous one, pass a function:

```jsx
setCount(prev => prev + 1)
```

Necessary when you update more than once in the same handler:

```jsx
setCount(count + 1)
setCount(count + 1)      // both compute from the same old value → +1 total

setCount(p => p + 1)
setCount(p => p + 1)     // each gets the latest → +2 total
```

### Never mutate state

This is the rule that C++ habits fight hardest.

```jsx
// WRONG — React sees the same array reference and skips the re-render
const [rows, setRows] = useState([])
rows.push(newRow)
setRows(rows)

// RIGHT — new array
setRows([...rows, newRow])
```

React decides whether to re-render by comparing references, not contents. Mutating in place leaves the reference unchanged, so React concludes nothing happened.

Patterns you will need constantly:

```jsx
// add
setRows([...rows, newRow])

// remove
setRows(rows.filter(r => r.id !== targetId))

// update one item
setRows(rows.map(r => r.id === targetId ? { ...r, status: 'done' } : r))

// sort — sort() mutates, so copy first
setRows([...rows].sort((a, b) => b.count - a.count))

// update one field of an object
setFilters({ ...filters, journey: 'pa' })
```

That `sort` one is a genuine trap. `.sort()` and `.reverse()` modify the array in place, unlike `.map()` and `.filter()` which return new ones. Always spread before sorting.

### In your dashboard

A journey selector:

```jsx
import { useState } from 'react'
import { journeys } from './data/journeys'

function App() {
  const [selectedId, setSelectedId] = useState(null)

  const selected = journeys.find(j => j.id === selectedId)

  return (
    <div>
      <h1>Loan Journey Dashboard</h1>
      {journeys.map(journey => (
        <div key={journey.id} onClick={() => setSelectedId(journey.id)}>
          <h3>{journey.name}</h3>
          <p>{journey.entries} entries · {journey.offerRate}% offer rate</p>
        </div>
      ))}

      {selected && <p>Selected: {selected.name}</p>}
    </div>
  )
}
```

Note `selected` is **derived**, not stored in state. It is computed from `selectedId` on every render. Do not put it in its own `useState` — that gives you two sources of truth that can disagree.

> **Rule worth writing down:** if a value can be computed from existing state, compute it. Do not store it. Redundant state is the most common cause of "why is my UI showing stale data."

### Checkpoint

Add a toggle to your funnel that switches between showing absolute counts and showing conversion percentages. One piece of boolean state, one ternary in the JSX.

---

## 6. Events and forms

### The idea

Event handlers are functions passed as props. `onClick`, `onChange`, `onSubmit` — camelCase, and they take a function reference, not a string.

```jsx
<button onClick={handleClick}>Refresh</button>       // pass the function
<button onClick={handleClick()}>Refresh</button>     // WRONG — calls it during render
<button onClick={() => handleClick(id)}>Refresh</button>  // pass args via arrow
```

That middle line is a classic first-week bug. It calls the function while rendering, uses the return value as the handler, and if the function sets state you get an infinite loop.

### Controlled inputs

React's approach to form inputs: **state is the source of truth, the input just displays it.**

```jsx
function SearchBox() {
  const [query, setQuery] = useState('')

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      placeholder="Search applications"
    />
  )
}
```

The loop: user types → `onChange` fires → state updates → re-render → input shows the new value. It feels circular but it means the state always matches what is on screen.

If you set `value` without `onChange`, the input becomes read-only and React warns you.

### A filter bar

```jsx
function FilterBar() {
  const [filters, setFilters] = useState({
    journey: 'all',
    dateRange: '7d',
  })

  return (
    <div>
      <select
        value={filters.journey}
        onChange={e => setFilters({ ...filters, journey: e.target.value })}
      >
        <option value="all">All journeys</option>
        <option value="pa">Pre-Approved</option>
        <option value="topup">Top Up PA</option>
      </select>

      <select
        value={filters.dateRange}
        onChange={e => setFilters({ ...filters, dateRange: e.target.value })}
      >
        <option value="1d">Today</option>
        <option value="7d">Last 7 days</option>
        <option value="30d">Last 30 days</option>
      </select>
    </div>
  )
}
```

Note the `{ ...filters, journey: ... }` pattern — spread the existing object, override one key. Setting `setFilters({ journey: e.target.value })` alone would delete `dateRange`.

### In your dashboard

Filtering a list is just `filter` over derived data:

```jsx
const [search, setSearch] = useState('')

const visible = journeys.filter(j =>
  j.name.toLowerCase().includes(search.toLowerCase())
)

return (
  <>
    <input value={search} onChange={e => setSearch(e.target.value)} />
    {visible.map(j => <JourneyCard key={j.id} {...j} />)}
  </>
)
```

No effect, no manual DOM work. Type, state changes, list re-renders filtered.

### Checkpoint

Build a filter bar with a journey dropdown and a search box, and have it filter your journey cards.

---

## 7. Lifting state up

### The idea

When two components need the same data, move that state to their nearest common parent and pass it down. This is "lifting state up," and it is the main structural decision you make repeatedly in React.

The parent owns the state and passes down two things: the **value**, and a **function to change it**.

### The code

```jsx
function Dashboard() {
  const [selectedJourney, setSelectedJourney] = useState('pa')

  return (
    <div>
      <JourneyList
        journeys={journeys}
        selected={selectedJourney}
        onSelect={setSelectedJourney}
      />
      <FunnelChart journeyId={selectedJourney} />
    </div>
  )
}

function JourneyList({ journeys, selected, onSelect }) {
  return (
    <div>
      {journeys.map(j => (
        <div
          key={j.id}
          onClick={() => onSelect(j.id)}
          style={{ fontWeight: j.id === selected ? 'bold' : 'normal' }}
        >
          {j.name}
        </div>
      ))}
    </div>
  )
}
```

`JourneyList` does not own the selection. It receives which one is selected and a function to request a change. The parent decides what actually happens. The child stays reusable and dumb.

This convention — props named `onSomething` for callbacks, `handleSomething` for the functions themselves — is universal in React code.

### Where should state live?

Ask: which components need this?

- **One component** → keep it local
- **A component and its child** → parent
- **Two siblings** → their common parent
- **Half the app** → Context (section 9)

Do not default to putting everything at the top. State that lives higher than necessary causes unnecessary re-renders and makes components harder to move around.

### Checkpoint

Split your dashboard into `JourneyList` and `JourneyDetail`. Clicking a journey in the list shows its funnel in the detail panel. The selection state lives in the parent.

---

## 8. Effects and data fetching

### The idea

`useEffect` runs code that reaches *outside* React — API calls, timers, subscriptions, direct DOM access. React calls these "side effects."

```jsx
import { useState, useEffect } from 'react'

function JourneyStats() {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  useEffect(() => {
    async function load() {
      try {
        setLoading(true)
        const res = await fetch('/api/journeys')
        if (!res.ok) throw new Error(`HTTP ${res.status}`)
        const json = await res.json()
        setData(json)
      } catch (err) {
        setError(err.message)
      } finally {
        setLoading(false)
      }
    }
    load()
  }, [])

  if (loading) return <p>Loading…</p>
  if (error) return <p>Failed to load: {error}</p>
  return <JourneyList journeys={data} />
}
```

That loading / error / data triad is the shape of nearly every data-fetching component you will write. Build it as a habit — a dashboard that shows a blank screen while loading and nothing at all on failure is worse than useless, because people cannot tell "no data" from "broken."

### The dependency array

The second argument to `useEffect` controls when it runs:

```jsx
useEffect(() => { ... })              // after EVERY render — almost always wrong
useEffect(() => { ... }, [])          // once, on mount
useEffect(() => { ... }, [journeyId]) // on mount, and whenever journeyId changes
```

The third form is what you want when refetching on a filter change:

```jsx
useEffect(() => {
  fetch(`/api/journeys/${journeyId}/funnel`)
    .then(r => r.json())
    .then(setFunnel)
}, [journeyId])
```

Change the selected journey, the effect reruns, new data arrives.

**Omitting the array is the classic infinite loop.** The effect fetches, sets state, causes a re-render, which runs the effect, which fetches. If your network tab is scrolling forever, check for a missing dependency array.

### Cleanup

If an effect starts something ongoing, return a function that stops it:

```jsx
useEffect(() => {
  const id = setInterval(() => refreshData(), 30000)
  return () => clearInterval(id)
}, [])
```

Without cleanup, navigating away leaves the timer running forever. Same applies to event listeners and websocket connections.

For fetches, cleanup prevents a race — the user switches journeys quickly and the slower first response overwrites the newer one:

```jsx
useEffect(() => {
  let cancelled = false

  fetch(`/api/journeys/${journeyId}`)
    .then(r => r.json())
    .then(json => {
      if (!cancelled) setData(json)
    })

  return () => { cancelled = true }
}, [journeyId])
```

### When NOT to use useEffect

Overuse of `useEffect` is the single most common React mistake. It is not for transforming data.

```jsx
// WRONG — extra render, state that can go stale
const [total, setTotal] = useState(0)
useEffect(() => {
  setTotal(rows.reduce((sum, r) => sum + r.count, 0))
}, [rows])

// RIGHT — just calculate it
const total = rows.reduce((sum, r) => sum + r.count, 0)
```

Ask before writing an effect: *am I synchronising with something outside React?* If not — if you are just deriving a value from props or state — calculate it during render.

Effects are for: fetching, timers, subscriptions, logging, browser APIs. That is close to the whole list.

### In your dashboard

Replace your mock data import with a fetch. Keep the mock file around and point at it during development:

```jsx
useEffect(() => {
  fetch('/data/journeys.json')
    .then(r => r.json())
    .then(setJourneys)
    .catch(err => setError(err.message))
}, [])
```

Put `journeys.json` in the `public/` folder and Vite serves it as a real HTTP request. You get the actual async behaviour — loading states, error handling, the lot — without needing a backend yet.

### Checkpoint

Convert your journey list to load from a JSON file with proper loading and error states. Then add a refetch when the date filter changes.

---

## 9. Context — global filters

### The idea

Passing a prop down through five layers of components to reach the one that needs it — "prop drilling" — gets tedious fast. Context lets a value be read by any component beneath a provider, without threading it through every level.

Your dashboard needs this for global filters, which must persist as the user moves between MIS, Insights, and Actionable pages.

### The code

Three steps. Create the context:

```jsx
// src/context/FilterContext.jsx
import { createContext, useContext, useState } from 'react'

const FilterContext = createContext(null)

export function FilterProvider({ children }) {
  const [filters, setFilters] = useState({
    dateRange: '7d',
    journey: 'all',
    region: 'all',
  })

  function updateFilter(key, value) {
    setFilters(prev => ({ ...prev, [key]: value }))
  }

  return (
    <FilterContext.Provider value={{ filters, updateFilter }}>
      {children}
    </FilterContext.Provider>
  )
}

export function useFilters() {
  const ctx = useContext(FilterContext)
  if (!ctx) throw new Error('useFilters must be used inside FilterProvider')
  return ctx
}
```

Wrap your app:

```jsx
function App() {
  return (
    <FilterProvider>
      <Dashboard />
    </FilterProvider>
  )
}
```

Read it anywhere below:

```jsx
function DateSelector() {
  const { filters, updateFilter } = useFilters()

  return (
    <select
      value={filters.dateRange}
      onChange={e => updateFilter('dateRange', e.target.value)}
    >
      <option value="1d">Today</option>
      <option value="7d">Last 7 days</option>
      <option value="30d">Last 30 days</option>
    </select>
  )
}
```

No props passed. Any component at any depth can read and update the filters.

The `[key]: value` syntax in `updateFilter` is a *computed property name* — it uses the value of the `key` variable as the property name. Plain JavaScript, but easy to misread as an array.

### When to use it

Context is for genuinely global, rarely-changing values: filters, theme, current user, auth. Not for everything.

Every component reading a context re-renders when that context value changes. Putting fast-changing state in a context shared by the whole app makes everything re-render constantly.

**You do not need Redux.** For a dashboard of this size, Context plus `useState` covers it comfortably. Redux tutorials from a few years ago will tell you otherwise; they were written before hooks made this easy.

### Checkpoint

Move your filter state into a Context. Confirm a filter dropdown in the header changes data in a component that never receives it as a prop.

---

## 10. Routing — multiple pages

### The idea

React Router maps URLs to components. You have three segments and multiple pages under each, so this is essential rather than optional.

```bash
npm install react-router-dom
```

### The code

```jsx
// src/App.jsx
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom'

function App() {
  return (
    <BrowserRouter>
      <FilterProvider>
        <nav>
          <Link to="/">Overview</Link>
          <Link to="/mis/disbursal">Disbursal</Link>
          <Link to="/insights/funnel">Offer Funnel</Link>
          <Link to="/actionable/aging">Stage Aging</Link>
        </nav>

        <Routes>
          <Route path="/" element={<Overview />} />
          <Route path="/mis/disbursal" element={<DisbursalPage />} />
          <Route path="/insights/funnel" element={<OfferFunnel />} />
          <Route path="/insights/journey/:journeyId" element={<JourneyDeepDive />} />
          <Route path="/actionable/aging" element={<StageAging />} />
          <Route path="*" element={<NotFound />} />
        </Routes>
      </FilterProvider>
    </BrowserRouter>
  )
}
```

Use `<Link>`, never `<a href>`. An anchor tag triggers a full page reload and wipes all your state.

### URL parameters

The `:journeyId` in that route is a parameter. Read it with `useParams`:

```jsx
import { useParams } from 'react-router-dom'

function JourneyDeepDive() {
  const { journeyId } = useParams()

  useEffect(() => {
    fetch(`/api/journeys/${journeyId}/funnel`).then(...)
  }, [journeyId])

  return <h1>Deep dive: {journeyId}</h1>
}
```

Navigating to `/insights/journey/pa` gives `journeyId === 'pa'`. Your journey cards become links:

```jsx
<Link to={`/insights/journey/${journey.id}`}>
  <JourneyCard {...journey} />
</Link>
```

### Filters in the URL

Worth doing on a dashboard: put filters in the query string so people can share a link to a filtered view.

```jsx
import { useSearchParams } from 'react-router-dom'

function OfferFunnel() {
  const [searchParams, setSearchParams] = useSearchParams()
  const journey = searchParams.get('journey') ?? 'all'

  return (
    <select
      value={journey}
      onChange={e => setSearchParams({ journey: e.target.value })}
    >
      ...
    </select>
  )
}
```

Now `/insights/funnel?journey=nach` is a shareable link. On an ops dashboard where people paste links into Slack, this feature earns its keep quickly.

### Suggested structure

```
src/
  pages/
    mis/         Scorecard.jsx, Disbursal.jsx
    insights/    OfferFunnel.jsx, JourneyDeepDive.jsx, FailureReasons.jsx
    actionable/  StageAging.jsx, WorkQueues.jsx
  components/    JourneyCard.jsx, FunnelChart.jsx, FilterBar.jsx
  context/       FilterContext.jsx
  data/          mock JSON
```

Folders mirroring your three segments. Obvious where things go.

### Checkpoint

Set up routing with an overview page and one deep-dive page reached by clicking a journey card. Confirm the browser back button works.

---

## 11. Performance and custom hooks

### Custom hooks

A custom hook is a function starting with `use` that calls other hooks. It exists to share stateful logic between components.

You will write the same fetch-with-loading-and-error code in a dozen components. Extract it once:

```jsx
// src/hooks/useFetch.js
import { useState, useEffect } from 'react'

export function useFetch(url) {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  useEffect(() => {
    let cancelled = false
    setLoading(true)
    setError(null)

    fetch(url)
      .then(r => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`)
        return r.json()
      })
      .then(json => { if (!cancelled) setData(json) })
      .catch(err => { if (!cancelled) setError(err.message) })
      .finally(() => { if (!cancelled) setLoading(false) })

    return () => { cancelled = true }
  }, [url])

  return { data, loading, error }
}
```

Every page becomes three lines:

```jsx
function OfferFunnel() {
  const { data, loading, error } = useFetch('/api/funnel')

  if (loading) return <Spinner />
  if (error) return <ErrorState message={error} />
  return <FunnelChart data={data} />
}
```

This is the highest-leverage thing in this document for your project. Write it early.

The two rules for hooks, custom or built-in:

1. Only call them at the top level of a component or another hook — never inside `if`, loops, or nested functions
2. Only call them from React components or other hooks

React tracks hooks by call order. Conditional calls break that tracking, which is why the ESLint rule enforcing this is worth keeping on.

### useMemo

Caches an expensive calculation between renders.

```jsx
const sortedRows = useMemo(
  () => [...rows].sort((a, b) => b.count - a.count),
  [rows]
)
```

Only recomputes when `rows` changes. Useful for sorting or aggregating thousands of rows — an aging table, say. Not useful for arithmetic on five numbers, where the memo costs more than the calculation.

**Do not reach for this by default.** Write plain code, then optimise when something is measurably slow. Premature memoisation makes code harder to read for no benefit.

### useCallback

The same idea for functions. Relevant mainly with `React.memo`. Skip it for now; come back if you hit a real performance problem.

### Checkpoint

Extract your data fetching into a `useFetch` hook and use it on two different pages.

---

## 12. Common mistakes coming from C++

Things that will bite you specifically, given your background.

**Mutating state.** `array.push()`, `obj.field = x`, `array.sort()` — all silently fail to trigger a re-render. Always create a new object or array. This is the number one source of "my code is right but nothing updates."

**Expecting state to update immediately.** `setCount(5); console.log(count)` prints the old value. State applies on the next render. There is no way to force it and you do not need one.

**Calling a function in a handler instead of passing it.** `onClick={handleClick()}` runs during render. It should be `onClick={handleClick}`.

**Missing the dependency array.** `useEffect` with no second argument runs after every render. If it sets state, that is an infinite loop.

**Reaching for useEffect to compute things.** If a value derives from props or state, calculate it inline. Effects are for talking to the outside world.

**Storing derived state.** Two `useState` calls where one value can be computed from the other means two things that can disagree. Compute, do not store.

**`=== ` vs `==`.** Always use `===`. `==` does type coercion with results nobody wants — `0 == ''` is true, `null == undefined` is true.

**Undefined at runtime.** No compiler catches `data.journeys.map(...)` when `data` is still `null` during the first render. Guard with `data?.journeys` or an early return. This is the single biggest adjustment from C++, and the reason TypeScript is worth adding once React itself feels comfortable.

**Floating point in money.** `0.1 + 0.2 === 0.30000000000000004`. On a lending dashboard this shows up as ₹1,23,456.78000000001. Store paise as integers, or round explicitly at display time with `toFixed(2)`.

---

## 13. What to skip

Tutorials, especially older ones, spend time on things modern React does not use. Skip all of it:

| Skip | Why |
|---|---|
| Class components | Legacy. Hooks replaced them. You will only meet them in old codebases. |
| `this` binding, `bind(this)` | Only exists because of class components. |
| Lifecycle methods (`componentDidMount` etc.) | Replaced by `useEffect`. |
| Redux, Redux Toolkit | Overkill here. Context plus `useState` handles your dashboard. |
| Create React App | Deprecated. You are on Vite, which is correct. |
| PropTypes | Superseded by TypeScript. Skip both for now; add TS later. |
| Higher-order components, render props | Old patterns for sharing logic. Custom hooks replaced them. |
| `useRef` for most things | You will need it eventually — chart libraries, focus management — but not to learn React. |
| Server components, Next.js | Different architecture. Not needed for an internal dashboard. |

If a tutorial opens with `class App extends React.Component`, it predates 2019. Find another.

---

## 14. Putting it together — the dashboard build order

Concrete sequence for your project. Each step is buildable with what you know by that point.

**Week 1 — static shell**
Sections 1–4. Build the overview page with five journey cards from a hardcoded array. No interactivity. Get it looking right.

**Week 2 — interactivity**
Sections 5–7. Add the filter bar. Make cards clickable. Build a funnel component for a single journey. Everything still on mock data.

**Week 3 — data and structure**
Sections 8–10. Move mock data to JSON files in `public/`, load via `useFetch`. Add routing for the three segments. Move filters to Context.

**Week 4 — real pages**
Build the actual pages in this order:

1. **Overview** (MIS) — journey cards, the scorecard numbers
2. **Offer funnel** (Insights) — the four-stage cross-journey funnel
3. **Stage aging** (Actionable) — the stuck queue, the page ops will actually use

Three pages. Ship those, then watch which questions people still ask over Slack — those become pages four and five.

**After that**
Add TypeScript. You will feel the need before you finish week three, and coming from C++ the type system will be a relief rather than a burden. Add it once React itself is no longer the thing you are thinking about.

**Charting library**
Recharts is the usual pick for React dashboards — component-based, so it fits the mental model you have just learned. Add it in week four, not before. Learning React and a charting library simultaneously means you will not know which one is causing the error.

---

## Reference

**Docs** — [react.dev](https://react.dev). The rewritten docs are genuinely good and hook-based throughout. The "Learn" section is a full course. When something here is unclear, go there.

**Router** — [reactrouter.com](https://reactrouter.com)

**Vite** — [vite.dev](https://vite.dev)

**Rule of thumb for tutorials:** if it uses `create-react-app` or class components, it is out of date. Check the publish date before investing time.

---

## The five things that matter most

If you forget everything else:

1. **UI is a function of state.** Describe what the screen looks like for given data. Do not manipulate the DOM.
2. **Never mutate state.** Always create new objects and arrays.
3. **Derive, do not store.** If it can be calculated from existing state, calculate it.
4. **`useEffect` is for the outside world.** Fetching, timers, subscriptions. Not for computing values.
5. **Every list item needs a stable `key`.** Use an id from your data, not the array index.
