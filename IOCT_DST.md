# TOD DST implementation — behaviour, ioctls, effort and test plan

Reference for the DST feature that existed in `linesync_old/timing.c` and was
missing from `linesync_new/files/tod_gen2.c`.

---

## 1. What was missing, what was added

The new driver had **no DST code at all** (`grep -i dst` returned one unrelated
log string). The old driver had a complete table-driven implementation plus
seven DST-related ioctls.

Implemented in `tod/linesync_new/files/tod_gen2.c` (port of the old design, not
a rewrite):

| Area | Old location | New location |
|---|---|---|
| State fields (`dst_enabled`, `atc_dst_offset`, `next_*`) | `linesync_driver` | `linesync_driver` (near `ost_time`) |
| ABI structures (`dst_info_t`, items, infos) | ~line 2100 | DST block, ~2200 |
| Calendar rule resolution | `get_absolute_time_for_year()` | same name |
| Next/previous event search | `find_dst_event()` / `..._relative_to_time()` | same names |
| Recompute helpers | `recalc_dst()`, `bottom_work_recalc_next_dst_event()` | same names |
| Per-second transition check | inline in both IRQ handlers | `dst_tick_second()` |
| LineSync tick hook | `do_time_zerocross()` | `do_time_zerocross()` |
| SQW tick hook | `linesync_rtc_sqrhandler()` | `linesync_scm_sqrhandler()` |
| Six ioctls | `timing_ioctl()` | `timing_ioctl()` |
| Workqueue lifecycle | partial | `INIT_WORK`, `cancel_work_sync` |

Design points preserved from the old driver:

* The calendar never runs in interrupt context. The IRQ path compares the TOD
  second against one pre-computed value and, on a hit, queues the bottom half.
* `dst_info_array` is a fixed 100-entry table owned by user space.
* `atc_dst_offset` is **advisory**: it reports the offset in effect, it does not
  move the TOD clock or rewrite the RTC. This matches the old ABI, where the
  application applied timezone + DST itself. See section 5.

Two defects in the old code were corrected while porting (both documented in
the source):

* `DST_DOM_ON_OR_BEFORE` walked **forward** like the "on or after" rule, so
  "last Sunday on or before Oct 31" resolved into November. It now walks
  backwards.
* "Previous transition" used a strict `<`, so at the exact transition second the
  old offset was still reported while the tick path had already applied the new
  one. The previous-event search is now inclusive (`<=`) and agrees with
  `dst_tick_second()`.

---

## 2. Data model and ABI

`struct dst_info` (36 bytes, 4-byte aligned, layout must not change):

```c
struct dst_info {
    char enabled;              /* 0 = entry ignored                      */
    char type;                 /* 0 = absolute, nonzero = calendar rule  */
    union dst_types_u begin, end;
};
```

`union dst_types_u` is 16 bytes; `begin` and `end` are two separate members of
it, so the entry is `2 + 2 pad + 16 + 16 = 36` bytes (verified by the self-test):

```c
/* type == 0 */
struct dst_absolute_struct {
    int secs_from_epoch_to_transition;
    int dst_offset;
};

/* type != 0 */
struct dst_generic_struct {
    char month;                                  /* 1..12                */
    char dom_type;                               /* 0, 1 or 2            */
    union {
        char dom;                                /* dom_type 0           */
        struct { char dow, occur, on_after_dom; }  week_and_day;      /* 1 */
        struct { char dow, occur, on_before_dom; } reverse_occurances_of_day; /* 2 */
    } gendom;
    int dst_offset;                              /* seconds, e.g. 3600   */
    int seconds_from_midnight_to_transition;     /* e.g. 7200 = 02:00    */
};
```

Conventions, identical to the old driver:

* `dow`: 1 = Sunday … 7 = Saturday.
* `dom_type`:
  * `0` fixed day of month (`gendom.dom`).
  * `1` Nth `dow` **on or after** `on_after_dom`.
  * `2` Nth `dow` **on or before** `on_before_dom`; `occur` counts backwards
    (1 = last).
* `seconds_from_midnight_to_transition` is added to the resolved day.
* `dst_offset` is the offset that becomes effective **at** the transition.

Container types:

```c
typedef struct dst_info_item { unsigned char index; dst_info_t __user *info; } dst_info_item_t;
typedef struct dst_infos     { unsigned int numinfos; dst_info_item_t __user *dst_info_items; } dst_infos_t;
```

---

## 3. The six ioctls

All are handled on `/dev/tod` (minor `LINESYNC_DRIVER_DEV`). Command numbers
match the historical ATC application (10..14, 32); only the encoding changed to
the `_IO`/`_IOW` style used by the rest of this driver.

| Command | Code | Direction | Argument | Returns |
|---|---|---|---|---|
| `ATC_TOD_DST_ENABLE` | `_IO('T',10)` | none | — | 0 |
| `ATC_TOD_DST_DISABLE` | `_IO('T',11)` | none | — | 0 |
| `ATC_TOD_DST_SETINFO` | `_IOW('T',12)` | user → kernel | `struct dst_infos` | 0 / `-EINVAL` / `-EFAULT` / `-ENOMEM` |
| `ATC_TOD_DST_GETINFO` | `_IOW('T',13)` | kernel → user | `struct dst_infos` | 0 / `-EINVAL` / `-EFAULT` / `-ENOMEM` |
| `ATC_TOD_DST_GETSTATE` | `_IO('T',14)` | none | — | 0 or 1 |
| `ATC_TOD_DST_GETOFFSET` | `_IOW('T',32)` | user → kernel | pointer to `unsigned int` second | offset in seconds / `-ENOENT` / `-EFAULT` |

### 3.1 `ATC_TOD_DST_ENABLE`
Enables table processing, immediately recomputes both the offset in effect and
the next transition (process context, so the calendar walk is allowed here), and
sends the on-change signal so listeners re-read their offset. Idempotent.

What it must achieve: after the call, `GETSTATE` returns 1 and `GETOFFSET(now)`
returns the offset of the most recent enabled rule.

### 3.2 `ATC_TOD_DST_DISABLE`
Clears `dst_enabled`, forces `atc_dst_offset` to 0, disarms the next event
(`0xFFFFFFFF`) so no transition can fire, and sends the on-change signal.
The table itself is kept, so a later `ENABLE` needs no re-upload.

### 3.3 `ATC_TOD_DST_SETINFO`
Installs/overwrites table entries. `numinfos` must be ≤ 100. The item array is
copied to kernel memory first (`kcalloc`), then each `items[i].info` is copied
into `dst_info_array[items[i].index]` while holding `dst_lock`. Items with
`index >= 100` are skipped, as in the old driver — partial updates are allowed.
If DST is currently enabled the offset and next event are recomputed and the
on-change signal is sent.

What it must achieve: a subsequently enabled table resolves the expected
transitions, and editing the table while enabled takes effect without a
disable/enable cycle.

### 3.4 `ATC_TOD_DST_GETINFO`
Read-back path. Same container, direction reversed: the driver fills each
`items[i].info` buffer with a copy of `dst_info_array[items[i].index]`.
`numinfos == 0` is a no-op success. Used to verify what the kernel actually
stored after `SETINFO`.

### 3.5 `ATC_TOD_DST_GETSTATE`
Returns 1 when enabled, 0 when disabled. No pointer, no copy.

### 3.6 `ATC_TOD_DST_GETOFFSET`
Answers "what DST offset applies at second S?" without needing S to be now:
the caller passes a pointer to `unsigned int` seconds-since-epoch, and the
driver runs the same previous-event search against that value. Returns the
offset (usually 3600 or 0) or `-ENOENT` when no enabled rule is earlier than S.
Useful for logs, replay and for the application to label archived samples.

---

## 4. Runtime path

```
FTM2 LineSync capture ─┐
                       ├─> do_time_zerocross()      ─┐
FTM3 PLL tick ─────────┘                             ├─> second boundary?
                                                     │
SCM square wave ─────────> linesync_scm_sqrhandler() ─┘
                                                     │
                          dst_tick_second(secs)
                              │ secs <  next_dst_secs → return (common case)
                              │ secs >= next_dst_secs:
                              ├─ atc_dst_offset = next_dst_offset
                              ├─ next_dst_secs  = 0xFFFFFFFF   (one-shot)
                              ├─ sendTimingSignals_change()
                              └─ schedule_work(&dst_btm_wq)
                                        │
                          bottom_work_recalc_next_dst_event()
                              ├─ mutex_lock(&dst_lock)
                              ├─ find_dst_event(..., NEXT)
                              └─ next_dst_secs / next_dst_offset
```

Costs: one integer compare per TOD second in IRQ context; a table walk
(≤ 200 rules) once per transition, in process context. `dst_lock` is never
taken from an ISR. `cancel_work_sync(&dst_btm_wq)` was added to
`linesync_exit()` so unload cannot race the table.

---

## 5. Semantics you must be aware of

1. **The offset is reported, not applied.** The system clock, `/dev/tod` time
   and the RTC are unchanged at a transition; only `atc_dst_offset` changes and
   the on-change signal fires. This is exactly what the old driver did: the ATC
   application read `tzsec_offset` + `dst_offset` and computed local time.
   If you want the driver to move the clock itself (add/subtract the offset in
   `get_timeval()`/`set_timespec()` and force an RTC writeback), that is a
   separate change of roughly 2–3 h plus a re-test of the hwclock guard.
2. **The new driver still has no timezone ioctl.** The old `ATC_TOD_SET`/`GET`
   pair (`tv`, `tzsec_offset`, `dst_offset`) is absent, as is the RTC-side
   timezone storage (`setRTC_timezone_offset()` used register 0x18 of the
   M41ST85W; the ABX80x path has no equivalent). DST offsets are relative to
   whatever local time the TOD already tracks.
3. **Invalid rules are accepted but never selected.** A malformed month or
   `dow` outside 1..7 resolves to 0, which can never be a "next" event.
4. Entry `type` 0 is absolute; any nonzero value is treated as a calendar rule.

---

## 6. Verification performed

Target-kernel compilation was **not** possible in this workspace: the Yocto
kernel build tree's `Makefile` points at `/home/root/Oriux/.../kernel-source`,
which does not exist here, so `make -C <kernel build>` aborts before compiling.

What was done instead:

1. **Full-file parse** against the host 7.0 headers. The four errors reported
   (`class_create`, `struct timespec`, `do_settimeofday`, `.remove` prototype)
   are pre-existing target-kernel-4.1-vs-7.0 drift at lines 4261, 4829, 4931 and
   5591. No diagnostic points at the DST block or the new ioctl cases.
2. **Behavioural unit test** of the exact DST code, extracted verbatim from the
   driver by `tod/dst_selftest/run.sh` and linked against stubs
   (`tod/dst_selftest/harness.c`). Result: `all checks passed`. Covered:
   * US 2026 rules: next/previous transitions, offsets, winter/summer switching.
   * EU 2026/2027 rules, including the corrected "on or before" walk
     (Oct 25 2026, not a November date) and "last Sunday of March".
   * Absolute-epoch entries.
   * `GETOFFSET`-equivalent path, including the empty-table `-ENOENT` case.
   * `dst_tick_second()`: not-yet-due, due (adopt + signal + queue + disarm),
     and disabled cases.
   * Fixed-day rule, second-to-last occurrence, malformed month rejection.

Not yet verified (needs the board): module load/unload, ioctl plumbing with a
real userspace consumer, signal delivery to a listener, and a real transition
observed across a second boundary.

---

## 7. Effort estimate and justification

Hands-on engineering time; excludes idle waiting for a real transition.

| # | Task | Hours | Why it takes that long |
|---|---|---|---|
| 1 | DST state + ABI structures ported | 1.5 | Must reproduce the historical 20-byte layout exactly or the application's `GETINFO`/`SETINFO` buffers corrupt; requires checking old header semantics field by field. |
| 2 | Calendar rule resolution (fixed day, Nth DOW after, Nth DOW before, year rollover, leap years) | 3.0 | Three rule types × next/previous year resolution, plus fixing the old "before" direction and validating month/DOW ranges. Easiest place to be off by one day. |
| 3 | Next/previous event search over 100 entries | 1.5 | Two variants (current time, arbitrary time), inclusive-at-transition semantics, handling of begin vs end rules, returning the offset that belongs to the winning rule. |
| 4 | Second-boundary hooks in both tick paths | 1.0 | IRQ-safe: one compare, no sleeping, no table walk; disarming to avoid repeated signals if the workqueue is late. |
| 5 | Six ioctl handlers, validation, user copies | 2.5 | Six commands, correct `-EINVAL`/`-EFAULT`/`-ENOMEM`/`-ENOENT` behaviour, bounds checks on 100 entries, copies done outside the lock where they can sleep. |
| 6 | Workqueue + locking + lifecycle | 1.0 | `INIT_WORK`, `cancel_work_sync` on unload, `dst_lock` never taken in ISR, no allocation in the tick path. |
| 7 | Host unit-test harness | 2.5 | Extract-and-stub so the tested code is the shipped code; 6 test groups, expected epochs derived from the real 2026/2027 calendars. |
| 8 | Target build bring-up and functional test | 4.0 | Build in Yocto, insmod, `dmesg`, ioctl tool run, `GETINFO` round-trip, signal test, unload/rebind check. |
| 9 | Real transition soak | 2.0 hands-on (+24–72 h elapsed) | Arm a rule a few minutes ahead, wait across the boundary, confirm offset adoption, one change signal, and that the next event is re-armed. |
| 10 | Review, doc, hand-over | 1.5 | This document, ioctl examples, review diff. |
| | **Total hands-on** | **20.5 h** | ≈ 2.5 working days, or ≈ 1 calendar week including a 72 h soak. |

If you only need the minimal path (enable/disable/state, no table upload), items
2, 3, 5 and part of 7 shrink to about 8 h total, but a fixed-rule table is what
makes the feature usable for real DST calendars.

---

## 8. What you need to run on the target

```c
/* install a US-style rule and enable it */
struct dst_info us = {
    .enabled = 1, .type = 1,
    .begin.generic = { .month = 3, .dom_type = 1,
        .gendom.week_and_day = { 1, 2, 1 },      /* 2nd Sunday of March  */
        .dst_offset = 3600, .seconds_from_midnight_to_transition = 7200 },
    .end.generic = { .month = 11, .dom_type = 1,
        .gendom.week_and_day = { 1, 1, 1 },      /* 1st Sunday of Nov    */
        .dst_offset = 0, .seconds_from_midnight_to_transition = 7200 },
};
struct dst_info_item item = { .index = 0, .info = &us };
struct dst_infos infos = { .numinfos = 1, .dst_info_items = &item };

int fd = open("/dev/tod", O_RDWR);
ioctl(fd, ATC_TOD_DST_SETINFO, &infos);
ioctl(fd, ATC_TOD_DST_ENABLE, 0);

int state = ioctl(fd, ATC_TOD_DST_GETSTATE, 0);          /* 1   */
unsigned int now = time(NULL);
long off = ioctl(fd, ATC_TOD_DST_GETOFFSET, &now);       /* 0 or 3600 */
```

Checklist on the board:

1. `dmesg | grep -i "TOD DST"` — next transition logged after `ENABLE`.
2. `GETINFO` returns the bytes you wrote (proves ABI match).
3. `GETSTATE` flips 1 → 0 after `DISABLE`; `GETOFFSET` returns `-ENOENT`
   (`errno = ENOENT`) when the table is empty.
4. Register `ATC_TOD_REQUEST_ONCHANGE_SIG`, arm a transition a few minutes
   ahead, confirm exactly **one** signal at the boundary.
5. `rmmod` / re-`insmod` while DST is enabled — no oops, table resets to
   disabled, offset 0 (rebind safety).

---

## 9. Known gaps / recommended follow-ups

* Timezone ioctls (`ATC_TOD_SET`/`ATC_TOD_GET`) are still absent from the new
  driver; only DST offsets exist. If your application needs `tzsec_offset` and
  `dst_offset` in one call, that pair must be ported too (~3 h).
* Offset is advisory (section 5). In-driver clock shifting, if required, is
  ~2–3 h and needs a full re-test of the hwclock writeback guard.
* Persistence: nothing stores DST state across reboot. The application must
  `SETINFO`/`ENABLE` again after start-up (old driver behaved the same).
