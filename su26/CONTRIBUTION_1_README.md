# Contribution 1: Extra table header cell incorrectly added

**Contribution Number:** 1  
**Tech Fellow:** Matthew Wyatt  
**Issue:** https://github.com/phpmyadmin/phpmyadmin/issues/17869  
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because it's about fixing a well-scoped, self-contained bug in phpMyAdmin, a mature and widely-used open source tool for administering MySQL and MariaDB databases over the web. The bug is concrete and verifiable: an extra `<th>` cell is incorrectly rendered in a table header, so I can reproduce it visually, trace it back to the rendering logic, and confirm a fix without needing deep prior knowledge of the entire codebase. That makes it an ideal first contribution.

It also lines up well with what I want to learn. Tracking down the bug means working through phpMyAdmin's PHP and Twig-based table-rendering code, and fixing it properly means adding regression test coverage so the extra cell can't reappear. This will allow me to demonstrate the skills (reading an established codebase, debugging UI/rendering output, and writing meaningful tests) I'm hoping to build through this contribution.

---

## Understanding the Issue

### Problem Description

When browsing the rows of a table in phpMyAdmin (Browse mode), the results table's header row ends up with one more cell than each of the data rows below it. An extra, empty utility cell is rendered at the right edge of the header only, which leaves the header misaligned with the body. It was reported against phpMyAdmin 5.2.0 and is also confirmed on the 6.0.x branch; tracing the code through git history shows the empty placeholder cell has been present at least since phpMyAdmin 3.5.0 and was originally introduced all the way back in March 2002, so this is a long-standing bug rather than a recent regression. There is a second related defect in the same code path (see what I did there?): the stray cell is emitted as a `<td>` inside `<thead>`, which is invalid table markup! Header rows should only contain `<th>` cells.

### Expected Behavior

The header row should have exactly the same number of columns as the body rows so that every column label lines up with its data. A right-side action/utility column should appear in the header only when a matching cell also appears in each body row, and every cell in the header should be a `<th>` element.

### Current Behavior

In Browse mode an extra empty cell (`<td class="d-print-none">…</td>`) is appended to the right end of the header row, but no corresponding cell exists in the body rows, so the header ends up one column wider. The original reporter saw the cell rendered as `<td class="d-print-none"><span></span></td>` and suspected the active theme was the cause, since it did not reproduce on the public demo server. On closer inspection, the extra cell is controlled by the `$cfg['RowActionLinks']` configuration setting (default `'left'`) rather than the theme: when row action links are placed on the left, and there are no edit/delete links to display, the code still emits an empty placeholder column on the right side of the header.

### Affected Components

- `libraries/classes/Display/Results.php` — `getColumnAtRightSide()` (≈ lines 1969–2013) builds the right-hand header column. Its `elseif` branch (≈ line 1993) emits the stray empty `<td class="d-print-none">` at line ~2006 when `RowActionLinks` is `left`/`both` and edit/delete links are disabled. A mirror-image branch in `getFieldVisibilityParams()` emits a similar empty `<td>` for the left side at line ~1333.
- `templates/display/results/table.twig` (lines 210–212) — assembles the `<thead>` row by concatenating `headers.button`, `headers.table_headers_for_columns`, and `headers.column_at_right_side` (the output of the method above).
- `libraries/config.default.php` (line ~2565) — `$cfg['RowActionLinks'] = 'left'` is the default value that triggers the buggy branch.
- `test/classes/Display/ResultsTest.php` (line ~1583) — currently asserts the buggy output (`'column_at_right_side' => "\n" . '<td class="d-print-none"></td>'`), so any fix will require updating this expectation.

---

## Reproduction Process

### Environment Setup

I set up phpMyAdmin locally on macOS, targeting the **QA_5_2** branch (phpMyAdmin 5.2.4-dev), since the issue was reported against 5.2.0. (I first stood up `master` / 6.0.0-dev, then switched branches once I decided to work against the reported version.)

**Toolchain (Homebrew):**
```sh
brew install php composer yarn      # PHP 8.5.7, Composer 2.10.1, Yarn 1.22.22
```

**PHP + JS dependencies:**
```sh
composer install
yarn install                        # builds the theme CSS/JS (Sass + Babel on 5.2)
```

**Database (MariaDB in Docker):**
```sh
docker run --name pma-mysql -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 -d mariadb:latest
```

**Config:**
```sh
cp config.sample.inc.php config.inc.php
openssl rand -hex 16                 # value used for blowfish_secret
```
In `config.inc.php` I set `$cfg['blowfish_secret']` to the generated value and changed `$cfg['Servers'][$i]['host']` from `localhost` to `127.0.0.1`, so PHP connects to the Docker container over TCP instead of a unix socket.

**Run** (on 5.2 the web root is the repo root):
```sh
php -S localhost:8000 -t .
```
Then logged in at http://localhost:8000 with `root` / `root`.

#### Challenges faced

- **`composer install` failed on QA_5_2** with a git "bare repository" error (`fatal: cannot use bare repository ... (safe.bareRepository is 'explicit')`) while cloning the `thecodingmachine/safe` dependency from its cached bare repo. I resolved it with a one-off, scoped git config override that does **not** modify my global git config:
  ```sh
  GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=safe.bareRepository GIT_CONFIG_VALUE_0=all composer install
  ```
- **PHP version mismatch:** my machine runs PHP 8.5, but 5.2 officially targets PHP ≤ 8.2, so I expected possible deprecation notices on some screens. Core functionality ran with no fatal errors, so I left it as-is.
- **Connection over socket vs TCP:** the default `localhost` host made PHP try a unix socket to a non-existent local MySQL; switching the host to `127.0.0.1` forced a TCP connection to the Docker container and fixed login.

### Steps to Reproduce

Using the local 5.2.4-dev setup above (stock configuration, so `$cfg['RowActionLinks']` is its default `'left'`):

1. Log in to phpMyAdmin at http://localhost:8000 as `root` / `root`.
2. Open any table in **Browse** mode. I confirmed it on two:
   - `information_schema.COLLATIONS` — a read-only system view with no unique key, so no Edit/Copy/Delete row actions are shown.
   - A hand-made `repro_17869.staff` table with a primary key, so Edit/Copy/Delete row actions *are* shown.
3. Inspect the rendered results table (`table.table_results`) and compare the `<thead>` row against any `<tbody>` row (browser devtools, or the JS check in the evidence below).

**Observed result:** the header row carries one extra cell on its right edge that has no counterpart in the body rows, so the header is wider than the data. The extra cell is rendered as a `<td>` *inside* `<thead>`, which is also invalid table markup.

- On **`information_schema.COLLATIONS`**: the header has **9** cells vs **8** in each body row — exactly one extra — and the stray cell is `<td class="d-print-none"><span></span></td>`, matching the original bug report verbatim.
- On **`repro_17869.staff`**: the stray cell is `<td class="d-print-none" colspan="4"><span></span></td>`. Because Edit *and* Delete are enabled, `emptyafter` is 4, so it spans 4 phantom columns (header spans 12 column-widths vs the body's 8).

### Reproduction Evidence

- **Commit showing reproduction:** This isn't a recent regression I can pin to a fork commit since the bug has existed since the feature was first written. The commit where it originally surfaced is [`16843c6`](https://github.com/phpmyadmin/phpmyadmin/commit/16843c684b5d5229343595c68e4331bbc8dc0c3e) (Loïc Chapeaux, 2002-03-03), which first introduced the empty placeholder `<td>` in the results header.
- **Screenshots/logs:** Reproduced locally on phpMyAdmin 5.2.4-dev (Playwright-driven). In each highlighted shot, the stray header cell is outlined in red, the header row is outlined in blue and the first body row in green, making the width mismatch visible.

  `information_schema.COLLATIONS` (matches the report exactly — one extra cell, no colspan):

  ![COLLATIONS browse view with the extra header cell highlighted](assets/repro-17869-collations-highlighted.png)

  `repro_17869.staff` (primary key present, so the stray cell spans the 4 action columns):

  ![staff browse view with the extra header cell highlighted](assets/repro-17869-staff-highlighted.png)

  Unannotated reference shot of the `staff` browse view: [assets/repro-17869-staff-clean.png](assets/repro-17869-staff-clean.png)

  The mismatch was confirmed in the DOM, not just by eye. Running this in the browser console on the `COLLATIONS` page:
  ```js
  const t = document.querySelector('table.table_results');
  t.querySelectorAll('thead tr')[0].children.length;  // => 9
  t.querySelector('tbody tr').children.length;         // => 8
  t.querySelectorAll('thead td')[0].outerHTML;         // => '<td class="d-print-none"><span></span></td>'
  ```
  Note on the `<span>`: the server-side PHP emits an *empty* `<td class="d-print-none"></td>` (no span). The inner `<span></span>` is added by phpMyAdmin's client-side JS, which wraps every header/data cell's content in a `<span>` for the column drag/resize feature — which is why the original report shows `<span></span>` even though it isn't in the rendered server markup.
- **My findings:** Walking the file's git history (a trail also pointed out in [this comment on the issue](https://github.com/phpmyadmin/phpmyadmin/issues/17869#issuecomment-1313569075)) shows the stray empty cell is over two decades old and survived several refactors intact:
  - [`16843c6`](https://github.com/phpmyadmin/phpmyadmin/commit/16843c6#diff-7b9a476426ac829a8aacf9901ba4cd3d0e6f75fe067defa00be7e5c258f9c6f7R662) — *Loïc Chapeaux, 2002-03-03.* Where the empty placeholder `<td>` was first introduced (part of feature request #503015, "No 'xxxtext' button on vertical mode"). This is the origin the issue comment refers to as "21 years ago."
  - [`37d50c1`](https://github.com/phpmyadmin/phpmyadmin/commit/37d50c1#diff-98f45ff6ce5760b40756ed7ce67dc0b880e8ba55f1298d7c79bd7bb8b49a3d60R671-R685) — *Alexander M. Turek, 2003-11-26 ("Huge set of optimizations").* The empty-cell logic is carried forward unchanged.
  - [`76409e2`](https://github.com/phpmyadmin/phpmyadmin/commit/76409e2#diff-401fa9ac455f80ecd82d75155b189c586eadf55f976b2050915d031eaf79a39dR2050-R2065) — *Chanaka Indrajith, 2012-07-04.* The 3.5.x-era refactor that broke `_getTableHeaders()` into smaller helper functions (the ancestors of today's `getColumnAtRightSide()`); the empty `<td>` survives the refactor, which is why the bug is present as far back as phpMyAdmin 3.5.0.

---

## Solution Approach

### Analysis

The extra cell is produced entirely server-side by `getColumnAtRightSide()` in `libraries/classes/Display/Results.php` — the method whose output becomes `headers.column_at_right_side` in `templates/display/results/table.twig`. It builds the right-hand action-column header in two branches, and the second (`elseif`) branch emits the stray cell. Its guard condition is malformed:

```php
} elseif (
    ($GLOBALS['cfg']['RowActionLinks'] === self::POSITION_LEFT)        // (1) wrong side
    || ($GLOBALS['cfg']['RowActionLinks'] === self::POSITION_BOTH)
    && (($displayParts['edit_lnk'] === self::NO_EDIT_OR_DELETE)        // (2) && binds before ||
    && ($displayParts['del_lnk'] === self::NO_EDIT_OR_DELETE))
    && (! isset($GLOBALS['is_header_sent']) || ! $GLOBALS['is_header_sent'])
) {
    $rightColumnHtml .= "\n" . '<td class="d-print-none"' . $colspan . '></td>';   // (3) <td> in <thead>
}
```

Because `&&` binds tighter than `||` in PHP, this parses as `A || (B && C && D)`, where `A` is `RowActionLinks === 'left'`. With the default `'left'` setting `A` is `true`, so the whole condition is `true` regardless of the edit/delete/header-sent checks, and the empty cell is emitted on every Browse view. But left-positioned row actions mean the body has no right-side action column, so the header ends up one (effective) column wider than each data row. This is exactly what the reproduction shows: on `information_schema.COLLATIONS` (all row actions disabled) the cell *still* appears, which is only possible if the edit/delete sub-checks are being bypassed.

So there are three compounding defects:

1. **Wrong side:** the branch keys off `POSITION_LEFT`, but this is the *right*-side column builder — a left-only layout should produce nothing here.
2. **Operator precedence:** the missing parentheses around the position test create the `A || (…)` short-circuit, so the guard is effectively "always true when `RowActionLinks === 'left'`."
3. **Invalid markup:** when the cell *is* emitted it's a `<td>` inside `<thead>`; header rows should contain only `<th>`.

The smoking gun is the sibling method right above it. The **left**-side builder, `getFieldVisibilityParams()` (lines ~1293–1343), is written correctly: it precomputes `$leftOrBoth = (RowActionLinks === POSITION_LEFT) || (RowActionLinks === POSITION_BOTH)` into a variable and then writes `$leftOrBoth && (…)`, which sidesteps the precedence trap entirely. `getColumnAtRightSide()` simply never got the same treatment — it inlines the `||`/`&&` test without grouping it.

### Proposed Solution

Refactor `getColumnAtRightSide()` to mirror its already-correct sibling `getFieldVisibilityParams()`:

- Introduce a `$rightOrBoth = (RowActionLinks === POSITION_RIGHT) || (RowActionLinks === POSITION_BOTH)` local and gate both branches on it. This fixes the wrong-side constant *and* the precedence bug in one move, and makes the right-side cell render only when row actions are actually on the right or both sides. A `'left'` (default) layout then returns an empty string — no extra header cell, and the header matches the body.
- Emit the placeholder as `<th class="…">` instead of `<td>` so `<thead>` contains only header cells. (The same `<td>`-in-`<thead>` pattern exists in the left builder at line ~1333, worth cleaning up for symmetry.)
- Update the now-stale unit-test expectation at `test/classes/Display/ResultsTest.php:1583`, which currently asserts the buggy `'<td class="d-print-none"></td>'` output.

I'll confirm the exact `edit_lnk`/`del_lnk` sub-condition for the empty-placeholder branch empirically rather than copy it blindly, since the right builder currently checks the *inverse* of what the left builder checks (`edit === NO && del === NO` vs. the left side's `edit != NO || del != NO`).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** In Browse mode the results `<thead>` gains one extra cell on the right that the body rows lack, because `getColumnAtRightSide()` emits a stray `<td class="d-print-none">` whenever `$cfg['RowActionLinks']` is `'left'` (the default). Verified on 5.2.4-dev against both a custom keyed table and the keyless `information_schema.COLLATIONS` view.

**Match:** The correctly-written sibling `getFieldVisibilityParams()` in the same file is the template — it guards its branches with a precomputed `$leftOrBoth` boolean and so avoids the precedence pitfall. I'll mirror that with a `$rightOrBoth` guard.

**Plan:**
1. In `getColumnAtRightSide()` (`libraries/classes/Display/Results.php`), add a `$rightOrBoth` local and rewrite both branch conditions to use it (fixing the wrong constant + precedence).
2. Change the emitted placeholder from `<td class="d-print-none">` to `<th class="…">` for valid `<thead>` markup.
3. Update `ResultsTest.php` (line ~1583) and add a regression assertion that the rendered header cell count equals the body cell count.
4. (Consistency) apply the same `<td>`→`<th>` cleanup to the left builder at line ~1333.

**Implement:** https://github.com/MatthewOscar/phpmyadmin/tree/QA_5_2

**Review:** Follow phpMyAdmin's `CONTRIBUTING` guidelines — run the project's static analysis and linters (`composer run phpcs` / `phpstan`, and the JS lints), keep the diff minimal and focused, add a changelog entry if required, and reference issue #17869 in the PR.

**Evaluate:** Re-run the Playwright/DOM check across the `RowActionLinks` matrix (`left` / `right` / `both` / `none`) × (table with primary key / view without one) and confirm in every case that the header cell count equals the body cell count and that no `<td>` remains inside `<thead>`; run the PHPUnit suite (`ResultsTest`) and confirm the updated/added assertions pass.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
