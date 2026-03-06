# Lesson: Playroom Button Semantics vs System Architecture

**Category:** Messaging / Product Design
**Applies to:** 42Bros Playroom simulation UI

---

## Problem

A UI button labeled after a service (e.g., "YOSHI") may imply to the user that clicking it simulates only that service's contribution. But the downstream system (Mario) evaluates all trade conditions against full state regardless of which button triggered the event.

**Result:** The user expects 1 feed entry (rejection), but receives N entries (one per TC evaluated). The user expects a Telegram notification to the YOSHI channel from Yoshi itself, but Yoshi doesn't send Telegram notifications — that's Toad's job, and Toad only sends for Mario decisions.

**This failure mode only surfaces during live user E2E testing.** Unit tests pass, CI passes, the system behaves correctly per spec — but the spec did not match the user's mental model.

---

## Example

**42Bros — block-12-06 Playroom YOSHI button (2026-03-06):**

- User clicked YOSHI button once
- Expected: 1 feed entry (REJECTED because only Yoshi fired) + Yoshi-channel Telegram notification
- Actual: 5 feed entries (one per TC) + 2 WOULD_EXECUTE notifications to MARIO channel (for TCs with all-YOSHI triggers)

**Root causes:**
1. Yoshi's normal workflow does not publish to `42B_NOTIFICATIONS` — it only publishes data to `42B_YOSHI`. So "YOSHI-channel notification" is architecturally impossible from this path.
2. Mario evaluates all TCs against current real state. 2 TCs have all-YOSHI trigger requirements — they pass when Yoshi data is current, regardless of whether PEACH/NABBIT also fired.
3. The block spec said "Mario evaluates → Telegram [TEST]" without specifying expected result or channel.

---

## Prevention

Before implementing any "simulate service X" button, the spec must define:

1. **What Telegram channel should receive a notification?** (YOSHI? MARIO? Neither?)
2. **What should Mario's evaluation result be?** (Always REJECT when triggered from a single-button? Or evaluate against real state?)
3. **How many feed entries per button click?** (1? N per TC? Grouped by sim_run_id?)
4. **Does the service itself send a notification, or only Mario?** (Depends on whether the service normally uses 42B_NOTIFICATIONS)

Without these decisions in the spec, the implementation will be technically correct but will fail user acceptance.

---

## 42Bros Specifics

- Yoshi, Peach, Nabbit do NOT send Telegram notifications directly — only Mario/Toad do (via `42B_NOTIFICATIONS`)
- Mario evaluates ALL active TCs every time a Yoshi/Nabbit event fires — not just TCs related to the triggering service
- TCs with all-YOSHI triggers will WOULD_EXECUTE when Yoshi data is present, even if no PEACH/NABBIT event fired
- "Yoshi-channel Telegram notification" would require Yoshi to publish to `42B_NOTIFICATIONS`, which it currently does not do

---

**Last Updated:** 2026-03-06
**Related:** `meta/logs/engineering/log-260306-0020-block-12-06-playroom-v01.md`
