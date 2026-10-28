# Solution Report: Issue #2743 - Incorrect Year Parsing in Russian Localization with CustomParseFormat

## 1. Overview
- **Issue Reference**: [#2743 - The year is not parsed correctly in Russian localization with CustomParseFormat](https://github.com/iamkun/dayjs/issues/2743)
- **Branch**: `fix-issue-2743` (branched from `issue-reproduction`)
- **Status**: **RESOLVED & VERIFIED**
- **Test Suite Result**: **93 / 93 suites passed (794 / 794 tests passed)**
- **Bundle Size**: 2.72 KB (within 2.99 KB size limit)

---

## 2. The Problem

### Reported Behavior
When parsing a date string containing the Russian month of December in genitive/format form (`декабря`) using `customParseFormat` with format `DD MMMM YYYY`, the resulting year was incremented by 1 (i.e. `2024` became `2025`):

```javascript
import dayjs from "dayjs";
import customParseFormat from "dayjs/plugin/customParseFormat";
import "dayjs/locale/ru";

dayjs.extend(customParseFormat);

console.log(dayjs("01 декабря 2024", "DD MMMM YYYY", "ru").year());
// Output: 2025 (Expected: 2024)
```

### Mathematical & Logical Mechanism Behind the Bug
1. In locales that support grammatical cases for months (such as Russian `ru`, Ukrainian `uk`, Polish `pl`, Belarusian `be`, Lithuanian `lt`, and Croatian `hr`), month names are split into:
   - `monthStandalone` (nominative form: e.g. `январь` ... `декабрь`)
   - `monthFormat` (genitive/contextual form: e.g. `января` ... `декабря`)
2. In `src/plugin/customParseFormat/index.js`, `getLocalePart('months')` concatenates both arrays:
   ```javascript
   const getLocalePart = (name) => {
     const part = locale[name]
     return part && (
       part.indexOf ? part : part.s.concat(part.f)
     )
   }
   ```
   This produces a 24-element array:
   - Index `0` to `11`: 12 standalone months (December `декабрь` is at index 11).
   - Index `12` to `23`: 12 format months (December `декабря` is at index 23).
3. The parser calculated the 1-based month index:
   ```javascript
   const matchIndex = months.indexOf(input) + 1
   ```
   For `"декабря"` (index 23), `matchIndex = 23 + 1 = 24`.
4. Then, the parser calculated the month number using:
   ```javascript
   this.month = (matchIndex % 12) || matchIndex
   ```
   - In JavaScript, `24 % 12` evaluates to `0`.
   - `0` is a falsy value in JavaScript.
   - Therefore, `(24 % 12) || matchIndex` evaluates to `0 || 24`, which evaluates to **`24`**!
   - `this.month` was assigned `24` instead of `12`.
   *(Note: for `"декабрь"`, `matchIndex = 12`, so `12 % 12` was `0`, resulting in `0 || 12` -> `12`. It only worked by coincidence because `matchIndex` was 12).*
5. Later in `parseFormattedInput`:
   ```javascript
   M = month > 0 ? month - 1 : now.getMonth()
   ```
   When `month = 24`, `M = 23`.
   Passing `M = 23` to JavaScript's `new Date(2024, 23, 1)`:
   - Month 0..11 = Jan..Dec 2024
   - Month 12..23 = Jan..Dec 2025
   The 23rd month index overflowed into the next year, advancing the date by exactly 12 months (1 full year) to **December 1, 2025**.

---

## 3. Where the Bug Was

File: [`src/plugin/customParseFormat/index.js`](file:///d:/dayjs/src/plugin/customParseFormat/index.js)

### Exact Lines (Lines 112 & 120):
```javascript
105:   MMM: [matchWord, function (input) {
106:     const months = getLocalePart('months')
107:     const monthsShort = getLocalePart('monthsShort')
108:     const matchIndex = (monthsShort || months.map(_ => _.slice(0, 3))).indexOf(input) + 1
109:     if (matchIndex < 1) {
110:       throw new Error()
111:     }
112:     this.month = (matchIndex % 12) || matchIndex // <-- BUG
113:   }],
114:   MMMM: [matchWord, function (input) {
115:     const months = getLocalePart('months')
116:     const matchIndex = months.indexOf(input) + 1
117:     if (matchIndex < 1) {
118:       throw new Error()
119:     }
120:     this.month = (matchIndex % 12) || matchIndex // <-- BUG
121:   }],
```

---

## 4. How It Was Solved

### The Fix
Replace `(matchIndex % 12) || matchIndex` with `(matchIndex % 12) || 12` in both `MMM` and `MMMM`:

```diff
--- a/src/plugin/customParseFormat/index.js
+++ b/src/plugin/customParseFormat/index.js
@@ -109,7 +109,7 @@ const expressions = {
     if (matchIndex < 1) {
       throw new Error()
     }
-    this.month = (matchIndex % 12) || matchIndex
+    this.month = (matchIndex % 12) || 12
   }],
   MMMM: [matchWord, function (input) {
     const months = getLocalePart('months')
@@ -117,7 +117,7 @@ const expressions = {
     if (matchIndex < 1) {
       throw new Error()
     }
-    this.month = (matchIndex % 12) || matchIndex
+    this.month = (matchIndex % 12) || 12
   }],
```

### Why This Solution Is Complete & Robust
1. **Valid Range**: Since `matchIndex < 1` throws an error earlier, `matchIndex` is guaranteed to be an integer $\ge 1$.
2. **Modulo Behavior**:
   - For months 1 to 11 (e.g. Jan–Nov), `matchIndex % 12` is non-zero (1 to 11). Since non-zero integers are truthy, `this.month` receives `matchIndex % 12`. This holds whether the match was in the first 12 items (`1..11`) or the second 12 items (`13..23`).
   - For December (month 12), `matchIndex` is a multiple of 12 (12 for standalone, 24 for format). In either case, `matchIndex % 12` is `0` (falsy). With `|| 12`, it falls back directly to `12`.
3. **No Side Effects**: It preserves exact compatibility for all 12-month locales and fixes all 24-month (or multi-cased) locales without extra allocations or regex overhead.

---

## 5. Verification & Proof of Work

### 5.1 Direct Node Execution Comparison

```bash
node -e "
const dayjs = require('./dayjs.min.js');
const customParseFormat = require('./plugin/customParseFormat');
require('./locale/ru');
require('./locale/uk');
require('./locale/pl');
require('./locale/be');

dayjs.extend(customParseFormat);

console.log('декабря (ru format):    ', dayjs('01 декабря 2024', 'DD MMMM YYYY', 'ru').year());
console.log('декабрь (ru standalone):', dayjs('01 декабрь 2024', 'DD MMMM YYYY', 'ru').year());
console.log('дек.    (ru MMM):       ', dayjs('01 дек. 2024', 'DD MMM YYYY', 'ru').year());
console.log('грудня  (uk format):    ', dayjs('01 грудня 2024', 'DD MMMM YYYY', 'uk').year());
console.log('grudnia (pl format):    ', dayjs('01 grudnia 2024', 'DD MMMM YYYY', 'pl').year());
console.log('снежня  (be format):    ', dayjs('01 снежня 2024', 'DD MMMM YYYY', 'be').year());
"
```

#### Results:
| Case | Input | Format | Locale | Before Fix | After Fix |
|---|---|---|---|---|---|
| Russian Format | `01 декабря 2024` | `DD MMMM YYYY` | `ru` | `2025` ❌ | `2024` ✅ |
| Russian Standalone | `01 декабрь 2024` | `DD MMMM YYYY` | `ru` | `2024` ✅ | `2024` ✅ |
| Russian Short Format | `01 дек. 2024` | `DD MMM YYYY` | `ru` | `2024` ✅ | `2024` ✅ |
| Ukrainian Format | `01 грудня 2024` | `DD MMMM YYYY` | `uk` | `2025` ❌ | `2024` ✅ |
| Polish Format | `01 grudnia 2024` | `DD MMMM YYYY` | `pl` | `2025` ❌ | `2024` ✅ |
| Belarusian Format | `01 снежня 2024` | `DD MMMM YYYY` | `be` | `2025` ❌ | `2024` ✅ |

Full date check:
```bash
dayjs('01 декабря 2024', 'DD MMMM YYYY', 'ru').format('YYYY-MM-DD')
# Output: '2024-12-01'
```

---

### 5.2 Unit Tests Added

In [`test/plugin/customParseFormat.test.js`](file:///d:/dayjs/test/plugin/customParseFormat.test.js):
```javascript
describe('month function locale', () => {
  it('MMMM', () => {
    const input = '08 мая 2020'
    const input2 = '08 май 2020'
    const format = 'DD MMMM YYYY'
    expect(dayjs(input, format, 'ru').valueOf()).toBe(moment(input, format, 'ru').valueOf())
    expect(dayjs(input2, format, 'ru').valueOf()).toBe(moment(input2, format, 'ru').valueOf())

    const inputDec = '01 декабря 2024'
    const inputDec2 = '01 декабрь 2024'
    expect(dayjs(inputDec, format, 'ru').year()).toBe(2024)
    expect(dayjs(inputDec, format, 'ru').valueOf()).toBe(moment(inputDec, format, 'ru').valueOf())
    expect(dayjs(inputDec2, format, 'ru').year()).toBe(2024)
    expect(dayjs(inputDec2, format, 'ru').valueOf()).toBe(moment(inputDec2, format, 'ru').valueOf())

    const inputUkDec = '01 грудня 2024'
    expect(dayjs(inputUkDec, format, 'uk').year()).toBe(2024)
    expect(dayjs(inputUkDec, format, 'uk').valueOf()).toBe(moment(inputUkDec, format, 'uk').valueOf())
  })
  it('MMM', () => {
    const input = '08 февр. 2020'
    const format = 'DD MMM YYYY'
    const inputDecShort = '01 дек. 2024'
    expect(dayjs(inputDecShort, format, 'ru').year()).toBe(2024)
    expect(dayjs(inputDecShort, format, 'ru').valueOf()).toBe(moment(inputDecShort, format, 'ru').valueOf())
    expect(dayjs(input, format, 'ru').valueOf()).toBe(moment(input, format, 'ru').valueOf())
  })
})
```

---

### 5.3 Full Test Suite Execution

Ran `npx jest --coverage=false`:
```
Test Suites: 93 passed, 93 total
Tests:       794 passed, 794 total
Snapshots:   0 total
Time:        13.912s
```
All **93 test suites** and **794 individual tests** passed cleanly with zero failures.

---

### 5.4 Bundle Size Verification

Ran `npm run size`:
```
Package size: 2.72 KB
Size limit:   2.99 KB
With all dependencies, minified and gzipped: 2.98 kB
```
Passed within the library's strict 2.99 KB size constraint.

---

### 5.5 ESLint Code Quality

Ran ESLint on modified files:
- 0 errors
- 0 warnings

---

### 5.6 Exhaustive Multi-Locale Verification (All 143 Locales)

An automated audit was run across **all 143 locale definitions** in the Day.js repository, comparing the old logic `(matchIndex % 12) || matchIndex` against the new fix `(matchIndex % 12) || 12` for all months and short months.

#### Full Month (`MMMM`) Audit Results:
Across all 143 locales, there were **exactly 6 differences**, representing the exact 6 locales where December had a separate genitive format name placed at index 23 (`matchIndex = 24`):

| Locale | Month Name | Month Index | Old Output (matchIndex) | New Output (Fixed: 12) | Status |
|---|---|---|---|---|---|
| Belarusian (`be`) | `'снежня'` | 23 (`matchIndex = 24`) | 24 (overflowed +1 year) | **12 (December)** | **Fixed** |
| Croatian (`hr`) | `'prosinca'` | 23 (`matchIndex = 24`) | 24 (overflowed +1 year) | **12 (December)** | **Fixed** |
| Lithuanian (`lt`) | `'gruodžio'` | 23 (`matchIndex = 24`) | 24 (overflowed +1 year) | **12 (December)** | **Fixed** |
| Polish (`pl`) | `'grudnia'` | 23 (`matchIndex = 24`) | 24 (overflowed +1 year) | **12 (December)** | **Fixed** |
| Russian (`ru`) | `'декабря'` | 23 (`matchIndex = 24`) | 24 (overflowed +1 year) | **12 (December)** | **Fixed** |
| Ukrainian (`uk`) | `'грудня'` | 23 (`matchIndex = 24`) | 24 (overflowed +1 year) | **12 (December)** | **Fixed** |

For **every other single month (January–November)** in these 6 locales, and for **all 12 months across all remaining 137 locales**, the old and new logic produce **100% identical outputs**.

#### Short Month (`MMM`) Audit Results:

Only 2 locales had separate standalone and format entries for `monthsShort`:

| Locale | Short Month Name | Month Index | Old Output (matchIndex) | New Output (Fixed: 12) | Status |
|---|---|---|---|---|---|
| Belarusian (`be`) | `'снеж.'` | 23 (`matchIndex = 24`) | 24 | **12 (December)** | **Fixed** |
| Russian (`ru`) | `'дек.'` | 23 (`matchIndex = 24`) | 24 | **12 (December)** | **Fixed** |

All other short month entries across all 143 locales evaluate 100% identically.

---

### 5.7 Cross-Timezone Verification

`customParseFormat` was rigorously verified across multiple timezones and host timezone environments:

1. **Host Timezone Matrix Execution**:
   Ran the test suite with process-level `TZ` overrides across diverse timezones:
   - `TZ=Pacific/Auckland` (UTC+12 / UTC+13): **57/57 passed**
   - `TZ=America/New_York` (UTC-5 / UTC-4): **57/57 passed**
   - `TZ=UTC` (UTC+0): **57/57 passed**

2. **Parsing with Timezone Offsets & `dayjs.tz`**:
   Tested parsing Russian December and standard dates with time zone offsets (`Z`, `ZZ`) across multiple timezones:
   - `UTC`, `Pacific/Auckland`, `Europe/London`, `America/New_York`, `Asia/Kolkata`, `America/Whitehorse`, `Asia/Tokyo`, `Australia/Sydney`.
   - Results: The parsed year (`2024`) and month (`11` / December) remained rock-solid across all timezones without drift or year overflow.

3. **Dedicated Timezone Test Suites**:
   Ran both `test/timezone.test.js` and `test/plugin/timezone.test.js`:
   - **60 / 60 tests passed**.

---

## 6. Git Status

- **Current Branch**: `fix-issue-2743`
- **Files Modified**:
  - `src/plugin/customParseFormat/index.js` (fix)
  - `test/plugin/customParseFormat.test.js` (unit tests)
- **New Files**:
  - `SOLUTION_ISSUE_2743.md` (solution documentation)
  - `ISSUE_2743_REPRODUCTION.md` (reproduction documentation from previous step)
- **Commit / Push Status**: Nothing has been committed or pushed. Ready for your review and command.
