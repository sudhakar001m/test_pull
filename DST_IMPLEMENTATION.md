# DST implementation — annotated walkthrough, ioctl contract and delivery schedule

Companion to `DST_IOCTL_GUIDE.md`. This document explains **every line added** to
`tod/linesync_new/files/tod_gen2.c` for the DST port, and gives a phase-by-phase
implementation/testing schedule with hour estimates and a graceful (buffer-aware)
timeline.

Line numbers refer to the file as committed (5683 lines). Function order:

| Region | Line range | What it is |
|---|---|---|
| State fields | 719–760 | `dst_enabled` bit and three DST values |
| DST block | 2192–2765 | rules, search, work, tick check |
| LineSync hook | ~2922–2934 | second boundary in `do_time_zerocross()` |
| SQW hook | ~3298–3310 | second boundary in `linesync_scm_sqrhandler()` |
| ioctl commands | 4536–4550 | six command numbers |
| ioctl handlers | 4687–4807 | six `case` blocks |
| probe | 5286–5294 | work init + safe defaults |
| remove | 5604 | work flush |

---

## 1. State added to `struct linesync_driver`

```c
	uint32_t     determining_bus_ratio:2, mode:2,
		     rtc_writeback:1, sigen:1,
		     dst_enabled:1;
	...
	int32_t      atc_dst_offset;
	uint32_t     next_dst_secs;
	int32_t      next_dst_offset;
```

* `dst_enabled:1` — one spare bit in the bitfield word the driver already has.
  Set by `ATC_TOD_DST_ENABLE`, cleared by `DISABLE` and on probe. No extra
  storage and no alignment change to the struct.
* `atc_dst_offset` — the DST offset (seconds, normally 0 or 3600) that is in
  effect for the current TOD second. This is the value user space reads.
* `next_dst_secs` — epoch second of the next transition. Pre-computed in
  process context so the interrupt handler only has to compare one integer.
  `0xFFFFFFFF` is the "no event" sentinel (no real epoch second reaches it).
* `next_dst_offset` — the offset to copy into `atc_dst_offset` when that second
  arrives.

Design note: these are plain scalars on purpose. The IRQ path reads and writes
them without a lock; each is naturally aligned and singly written, so a torn
read cannot happen on the target. The table itself is never touched from an ISR.

---

## 2. Constants (2227–2233)

* `#define ATC_DST_MAX_ENTRIES 100` — bound for the table, the item array and
  every index check. Matches what the old driver exposed, so existing user-space
  tables stay valid.
* `#define DST_TYPE_ABSOLUTE 0` — historical encoding: `type == 0` means the
  rule stores a raw epoch second; any nonzero value means a calendar rule. This
  asymmetry is preserved because the ATC application already sends it.
* `DST_DOM_SPECIFIC 0` — `gendom.dom` is the literal day of month.
* `DST_DOM_ON_OR_AFTER 1` — "Nth `<dow>` on or after `<on_after_dom>`".
* `DST_DOM_ON_OR_BEFORE 2` — "Nth `<dow>` on or before `<on_before_dom>`";
  occurrence counts backwards (1 = last).

---

## 3. ABI structures (2235–2277)

These bytes travel through `copy_from_user`/`copy_to_user`, so the layout is the
contract with the application.

### 3.1 `struct dst_absolute_struct`

* `int secs_from_epoch_to_transition` — the exact second the offset changes.
  Absolute rules are used for pre-computed or one-off transitions.
* `int dst_offset` — offset to apply from that second.

### 3.2 `struct dst_gen1_struct` (on-or-after)

* `char dow` — 1 = Sunday … 7 = Saturday (driver-wide convention).
* `char occur` — 1 = first match, 2 = second, …; 0 disables the walk.
* `char on_after_dom` — anchor day of month the forward walk starts from.

### 3.3 `struct dst_gen2_struct` (on-or-before)

* `char dow` — same weekday encoding.
* `char occur` — 1 = last match, 2 = second to last, …
* `char on_before_dom` — anchor day of month the backward walk starts from.

### 3.4 `union dst_gen_dom_union`

* `char dom` — the fixed-day form.
* `week_and_day` — the forward form.
* `reverse_occurances_of_day` — the backward form.
  The union exists so one rule costs 3 bytes instead of 9; the name
  `reverse_occurances_of_day` keeps the historical spelling so the struct
  definition matches the old header byte for byte.

### 3.5 `struct dst_generic_struct`

* `char month` — 1–12.
* `char dom_type` — one of the three `DST_DOM_*` selectors.
* `union dst_gen_dom_union gendom` — the day selector payload.
* `int dst_offset` — offset from this transition onward.
* `int seconds_from_midnight_to_transition` — local time-of-day of the change
  (7200 = 02:00).

### 3.6 `union dst_types_u`

* `absolute` — raw epoch form.
* `generic` — calendar form.
  Same 16-byte footprint for both, so `begin`/`end` have a fixed size whatever
  the type.

### 3.7 `dst_info_t`, `dst_info_item_t`, `dst_infos_t`, `dst_adjustment_t`

* `dst_info_t.enabled` — 0 = entry ignored, 1 = entry active.
* `dst_info_t.type` — `DST_TYPE_ABSOLUTE` (0) or calendar (nonzero).
* `dst_info_t.begin, end` — the two rules of this entry.
* `dst_info_item_t.index` — destination slot, 0–99.
* `dst_info_item_t.info` — user pointer to one `dst_info_t`.
* `dst_infos_t.numinfos` — how many items follow.
* `dst_infos_t.dst_info_items` — user pointer to the item array.
* `dst_adjustment_t.seconds_from_epoch_to_transition` — search result time.
* `dst_adjustment_t.dst_offset` — offset that result selects.

Verified sizes (host self-test): `dst_info_t` = 36 bytes, `dst_types_u` = 16,
`dst_generic_struct` = 16, `dst_absolute_struct` = 8. The self-test asserts
these so a future layout change fails immediately.

---

## 4. Table, lock and work item (2279–2290)

* `static dst_info_t dst_info_array[ATC_DST_MAX_ENTRIES];` — the whole table,
  zero-initialised (all entries disabled) at load.
* `static DEFINE_MUTEX(dst_lock);` — serialises table edits against the
  workqueue search. Documented as never taken in interrupt context, which is why
  the tick path cannot sleep.
* `static struct work_struct dst_btm_wq;` — one deferred job: "look ahead and
  arm the next transition".

---

## 5. `get_absolute_time_for_year()` (2319–2417)

Purpose: turn one rule plus one year into an epoch second. Pure arithmetic,
no clock, no locking.

* Declarations `struct atc_rtc_date_struct rtcdate;`,
  `struct atc_rtc_time_struct rtctime;` — working date/time values.
* `int dow, target_dow, occur;` — weekday of the anchor, weekday wanted by the
  rule, occurrence index wanted by the rule.
* `if (dgs->month < 1 || dgs->month > 12) return 0;` — a bad month has no date;
  0 can never be selected as a future event, so the rule is silently unusable.
* `memset(&rtcdate, 0, sizeof(rtcdate));` — clears hundredths/fractions/dow so
  the conversion below starts from a known state.
* `rtcdate.century = (year >= (start_year + 100));` — 1970-based driver: years
  from 2070 need the RTC century bit; earlier years do not.
* `if (year >= 2000) rtcdate.year = year - 2000; else rtcdate.year = year - 1900;`
  — the RTC year field is two digits; which base applies is decided by the
  century bit above.
* `rtcdate.month = dgs->month;` — only the month is known at this point; the
  day comes from the selector.
* `switch (dgs->dom_type)` — dispatch on the three supported rule shapes.
* `case DST_DOM_SPECIFIC:` — fixed day.
* `rtcdate.dom = dgs->gendom.dom;` — take the day literally.
* `conv_rtcdate_to_rtctime(&rtcdate, &rtctime);` — existing driver helper
  converts the date to seconds since 1970 (it already handles leap years).
* `break;` — no day walking needed.
* `case DST_DOM_ON_OR_AFTER:`
* `rtcdate.dom = dgs->gendom.week_and_day.on_after_dom;` — anchor day.
* `conv_rtcdate_to_rtctime(&rtcdate, &rtctime);` — seconds of the anchor day,
  00:00.
* `target_dow = dgs->gendom.week_and_day.dow;` — weekday we must land on.
* `occur = dgs->gendom.week_and_day.occur;` — which occurrence.
* `if (occur > 0 && target_dow >= 1 && target_dow <= 7) {` — ignore
  unconfigured/invalid combinations instead of computing nonsense.
* `dow = (((rtctime.secs / 86400) + 4) % 7) + 1;` — weekday of the anchor:
  days since 1970, plus 4 because 1 Jan 1970 was a Thursday, modulo 7, made
  1-based.
* `if (dow <= target_dow)` — target is later in the same week.
* `rtctime.secs += (target_dow - dow) * 86400;` — step forward that many days.
* `else` — target already passed this week.
* `rtctime.secs += (7 - (dow - target_dow)) * 86400;` — wrap to next week.
* `rtctime.secs += ((occur - 1) * 7) * 86400;` — then add whole weeks for the
  Nth match.
* `break;`
* `case DST_DOM_ON_OR_BEFORE:`
* `rtcdate.dom = dgs->gendom.reverse_occurances_of_day.on_before_dom;` —
  anchor day (e.g. 31 for "last Sunday of October").
* `conv_rtcdate_to_rtctime(&rtcdate, &rtctime);` — anchor at 00:00.
* `target_dow = dgs->gendom.reverse_occurances_of_day.dow;`
* `occur = dgs->gendom.reverse_occurances_of_day.occur;`
* `if (occur > 0 && target_dow >= 1 && target_dow <= 7) {` — same guard.
* `dow = ...` — same weekday computation as above.
* Historical comment block — records why the direction matters.
* `if (dow >= target_dow)` — target is earlier or equal in this week.
* `rtctime.secs -= (dow - target_dow) * 86400;` — walk **backwards** (this is
  the bug fix: the old code added here and produced a November date).
* `else` — target is later in the week, so wrap backwards.
* `rtctime.secs -= (7 - (target_dow - dow)) * 86400;`
* `rtctime.secs -= ((occur - 1) * 7) * 86400;` — Nth-from-last goes further
  back.
* `break;`
* `default: return 0;` — unknown selector: unusable rule, never a bogus date.
* `rtctime.secs += dgs->seconds_from_midnight_to_transition;` — move from
  midnight to the configured local time of day.
* `return rtctime.secs;` — the resolved epoch second (unsigned).

---

## 6. `get_absolute_from_generic()` (2420–2456)

Purpose: resolve a rule for the snapshot's year, rolling one year forward or
backward as needed.

* `unsigned int abs_time;` — resolved candidate.
* `int year;` — full year of the snapshot.
* `year = (snapshot_date->century ? 2000 : 1900) + snapshot_date->year;` —
  rebuild the four-digit year from the RTC fields.
* `if (year < start_year) year += 100;` — same 1900/2000 ambiguity rule the
  driver's own date conversion uses.
* `if (prev_or_next_flag) {` — forward search.
* `abs_time = get_absolute_time_for_year(...);` — this year's date.
* `if (abs_time < snapshot_time)` — already past.
* `abs_time = get_absolute_time_for_year(..., year + 1);` — take next year's.
* `} else {` — backward search.
* `abs_time = get_absolute_time_for_year(...);`
* `if (abs_time > snapshot_time)` — still in the future.
* `abs_time = get_absolute_time_for_year(..., year - 1);` — take last year's.
* `return abs_time;`

---

## 7. `dst_snapshot_secs()` (2459–2484)

Purpose: one consistent view of "now" for the search.

* `struct timespec64 ts;` — receives the TOD clock.
* `struct atc_rtc_time_struct rtctime;` — intermediate for the date conversion.
* `get_timeval(&ts);` — reads the same TOD value used as the system clock, so
  DST decisions match the time user space sees.
* `rtctime.secs = (unsigned int)ts.tv_sec;` — seconds since 1970.
* `rtctime.hundredths = 0;` / `rtctime.fractions = 0;` — sub-second fields are
  irrelevant for a whole-second decision; zeroing them avoids uninitialised
  data in the date conversion.
* `conv_rtctime_to_rtcdate(&rtctime, rtcdate);` — fills the caller's date.
* `return rtctime.secs;`

---

## 8. `find_dst_event()` (2487–2571)

Purpose: nearest transition (past or future) relative to the live TOD clock.

* Declarations: `dst_event` (cursor over entries), `event_dst_type` (cursor over
  begin/end), `event_index` (winning rule, -1 = none), `event_time` (best time
  so far), `event_dst_offset` (offset of the winner), `snapshot_time`,
  `absolute_time`, `rtcdate`, `i`, `j`.
* `event_time = prev_or_next_flag ? 0xFFFFFFFFU : 0U;` — sentinel: max for
  "earliest future", zero for "latest past".
* `snapshot_time = dst_snapshot_secs(&rtcdate);` — one clock read for the whole
  scan.
* `dst_event = dst_array;` — start at entry 0.
* `for (i = 0; i < num_entries; i++) {` — fixed bound, no user-controlled loop
  limit.
* `if (dst_event->enabled) {` — disabled entries are ignored entirely.
* `event_dst_type = &dst_event->begin;` — begin first.
* `for (j = 0; j < 2; j++) {` — then end; `j` also encodes which rule won.
* `if (dst_event->type == DST_TYPE_ABSOLUTE) {`
* `absolute_time = event_dst_type->absolute.secs_from_epoch_to_transition;` —
  raw second, no arithmetic.
* `} else {`
* `absolute_time = get_absolute_from_generic(event_dst_type, snapshot_time,
  &rtcdate, prev_or_next_flag);` — calendar rule resolved for the relevant year.
* Comment block — explains why the previous search is inclusive.
* `if ((prev_or_next_flag && absolute_time > snapshot_time &&
  absolute_time < event_time) ||` — future candidate: strictly after now and
  earlier than the current best.
* `(!prev_or_next_flag && absolute_time <= snapshot_time &&
  absolute_time > event_time)) {` — past candidate: at or before now
  (inclusive, so the transition is active at its own second) and later than the
  current best.
* `event_time = absolute_time;` — new best time.
* `event_index = (i * 2) + j;` — encode entry + begin/end.
* `if (dst_event->type == DST_TYPE_ABSOLUTE)` /
  `event_dst_offset = event_dst_type->absolute.dst_offset;` — offset from the
  winning absolute rule.
* `else event_dst_offset = event_dst_type->generic.dst_offset;` — or from the
  winning calendar rule.
* `}` — closes the candidate test.
* `event_dst_type++;` — next rule (begin → end).
* `}` — closes the rule loop.
* `}` — closes the enabled test.
* `dst_event++;` — next entry.
* Comment + `if (event_index < 0) {` — nothing matched.
* `adj->seconds_from_epoch_to_transition = 0;` / `adj->dst_offset = 0;` —
  clear outputs so callers cannot read stale values.
* `return -1;`
* `adj->seconds_from_epoch_to_transition = event_time;` /
  `adj->dst_offset = event_dst_offset;` — publish the winner.
* `return event_index;`

---

## 9. `find_dst_event_relative_to_time()` (2581–2659)

Purpose: same search, but the reference second is supplied by the caller (this
is the engine behind `ATC_TOD_DST_GETOFFSET`).

Line-by-line it is the same as section 8 except:

* Its kernel-doc adds the "reference second is supplied" rationale.
* `rtctime.secs = snapshot_time;` — reference time from the ioctl argument
  instead of the clock.
* `rtctime.fractions = 0; rtctime.hundredths = 0;` — the date conversion must
  not carry garbage.
* `conv_rtctime_to_rtcdate(&rtctime, &rtcdate);` — derive the date of that
  second so calendar rules resolve against the right year.
* No `dst_snapshot_secs()` call, everything else identical (sentinel, enabled
  check, begin/end loop, inclusive previous test, offset selection, clear on
  no-match).

---

## 10. `dst_recalc_next_locked()` (2662–2684)

Purpose: arm the one-compare trigger used by the IRQ path.

* `dst_adjustment_t dst_adj;` — search result.
* `int index;` — who won (unused beyond the success test, kept for clarity).
* `index = find_dst_event(dst_info_array, ATC_DST_MAX_ENTRIES, 1, &dst_adj);` —
  forward search over the whole table.
* `if (index >= 0) {` — an event exists.
* `linesync.next_dst_secs = dst_adj.seconds_from_epoch_to_transition;` — arm
  the second.
* `linesync.next_dst_offset = (int32_t)dst_adj.dst_offset;` — remember what to
  adopt.
* `pr_debug("TOD DST: next transition ...");` — compile-time-free trace; no
  cost unless dynamic debug enables it.
* `} else {` — no future event.
* `linesync.next_dst_secs = 0xFFFFFFFFU;` — park the trigger past any real
  epoch second.
* `linesync.next_dst_offset = linesync.atc_dst_offset;` — keep the current
  offset so a spurious trigger cannot change anything.
* `pr_debug("TOD DST: no upcoming transition\n");`

---

## 11. `recalc_dst()` (2687–2708)

Purpose: recompute "now" and "next" together; called from ioctl context only.

* `dst_adjustment_t dst_adj;`
* `mutex_lock(&dst_lock);` — table and shared state are protected here.
* `dst_recalc_next_locked();` — arms the next event (already lock-aware).
* Comment: the most recent transition defines the offset in effect.
* `if (find_dst_event(dst_info_array, ATC_DST_MAX_ENTRIES, 0, &dst_adj) >= 0)`
  — backward search.
* `linesync.atc_dst_offset = (int32_t)dst_adj.dst_offset;` — adopt it.
* `else linesync.atc_dst_offset = 0;` — no enabled rule in the past: no DST.
* `mutex_unlock(&dst_lock);`

---

## 12. `bottom_work_recalc_next_dst_event()` (2711–2725)

* Kernel-doc points out it runs after a transition was consumed.
* `(void)data;` — work callbacks receive a pointer we do not use; silences
  `-Wunused-parameter`.
* `mutex_lock(&dst_lock);` — same protection as `recalc_dst()`.
* `dst_recalc_next_locked();` — find the following transition.
* `mutex_unlock(&dst_lock);`

---

## 13. `dst_tick_second()` (2728–2763)

Purpose: the only code that runs per second for DST, in IRQ context.

* Kernel-doc: interrupt context, no table walk, no sleeping.
* `if (!linesync.dst_enabled) return;` — feature off: one branch per second.
* `if (secs < linesync.next_dst_secs) return;` — common case: not due yet.
  Because `next_dst_secs` is `0xFFFFFFFF` when nothing is armed, this also
  covers "no event".
* `linesync.atc_dst_offset = linesync.next_dst_offset;` — adopt the offset
  exactly at the transition second (same second the boundary moved into).
* Comment: one-shot until the bottom half looks ahead again.
* `linesync.next_dst_secs = 0xFFFFFFFFU;` — disarm; prevents a signal storm if
  the workqueue is delayed, and makes the next call fall through the cheap test.
* `sendTimingSignals_change();` — notify `ATC_TOD_REQUEST_ONCHANGE_SIG`
  listeners so the application can re-read the offset.
* `schedule_work(&dst_btm_wq);` — hand the calendar walk to process context.

---

## 14. LineSync second-boundary hook (~2922)

```c
		if (linesync.mode == 0) {
			if (ok_to_writeback && linesync.rtc_writeback)
				schedule_work(&linesync_btm_wq);

			/* Apply a pending DST transition at this second boundary. */
			dst_tick_second(linesync.atc_linesync_time.secs);
		}
```

* `if (linesync.mode == 0) {` — the check moved from the combined condition to
  wrap both actions; behaviour for RTC writeback is unchanged.
* `if (ok_to_writeback && linesync.rtc_writeback)` — unchanged existing guard:
  only write the RTC when power/phase conditions allow and a write is pending.
* `schedule_work(&linesync_btm_wq);` — unchanged existing RTC writeback.
* Comment — states why the call sits here.
* `dst_tick_second(linesync.atc_linesync_time.secs);` — one integer compare in
  the LineSync tick; the second value is the TOD second that just started.

---

## 15. SQW second-boundary hook (~3298)

* `if (linesync.rtc_writeback) schedule_work(&linesync_btm_wq);` — existing
  behaviour.
* Comment + `dst_tick_second(linesync.atc_sqwr_time.secs);` — the old driver
  also applied DST in square-wave mode; without this, a board running on
  `ATC_TIMESRC_RTCSQWR` would never see transitions.

---

## 16. Six ioctl command numbers (4536–4550)

* `#define ATC_TOD_DST_ENABLE _IO('T', 10)` — no argument.
* `#define ATC_TOD_DST_DISABLE _IO('T', 11)` — no argument.
* `#define ATC_TOD_DST_SETINFO _IOW('T', 12, struct dst_infos)` — kernel reads
  the container and, from it, the item array.
* `#define ATC_TOD_DST_GETINFO _IOW('T', 13, struct dst_infos)` — same
  container; direction naming follows this file's existing style.
* `#define ATC_TOD_DST_GETSTATE _IO('T', 14)` — returns the flag.
* `#define ATC_TOD_DST_GETOFFSET _IOW('T', 32, unsigned int)` — argument is a
  pointer to the reference second.
  Numbers 10–14 and 32 match the historical ATC interface; only the encoding
  changed.

---

## 17. ioctl handlers (4687–4807)

### ENABLE
* `linesync.dst_enabled = 1;` — turn processing on.
* `recalc_dst();` — compute current offset and next transition immediately
  (process context, lock taken inside).
* `sendTimingSignals_change();` — listeners re-read the offset now.
* `break;`

### DISABLE
* `linesync.dst_enabled = 0;` — stop processing.
* `linesync.atc_dst_offset = 0;` — report "no DST" again.
* `linesync.next_dst_secs = 0xFFFFFFFFU;` — disarm so a pending event cannot
  fire after disabling.
* `sendTimingSignals_change();`
* `break;`

### GETSTATE
* Comment + `return linesync.dst_enabled;` — 0/1 as the ioctl result.

### GETOFFSET
* Comment: argument is a pointer to the reference second.
* Locals: `dst_adj` (result), `seconds` (input), `found` (search index).
* `if (copy_from_user(&seconds, (void __user *)param, sizeof(seconds)))` —
  copy the 4-byte second.
* `return -EFAULT;` — bad pointer.
* Comment: table may be edited concurrently.
* `mutex_lock(&dst_lock);`
* `found = find_dst_event_relative_to_time(dst_info_array, ATC_DST_MAX_ENTRIES,
  0, &dst_adj, seconds);` — most recent transition at or before `seconds`.
* `mutex_unlock(&dst_lock);`
* `if (found < 0) return -ENOENT;` — no enabled rule before that second.
* `return (long)(int)dst_adj.dst_offset;` — offset in seconds (0 or 3600).

### SETINFO
* Locals: `dstinfos` (container), `items` (kernel copy), `i`.
* `copy_from_user(&dstinfos, ...)` + `return -EFAULT;` — read container.
* `if (dstinfos.numinfos > ATC_DST_MAX_ENTRIES) return -EINVAL;` — bound.
* `if (dstinfos.numinfos) {` — allow a no-op call with zero items.
* `items = kcalloc(dstinfos.numinfos, sizeof(*items), GFP_KERNEL);` — kernel
  copy of the item array; `kcalloc` removes any overflow and zeroes it.
* `if (!items) return -ENOMEM;`
* `if (copy_from_user(items, ...)) { kfree(items); return -EFAULT; }` — no leak
  on a bad pointer.
* `mutex_lock(&dst_lock);` — edits are serialised against the search.
* `for (i = 0; i < dstinfos.numinfos; i++) {`
* `if (items[i].index >= ATC_DST_MAX_ENTRIES) continue;` — ignore out-of-range
  slots (historical behaviour: partial updates are allowed).
* Comment + `if (copy_from_user(&dst_info_array[items[i].index], ...))` — copy
  into the live table slot-by-slot, so a fault mid-way leaves earlier entries
  applied (same as the old driver).
* `mutex_unlock`, `kfree`, `return -EFAULT;` — cleanup on fault.
* After the loop: `mutex_unlock(&dst_lock);` and `kfree(items);`.
* Comment + `if (linesync.dst_enabled) { recalc_dst(); sendTimingSignals_change(); }`
  — a live edit takes effect immediately, not at the next boundary.
* `break;`

### GETINFO
* Same container handling and bounds.
* `if (!dstinfos.numinfos) break;` — nothing requested, success.
* `kcalloc` + copy-in of items with the same leak-free error paths.
* `mutex_lock(&dst_lock);`
* Loop + range check.
* Comment + `copy_to_user(items[i].info, &dst_info_array[items[i].index],
  sizeof(dst_info_t))` — read back under the lock so a concurrent `SETINFO`
  cannot tear an entry.
* Cleanup, `mutex_unlock`, `kfree`, `break;`.

---

## 18. Probe and remove

Probe (~5286–5294):

* `INIT_WORK(&dst_btm_wq, bottom_work_recalc_next_dst_event);` — must run
  before any IRQ can queue the work; placed with the other work initialisers.
* Comment + `linesync.dst_enabled = 0;` — DST is user-enabled only.
* `linesync.atc_dst_offset = 0;` — no offset until a table says otherwise.
* `linesync.next_dst_secs = 0xFFFFFFFFU;` — nothing armed.
* `linesync.next_dst_offset = 0;` — keeps the disarm value coherent.
  These four make unbind/rebind deterministic: the module never inherits a stale
  transition from a previous bind.

Remove (~5604):

* `cancel_work_sync(&dst_btm_wq);` — waits for a running search and drops a
  pending one, so no work touches `dst_info_array` or `linesync` after the
  module text is freed. Placed with the other `cancel_work_sync()` calls, after
  the IRQ sources are shut down.

---

## 19. Why it is built this way (design answers)

1. **Why not compute the offset inside the ISR?** The search can touch 200
   rules across years and must take a mutex. That is impossible in interrupt
   context and would add unbounded latency to the TOD tick. One integer compare
   is all the ISR does.
2. **Why an offset instead of changing the clock?** That is the historical ABI:
   the application reads `dst_offset` and applies it. Moving the clock would
   double-apply time and would also force an RTC write at every transition.
3. **Why `0xFFFFFFFF`?** The TOD second is a 32-bit epoch counter; no real
   second reaches it, so it is a safe "disarmed" value that still falls through
   a single `<` test.
4. **Why inclusive `<=` for the previous search?** The tick path applies the new
   offset at the transition second. If the search were exclusive, `ENABLE` or
   `GETOFFSET` at exactly that second would report the old offset while the
   clock was already using the new one.
5. **Why keep 100 entries and `char` indices?** Compatibility with the existing
   application buffers; the index is range-checked before use.

---

## 20. Delivery schedule (graceful timeline)

Hands-on effort, no idle waiting counted except where stated. Buffer is 20 %.

| Phase | Work | Hours | Output / done when |
|---|---|---|---|
| P0 | Freeze baseline, `git init`, commit original driver | 0.5 | Commit `Import original tod_gen2.c`; patch base exists |
| P1 | State fields + ABI structures | 1.5 | Structs compile; self-test asserts sizes |
| P2 | Calendar resolution, all three rule shapes, year rollover | 3.0 | `get_absolute_time_for_year()` passes fixed-day / after / before cases |
| P3 | Next/previous search (both entry points, inclusive rule) | 1.5 | `find_dst_event*()` returns expected 2026/2027 epochs |
| P4 | Second-boundary hooks, LineSync + SQW | 1.0 | Tick test: not-due / due / disabled |
| P5 | Six ioctl handlers, validation, error paths | 2.5 | All six commands exercised, `-EINVAL`/`-EFAULT`/`-ENOENT` verified |
| P6 | Workqueue, locking, probe/remove lifecycle | 1.0 | Unload does not race; rebind resets state |
| P7 | Host self-test harness (extract + stubs) | 2.5 | `tod/dst_selftest/run.sh` green |
| P8 | Target build + functional test on the board | 4.0 | Module loads, ioctls work, one signal per transition, unload clean |
| P9 | Real transition soak | 2.0 (+24–72 h elapsed) | Offset and signal observed across a scheduled boundary |
| P10 | Review, walkthrough, handover | 1.5 | This document + patch reviewed |
| | **Total hands-on** | **20.5 h** | |
| | **With 20 % buffer** | **24.6 h** | |

### Calendar plan (graceful)

| Day | Phases | Hours | Milestone |
|---|---|---|---|
| Day 1 | P0, P1, P2 | 5.0 | Calendar core resolves all rule shapes |
| Day 2 | P3, P4, P5 | 5.0 | Feature complete in code; host tick tests pass |
| Day 3 | P6, P7 | 3.5 | Self-test green; lifecycle safe |
| Day 4 | P8 | 4.0 | Board-verified ioctl round trip |
| Day 5 | P9 start, P10 | 3.5 | Transition armed; review done |
| Day 6–8 | P9 soak window (elapsed only) | 0–2 | Transition observed, next event re-armed |
| | **Total** | **20.5–22.5 h over ~6 working days** | |

### If a phase overruns

* P2 overrun (most likely): fall back to absolute-epoch rules only, which
  unblocks P3–P5; calendar shapes land the next day. Saving: 2 h, cost: the
  application must send pre-computed transition seconds.
* P8 blocked by build/board access: P9 cannot start. Everything up to P7 is
  already verified; the remaining 6 h are hands-on board work, and the module
  is not declared accepted until P8/P9 pass.
* Soak window: if the real transition is days away, arm an absolute rule a few
  minutes ahead to prove the mechanism, then repeat once on the real date
  (elapsed time, not effort).

### Acceptance criteria (all must hold)

1. `SETINFO` then `GETINFO` returns identical bytes.
2. `GETSTATE` = 1 after enable, 0 after disable; offset 0 after disable.
3. `GETOFFSET(now)` matches the expected rule for winter and summer.
4. A transition armed minutes ahead produces exactly one on-change signal and
   exactly one offset change, at the configured second.
5. The following event is re-armed automatically; `dmesg` shows it via
   dynamic debug.
6. `rmmod` while enabled: no oops, no workqueue use-after-free.
7. Re-`insmod`: DST disabled, offset 0, no stale transition.

---

## 21. Test matrix

| Behaviour | Test | Where |
|---|---|---|
| US spring/fall transitions, 2026–2027 | `test_us_rules()` | host |
| EU last-Sunday rules, corrected backward walk | `test_eu_rules()`, `test_reverse_rule_boundaries()` | host |
| Absolute-epoch rules | `test_absolute_entry()` | host |
| `GETOFFSET` incl. empty table | `test_getoffset_path()` | host |
| Tick not-due / due / disabled, one-shot | `test_one_shot_tick()` | host |
| ABI sizes (`dst_info_t` 36 B) | size checks in `main()` | host |
| Module load, ioctl plumbing, signals | board steps in `DST_IOCTL_GUIDE.md` §8 | target |
| Real transition + re-arm | soak test | target |

Host side is reproducible with `sh tod/dst_selftest/run.sh`.
