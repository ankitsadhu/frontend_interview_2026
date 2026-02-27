# React Machine Coding — MANG Interview Question Bank

> 40 machine coding problems organized by difficulty with estimated time, key concepts tested, and follow-up extensions

---

## Tier 1: Fundamentals (30–45 min)

> Test: State management, event handling, conditional rendering, basic hooks

| # | Problem | Key Concepts | Time |
|---|---|---|---|
| 1 | **Counter App** — increment, decrement, reset, min/max limits | `useState`, event handlers, conditional disable | 15 min |
| 2 | **Toggle Switch** — animated on/off with label | `useState`, CSS transitions, controlled component | 15 min |
| 3 | **Accordion / FAQ** — single or multi-expand | `useState`, conditional rendering, array state | 20 min |
| 4 | **Star Rating** — click to rate, hover preview, read-only mode | `useState`, `onMouseEnter/Leave`, props | 25 min |
| 5 | **Tabs Component** — switchable tab content, active indicator | `useState`, composition, `children` mapping | 25 min |
| 6 | **Modal / Dialog** — open/close, overlay click to dismiss, trap focus | Portal, `useEffect` for keydown, focus trap | 30 min |
| 7 | **Toast / Notification System** — auto-dismiss, stack multiple, types (success/error) | `useState` array, `setTimeout`, `useCallback` | 30 min |
| 8 | **Tooltip** — hover to show, positioning (top/bottom/left/right), delay | `useState`, `useRef` for positioning, `onMouse*` | 30 min |
| 9 | **Progress Bar** — animated fill, percentage label | `useState`, `useEffect`, CSS transitions / `requestAnimationFrame` | 20 min |
| 10 | **Traffic Light** — auto-cycling red → yellow → green | `useState`, `useEffect` with `setInterval`, cleanup | 25 min |

---

## Tier 2: Intermediate (45–60 min)

> Test: Forms, API calls, debouncing, useEffect patterns, composition

| # | Problem | Key Concepts | Time |
|---|---|---|---|
| 11 | **Searchable Dropdown / Autocomplete** — filter options, keyboard navigation, highlight match | `useState`, `useRef`, debounce, `onKeyDown` | 45 min |
| 12 | **Todo App** — CRUD, filter (all/active/completed), local storage persist | `useState`/`useReducer`, `useEffect`, `localStorage` | 45 min |
| 13 | **Infinite Scroll List** — fetch on scroll, loading indicator, "no more items" | `useEffect`, `IntersectionObserver`, pagination | 45 min |
| 14 | **Image Carousel / Slider** — auto-play, manual prev/next, dots indicator, swipe | `useState`, `useEffect` interval, touch events | 45 min |
| 15 | **Multi-Step Form / Wizard** — step navigation, validation per step, summary review | `useState` for step, form state, conditional render | 45 min |
| 16 | **Debounced Search with API** — type → debounce → fetch → display results | `useState`, `useEffect`, debounce, `AbortController` | 30 min |
| 17 | **Pagination Component** — page numbers, prev/next, ellipsis for large ranges | `useState`, derived state for page range, `useMemo` | 35 min |
| 18 | **Countdown Timer** — start, pause, resume, reset, format MM:SS | `useState`, `useRef` for interval ID, `useEffect` cleanup | 30 min |
| 19 | **Drag & Drop Sortable List** — reorder items via drag, visual drop indicator | `onDragStart/Over/Drop`, state reorder, `dataTransfer` | 45 min |
| 20 | **OTP Input** — 4-6 auto-focusing boxes, paste support, backspace navigation | `useRef` array, `onKeyDown`, `onPaste`, focus management | 40 min |

---

## Tier 3: Advanced (60–90 min)

> Test: Complex state, performance, custom hooks, architecture decisions

| # | Problem | Key Concepts | Time |
|---|---|---|---|
| 21 | **Data Table** — sort by column, filter, paginate, select rows, bulk actions | `useReducer`, `useMemo` for sorted/filtered data, performance | 75 min |
| 22 | **File Explorer / Tree View** — recursive folder/file rendering, expand/collapse, breadcrumb | Recursion, `useState` for expanded nodes, tree data structure | 60 min |
| 23 | **Kanban Board** — drag cards between columns, add/edit/delete cards | Complex state, drag & drop, multi-list state management | 75 min |
| 24 | **Typeahead / Mention System** — `@mention` in textarea, dropdown, insert selected | `contentEditable` or cursor position tracking, `useRef` | 60 min |
| 25 | **Chat Interface** — message list, input, scroll-to-bottom, typing indicator, timestamps | `useRef` for scroll, `useEffect`, message grouping | 60 min |
| 26 | **Calendar / Date Picker** — month grid, navigate months, select date/range, today highlight | Date math, grid layout, `useState` for selected/viewed month | 75 min |
| 27 | **Multi-Select Tag Input** — type to add tags, remove via ✕ or backspace, suggestions dropdown | `useState` array, `onKeyDown`, controlled input, composition | 45 min |
| 28 | **Undo/Redo** — any stateful app with undo/redo stack, keyboard shortcuts | `useReducer` with history stack, `Ctrl+Z`/`Ctrl+Y` | 45 min |
| 29 | **Virtualized List** — render 10,000 items efficiently with smooth scrolling | `useRef` for scroll, calculated visible window, absolute positioning | 60 min |
| 30 | **Form Builder** — dynamic form from JSON config, validation, submit | Recursive rendering, schema-driven UI, dynamic validation | 75 min |

---

## Tier 4: Expert / System Design (90+ min)

> Test: Architecture, performance at scale, real-world complexity

| # | Problem | Key Concepts | Time |
|---|---|---|---|
| 31 | **Spreadsheet / Excel Clone** — editable cells, formula support (`=A1+B1`), cell references | Cell state matrix, formula parser, dependency graph, memoization | 90 min |
| 32 | **Poll Widget** — create poll, vote, real-time results, bar chart visualization | State management, percentage calc, animation, optional WebSocket | 60 min |
| 33 | **Nested Comments / Reddit-style Thread** — reply, collapse, vote, infinite nesting | Recursive component, tree CRUD, optimistic updates | 60 min |
| 34 | **E-Commerce Cart** — add/remove items, quantity, discount codes, order summary, checkout | `useReducer` for cart, derived totals, side effects | 75 min |
| 35 | **Real-time Collaborative Editor** — multi-cursor, conflict resolution, sync indicator | CRDT/OT concepts, WebSocket, optimistic updates | 90 min |
| 36 | **Dashboard with Widgets** — draggable/resizable widget grid, save layout, responsive | Grid layout, drag/resize logic, layout persistence | 90 min |
| 37 | **Command Palette / Spotlight Search** — `Cmd+K`, fuzzy search, keyboard nav, grouped results | Portal, fuzzy matching, keyboard navigation, `useRef` | 60 min |
| 38 | **Image Cropper** — drag to select crop area, resize handles, preview, export | Canvas API, mouse event math, `useRef` for canvas | 75 min |
| 39 | **Transfer List** — two lists, move items between, select all, search/filter | Dual state, bulk operations, controlled checkboxes | 45 min |
| 40 | **Stepper / Workflow Visualizer** — step status (completed/active/pending), branching paths | State machine, SVG/CSS connectors, dynamic rendering | 60 min |

---

## How to Approach Machine Coding Rounds

### Template: 5-Step Structure

```
1. CLARIFY (2 min)
   - Ask about edge cases
   - Confirm: keyboard accessibility? Mobile support? API format?

2. PLAN (3–5 min)
   - Component tree (what components do you need?)
   - State design (what state, where does it live?)
   - Data flow (props down, events up)

3. BUILD CORE (60% of time)
   - Get the basic functionality working FIRST
   - Hardcode data initially, connect API later
   - Focus on happy path

4. HANDLE EDGE CASES (20% of time)
   - Loading/error states
   - Empty states
   - Keyboard accessibility (Tab, Enter, Escape)
   - Boundary values

5. POLISH (10% of time)
   - Clean code, meaningful names
   - Extract custom hooks if logic is reusable
   - Performance (memo/useCallback only if clearly needed)
```

### What Interviewers Evaluate

| Criteria | What They Look For |
|---|---|
| **Component design** | Composable, single-responsibility, reusable |
| **State management** | Right state location, minimal state, derived data |
| **Hook usage** | Correct dependency arrays, proper cleanup |
| **Edge cases** | Loading, error, empty, boundary conditions |
| **Code quality** | Naming, readability, no unnecessary complexity |
| **Communication** | Thinking out loud, tradeoff discussions |
| **Extensibility** | Can the design handle new requirements easily? |

---

## Common Follow-Up Extensions

Interviewers often ask "how would you extend this?" after you finish:

| Base Problem | Extension |
|---|---|
| Todo App | Add drag-to-reorder, categories, due dates, sync with backend |
| Search/Autocomplete | Highlight matching text, recent searches, keyboard nav |
| Modal | Stacked modals, animation enter/exit, confirm on close |
| Data Table | Column resize, virtual scroll for 100k rows, inline editing |
| Infinite Scroll | Pull-to-refresh, scroll position restore on back nav |
| Chat | Unread count, reactions, image upload, message search |
| Kanban | Column limits (WIP), swimlanes, card templates |
| Form | Conditional fields, file upload, progress % |
| Calendar | Multi-day events, drag to reschedule, recurring events |
| Carousel | Lazy load images, preload next, touch gesture support |

---

## Priority Picks for Interview Prep

If you have limited time, focus on these **10 high-frequency problems**:

1. ⭐ **Todo App** (#12) — tests end-to-end state management
2. ⭐ **Autocomplete / Searchable Dropdown** (#11) — debounce, keyboard nav, API
3. ⭐ **Infinite Scroll** (#13) — IntersectionObserver, pagination
4. ⭐ **Multi-Step Form** (#15) — form state, validation, step management
5. ⭐ **Data Table** (#21) — sort, filter, paginate, complex state
6. ⭐ **File Explorer / Tree View** (#22) — recursion, tree operations
7. ⭐ **Countdown Timer** (#18) — useRef for interval, useEffect cleanup
8. ⭐ **Star Rating** (#4) — controlled component, hover state
9. ⭐ **OTP Input** (#20) — focus management, paste handling
10. ⭐ **Undo/Redo** (#28) — useReducer with history, keyboard shortcuts

---

*Last updated: February 2026*
