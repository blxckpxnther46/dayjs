# Issue #2743 Reproduction: Year Misparsing in Russian Localization with CustomParseFormat

## 1. Overview
- **Issue**: [#2743 - The year is not parsed correctly in Russian localization with CustomParseFormat](https://github.com/iamkun/dayjs/issues/2743)
- **Branch**: `issue-reproduction`
- **Current Day.js Version in Repository**:
  - `package.json` version: `0.0.0-development` (standard Day.js repository development version for semantic-release)
  - Latest version in `CHANGELOG.md`: **1.11.22** (released 2026-08-16)
  - Git Commit: `436bde0bcded312781cbe45dc2b0ef079a36d8e3` (`chore: update doc`)
- **Node Version**: `v24.11.1`
- **Status**: **CONFIRMED & REPRODUCED** on the latest repository code without any rollbacks.

---

## 2. Bug Description
When parsing a date string containing the Russian genitive month of December (`декабря`) with format `DD MMMM YYYY`, the resulting year is parsed as **+1** to the actual year.

### Code under test:
```javascript
import dayjs from "dayjs";
import customParseFormat from "dayjs/plugin/customParseFormat";
import "dayjs/locale/ru";

dayjs.extend(customParseFormat);

console.log(dayjs("01 декабря 2024", "DD MMMM YYYY", "ru").year());
```

- **Expected**: `2024`
- **Actual**: `2025`

---

## 3. Reproduction Proof of Work

### 3.1 Direct Execution & Output

```bash
node -e "
const dayjs = require('./dayjs.min.js');
const customParseFormat = require('./plugin/customParseFormat');
require('./locale/ru');

dayjs.extend(customParseFormat);

const d = dayjs('01 декабря 2024', 'DD MMMM YYYY', 'ru');
console.log('Parsed year:', d.year());
console.log('Formatted date:', d.format('YYYY-MM-DD'));
"
```

**Terminal Output:**
```
Parsed year: 2025
Formatted date: 2025-12-01
```

The year was parsed as **2025** instead of **2024**, moving the entire date forward by exactly 1 year (12 months).

---

### 3.2 Comparison of Cases

Testing standalone nominative form (`декабрь`), genitive format form (`декабря`), short format (`дек.`), and previous months (`ноября`):

```bash
node -e "
const dayjs = require('./dayjs.min.js');
const customParseFormat = require('./plugin/customParseFormat');
require('./locale/ru');

dayjs.extend(customParseFormat);

console.log('декабря (MMMM - format form):', dayjs('01 декабря 2024', 'DD MMMM YYYY', 'ru').year());
console.log('декабрь (MMMM - standalone form):', dayjs('01 декабрь 2024', 'DD MMMM YYYY', 'ru').year());
console.log('дек.    (MMM  - short form):', dayjs('01 дек. 2024', 'DD MMM YYYY', 'ru').year());
console.log('ноября  (MMMM - November):', dayjs('01 ноября 2024', 'DD MMMM YYYY', 'ru').year());
"
```

**Terminal Output:**
```
декабря (MMMM - format form): 2025   <-- BUG
декабрь (MMMM - standalone form): 2024 <-- Works
дек.    (MMM  - short form): 2024    <-- Works (matches in 0..11)
ноября  (MMMM - November): 2024      <-- Works
```

---

### 3.3 Scope Across Other Locales

Any locale that specifies both `monthStandalone` and `monthFormat` (`months.s = monthStandalone`) has this identical bug for the month of December in format form:

| Locale | Month Input | Tested Format | Result Year | Expected Year | Status |
|---|---|---|---|---|---|
| Russian (`ru`) | `01 декабря 2024` | `DD MMMM YYYY` | `2025` | `2024` | **BUG** |
| Ukrainian (`uk`) | `01 грудня 2024` | `DD MMMM YYYY` | `2025` | `2024` | **BUG** |
| Polish (`pl`) | `01 grudnia 2024` | `DD MMMM YYYY` | `2025` | `2024` | **BUG** |
| Belarusian (`be`) | `01 снежня 2024` | `DD MMMM YYYY` | `2025` | `2024` | **BUG** |

---

## 4. Root Cause Analysis

### Source Location
File: [`src/plugin/customParseFormat/index.js`](file:///d:/dayjs/src/plugin/customParseFormat/index.js)

Lines 114–121:
```javascript
MMMM: [matchWord, function (input) {
  const months = getLocalePart('months')
  const matchIndex = months.indexOf(input) + 1
  if (matchIndex < 1) {
    throw new Error()
  }
  this.month = (matchIndex % 12) || matchIndex
}],
```

Lines 40–45:
```javascript
const getLocalePart = (name) => {
  const part = locale[name]
  return part && (
    part.indexOf ? part : part.s.concat(part.f)
  )
}
```

### Detailed Trace
1. When a locale provides both `monthStandalone` (`s`) and `monthFormat` (`f`), `getLocalePart('months')` concatenates them:
   - Index `0..11`: 12 standalone month names (`part.s`), e.g., index `11` = `"декабрь"`.
   - Index `12..23`: 12 format month names (`part.f`), e.g., index `23` = `"декабря"`.
   The combined array `months` has 24 elements.

2. When parsing `"декабря"`:
   - `months.indexOf("декабря")` evaluates to `23`.
   - `matchIndex = 23 + 1 = 24`.
   - Evaluating: `this.month = (matchIndex % 12) || matchIndex`:
     - `24 % 12` is `0` (which is falsy in JavaScript).
     - Because `0` is falsy, `0 || matchIndex` falls back to `matchIndex` (`24`).
     - Therefore, `this.month` is set to **`24`**!
   *(Note: for `"декабрь"`, `matchIndex = 12`, so `12 % 12` is `0`, which evaluated to `0 || 12 = 12`. It only worked by coincidence because `matchIndex` was 12!)*

3. In `parseFormattedInput` (lines 191–194):
   ```javascript
   let M = 0
   if (!(year && !month)) {
     M = month > 0 ? month - 1 : now.getMonth()
   }
   ```
   - When `month = 24`, `M = 24 - 1 = 23`.
   - When JavaScript builds the date `new Date(2024, 23, 1)`:
     - Month index `0` = January 2024
     - Month index `11` = December 2024
     - Month index `12` = January 2025
     - Month index `23` = December 2025
   - The date overflows by 12 months, resulting in year **2025** instead of **2024**.

4. The exact same vulnerability exists for `MMM` at line 112 if `monthsShort` contains different standalone and format entries:
   ```javascript
   this.month = (matchIndex % 12) || matchIndex
   ```

---

## 5. Potential Fix (For Reference)

Changing the fallback from `|| matchIndex` to `|| 12` (or using `((matchIndex - 1) % 12) + 1`):

```javascript
this.month = (matchIndex % 12) || 12
```

When `matchIndex = 24`:
`24 % 12` is `0`, so `0 || 12` results in `12`.
Month 12 -> `M = 12 - 1 = 11` (December), correctly preserving the year as 2024 without overflow.
