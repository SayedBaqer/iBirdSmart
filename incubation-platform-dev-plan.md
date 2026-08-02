# iBird Hatching Company — App Development Plan

**Status:** Planning
**Author:** Sayed Baqer Alseb3 Limited
**Last updated:** 2026-07-31

## 1. Vision

A unified app for artificial egg incubation and hatching management:
multiple incubators, per-egg tracking (absorbing the existing Dry-Down
project), a shared calendar with notifications, and a **universal**
species-agnostic guidance layer first — hardware-specific integration
(DWIN/ESP32 controller) comes later as a premium layer on top.

Arabic is the primary/default language; English is fully supported as
a toggle, not an afterthought.

---

## 1a. App structure: five core branches

The app is organized around five branches (modules), each a distinct
piece of the breeder's workflow rather than one monolithic screen.
Everything else in this plan (data model, notifications, cost
economics) is the implementation detail behind these five:

| # | Branch | Purpose | Free/Paid hook |
|---|---|---|---|
| 1 | **Incubation Info** | Reference library: species incubation days, temp/humidity ranges, turning frequency — the static knowledge base (§7) | Universal library free; extended/custom profiles paid |
| 2 | **Incubation Guidance** | Contextual, step-by-step "what do I do today" coaching tied to a batch's current day (setup checklist, lockdown prep, hatch-day actions) — turns the static Info library into actionable steps | Free for core steps; detailed troubleshooting guidance paid |
| 3 | **Incubation Schedule** | The batch calendar + notifications engine (§4) — turning, candling, lockdown, hatch window, chick removal, all derived from Info + the batch's set_date | Local reminders free; cloud-synced multi-device reminders paid |
| 4 | **Egg Test List** | Candling/testing workflow: per-checkpoint checklist (day 7, day 14...) to log each egg or batch as fertile / infertile / not developing, with visual reference photos (§4a) | Batch-level pass/fail free; per-egg detailed list (Dry-Down mode) paid |
| 5 | **Store** | Link-out / cross-sell to the existing ibirdbh WooCommerce store (rings, supplies) — surfaced contextually (e.g. "order more rings" near hatch day), not a rebuilt storefront | Always free to browse; this is the acquisition/monetization bridge, not a feature tier |

Guidance (2) and Schedule (3) both read from Info (1) and are two views
over the same underlying Species Profile data — Schedule says *when*,
Guidance says *what to do and how*. Egg Test List (4) is the one place
that writes new data back (fertile/infertile counts, individual egg
status), feeding directly into the cost/performance metrics in §5.

---

## 2. Phasing strategy: PWA → Flutter

| Phase | What | Why |
|---|---|---|
| 1 | Offline-first PWA (web) | Fast to iterate, zero app-store friction, validates the concept with real breeders quickly, reuses your existing single-file-PWA pattern (ShiftCal, Maliya, Dry-Down) |
| 2 | Add lightweight backend (free tier) | Needed once you want cross-device sync and *reliable* scheduled notifications |
| 3 | Flutter rebuild (iOS + Android) | Native push notifications, background reliability, App Store/Play Store distribution, better long-term maintainability once the product is validated |
| 4 | Hardware bridge (DWIN/ESP32) | Sync live controller readings into the app — premium/specific-hardware tier |

**Important gotcha to plan around now:** iOS Safari web push only works
if the PWA is added to the home screen (iOS 16.4+), and background
timers/scheduled silent notifications are unreliable in a browser
context. Since incubation reminders are time-critical (turning
schedule, lockdown day, hatch day), this is the main argument for not
delaying Flutter too long — the web app should be treated as an MVP
and validation tool, not the permanent notification engine.

---

## 3. Data model (species/hardware agnostic)

**Batch is the primary calendar/notification unit, not the individual
egg.** In real incubation, turning, lockdown, and hatch windows happen
at the incubator/batch level — all eggs set together move through
lockdown and hatch together (a second, later-set batch in the same
incubator is simply a second Batch record).

**Two tracking modes, selected per batch:**
- **Normal (default)** — batch + species only. No per-egg weight
  loss. Just reminders through the incubation period and simple
  counts (set / fertile / hatched) plus cost economics (§5).
- **Detailed (opt-in)** — adds Dry-Down-style per-egg records
  (weigh-ins, individual candling notes, individual status) nested
  inside the batch. This is for breeders who want granular tracking
  on specific valuable/rare clutches, not the everyday case.

Most users will stay in Normal mode — that should be the lighter,
faster path, not something a per-egg data model is bolted onto.

- **Incubator** — id, name, type (`manual` | `esp32-controller` — future), location label, rated_power_watts (for cost calc, §5)
- **Batch** — incubator_id, species_profile_id, set_date (day 0), tracking_mode (`normal` | `detailed`), eggs_set, eggs_fertile (input after candling), eggs_hatched (input after hatch), egg_cost_total, notes, status (`incubating` | `lockdown` | `hatching` | `complete`), hatcher_mode (`separate_hatcher` | `same_incubator` — carried over from Dry-Down: with no dedicated hatcher, lockdown keeps temperature unchanged and only humidity/turning change)
- **Egg** (detailed mode only) — batch_id, weigh-ins, candling notes, individual status (`viable` | `not developing` | `pipped` | `hatched`)
- **Species Profile** — universal library entry: incubation_days, temp_range (forced-air vs still-air), humidity_days_1_to_lockdown, humidity_lockdown, turning_frequency, lockdown_offset_days (default 3 days before expected hatch), candling_checkpoint_days (default day 7, day 14), hatch_spread_hours (default ~24-48h — hatch is a window, not a single moment), chick_removal_offset_hours (default 24-48h after first hatch)
- **Calendar Event** — derived automatically from Batch + Species Profile (see §4 below)
- **Notification** — mapped to Calendar Events, per user device, with configurable advance-lead time

Keep Species Profile as pure data (JSON), not hardcoded logic — this
lets you seed it with generic/universal defaults now and later plug in
your own 28-species parrot list from Dry-Down without touching app code.

**Multi-batch stage conflict:** since a second, later-set batch in the
same incubator is just a second Batch record (§3 above), two batches
can end up needing different humidity at the same time — e.g. one
batch in lockdown needing 65–70% RH while another in the same
incubator is still in the early drying/incubation stage at a lower
setting. Carried over directly from Dry-Down's `stageConflict` check:
flag this in the UI rather than silently raising the shared
incubator's humidity, and suggest moving the lockdown batch to a
separate hatching container/basket instead.

---


## 4. Notification schedule (batch-level, grounded in hatching best practice)

Hatching best practice treats lockdown and hatch as *windows*, not
single instants — the notification design should reflect that rather
than firing one alert on the exact calculated day.

| Event | Default timing | Why |
|---|---|---|
| Candling check-ins | Day 7 and Day 14 of incubation (species-adjustable) | Standard checkpoints to spot non-developing eggs and remove them before they risk spoiling the batch |
| Turning reminders | Per species turning_frequency, until lockdown_offset_days before expected hatch | Stops automatically once lockdown begins |
| **Lockdown — advance warning** | 1 day before lockdown | Gives the breeder time to prep (stop turner, raise humidity, cover water) rather than being caught off guard |
| **Lockdown — day-of** | Morning of lockdown day | Actionable reminder: stop turning, raise humidity to target range, close and don't reopen |
| **Do-not-disturb reminder** | Once, during lockdown | Informational nudge — opening the incubator during lockdown drops humidity and can harm hatching chicks |
| **Hatch window — advance warning** | 1 day before expected hatch date | Expected hatch is a range (species incubation_days ± hatch_spread_hours), not a fixed hour — set expectations, not a false-precision countdown |
| **Hatch window — start** | Morning of expected hatch day | "Hatching may begin today" |
| **Chick removal reminder** | chick_removal_offset_hours after first recorded hatch (default 24-48h) | Matches best practice of leaving chicks to dry/fluff before moving to brooder, while not leaving them too long with unhatched eggs |
| **Stalled batch check** | If batch still has un-hatched eggs past expected_hatch_date + buffer | Prompts the breeder to check for pipped-but-stalled eggs rather than silently missing them |

This schedule lives entirely in Species Profile defaults (offsets, not
hardcoded dates), so it works the same whether it's chickens, ducks, or
parrots — only the day-counts differ.

---

## 4a. Egg Test List — candling checklist content

Grounds the "Egg Test List" branch in what breeders are actually
looking for at each checkpoint, so the UI can prompt a specific
observation rather than a vague "candle your eggs today":

| Checkpoint | What to look for | Log as |
|---|---|---|
| Day 7 (first candling) | Dark spot (embryo) with a radiating web of blood vessels branching outward | **Fertile** |
| Day 7 | Clear/translucent interior, no vessels, yolk visible as a plain darker shape | **Infertile — remove** |
| Day 7 | Blood ring or vessels that started then stopped/collapsed | **Early death — remove** |
| Day 14 (second candling) | Embryo fills most of the shell (mostly dark), a clearly defined, correctly-sized air cell at the large end | **Developing normally** |
| Day 14 | Little growth since day 7, air cell undersized or oddly shaped | **Stalled — flag, recheck or remove** |
| Optional day 18 (large/opaque species) | Air cell size and position ("drawdown" check) confirms hatch is on track | **On track / recheck** |

Practical logging notes carried over into the Egg Test List UI:
- Candle in a darkened room with a bright, focused light; work quickly
  to limit time out of the incubator (temperature/humidity drop risk)
- Each checkpoint should let the breeder bulk-mark "all fertile" and
  then just tap out the exceptions, rather than tapping every egg —
  faster for large batches, and matches how breeders actually candle
- Numbers logged here (fertile/infertile/removed) are exactly
  `eggs_fertile` in §3/§5 — the Egg Test List *is* the data-entry point
  for the fertility-rate and HOF metrics, not a separate log

---

## 5. Batch performance & cost economics

This is standard hatchery-management accounting, scaled down to a
hobbyist/small-breeder batch. Three inputs, entered at their natural
point in the workflow, drive everything:

1. `eggs_set` — entered when the batch starts
2. `eggs_fertile` — entered after candling (clears removed)
3. `eggs_hatched` — entered as chicks hatch / at end of hatch window

**Standard formulas** (industry-consistent terms, so results are
comparable to what breeders already know from forums/hatcheries):

| Metric | Formula | What it tells you |
|---|---|---|
| Fertility rate | `eggs_fertile / eggs_set × 100` | Breeder-flock/egg-source quality, independent of incubator performance |
| Hatch of Set (HOS) | `eggs_hatched / eggs_set × 100` | Overall result — the number breeders usually quote casually |
| Hatch of Fertile (HOF) | `eggs_hatched / eggs_fertile × 100` | *True* incubation performance — isolates the incubator/process from egg-source fertility problems, since infertile eggs were never hatchable in the first place |

Showing all three (not just one) is the best practice — HOS alone
conflates two different problems (bad eggs vs. bad incubation), and
HOF is what tells the breeder whether *their process* needs fixing.

**Worked example** (matches the natural 3-step data-entry flow):
a breeder sets `eggs_set = 300`. At day-7 candling (§4a, Egg Test List)
50 are marked infertile and removed, so `eggs_fertile = 250`. At the
end of the hatch window, `eggs_hatched = 200`.

| Metric | Calc | Result |
|---|---|---|
| Fertility rate | 250 / 300 | 83.3% |
| Hatch of Set (HOS) | 200 / 300 | 66.7% |
| Hatch of Fertile (HOF) | 200 / 250 | **80%** |
| Cost per chick | (egg_cost_total + power_cost + other_costs) / 200 | e.g. cost ÷ 200 |

Reading it: fertility (83.3%) was decent — the egg source was fine.
HOF (80%) is the number that actually judges *this* incubation run,
and it's the one that should trend upward batch-over-batch if the
breeder is improving their process; HOS (66.7%) is what most breeders
quote casually but conflates the two.

**Cost economics:**

- `egg_cost_total` — entered as a lump sum or per-egg price × eggs_set
- `power_cost` — either (a) auto-estimated from `incubator.rated_power_watts × incubation_days × 24h × local electricity rate`, or (b) manually entered if the breeder has a smart-plug/meter reading — offer both, default to the estimate
- `other_costs` (optional) — shipping, candler, etc.
- **Cost per chick** = `(egg_cost_total + power_cost + other_costs) / eggs_hatched`

This turns every batch into a simple P&L line — useful both for a
hobbyist deciding if a species is worth incubating and, later, as a
selling point over spreadsheet-based tracking.

---

## 5a. Troubleshooting guide (Guidance branch content)

This is the content that fills the **Incubation Guidance** branch
(§1a #2) beyond "what to do today" — it's "something looks off, what
now." Two tiers, matching the two tracking modes (§3):

**Normal mode — batch-level diagnosis (uses §5's own numbers):**

| Symptom | Likely cause | Points to |
|---|---|---|
| Low fertility rate, normal HOF | Egg source / breeder flock issue (age, nutrition, male:female ratio, storage before setting) | Not an incubation problem — look upstream of the incubator |
| Fertility rate fine, low HOF | Incubation process issue (temp, humidity, turning, ventilation) | The incubator/process itself — check calibration and the stage table |
| Eggs pipped but not progressing past pip (stalled) | Humidity too low in the days *before* lockdown (chick can't finish absorbing/positioning) or too low humidity *at* lockdown | Cross-check the batch's logged humidity against the lockdown target; this is what the "stalled batch check" notification (§4) should prompt the breeder to investigate |
| Whole batch dies around day 3-5 | Temperature spike/drop (power cut, thermostat drift) more than a single-cause fertility problem | Incubator hardware/calibration, not the eggs |

**Detailed mode — per-egg weight-loss diagnosis** (carried over directly
from the existing Dry-Down logic, since it's already validated with
real users — same correction hierarchy, ported as-is):

- **Losing too fast:** raise humidity first (~a few % steps), re-weigh
  in 2-3 days. Humidity is the primary lever — only nudge temperature
  (+0.1-0.2°C, to shorten time-to-pip) if humidity is already maxed out
  and the trend still won't close
- **Losing too slow:** lower humidity (or open a vent) first, re-weigh
  in 2-3 days. Only nudge temperature down (-0.1-0.2°C, to lengthen
  incubation) if humidity is already at its minimum and the trend
  still won't close
- **On target:** hold humidity steady — don't tinker with a
  well-tracking egg
- Every suggestion should log a hint to record humidity/temperature at
  each weigh-in — with 2+ logged readings the suggested correction
  becomes a calculated target instead of a rough estimate
- At lockdown: raise RH to 65-70% regardless of hatcher setup, but only
  *drop temperature* if moving to a genuinely separate hatcher —
  staying in the same incubator, temperature is left unchanged
  (`hatcher_mode`, §3) since there's no second stage to drop it for

Both tiers share the same underlying principle worth stating explicitly
in the Guidance copy: **humidity is the primary correction lever,
temperature is a last-resort, small, cautious nudge** — this should be
a hard rule in the Guidance branch's copy, not left to the breeder to
infer.

---

## 6. Free-tier-first backend strategy

Start with **zero backend** where possible:
- Local notifications via Notification API + service worker (PWA phase)
- IndexedDB (not localStorage — you'll have more data volume than
  ShiftCal/Maliya: multiple incubators × batches × eggs × weigh-ins)

When you need sync/cross-device reminders, free tiers that cover an MVP
comfortably:

| Need | Free option | Notes |
|---|---|---|
| Auth + database | Firebase (Firestore free quota) or Supabase (Postgres free tier) | Supabase gives you SQL + easier migration path later; Firebase gives you tighter push-notification integration (FCM) for the future Flutter app |
| Push notifications (Flutter phase) | Firebase Cloud Messaging | Free, and it's the standard pairing with Flutter |
| Scheduled reminders / cron | Supabase Edge Functions + pg_cron, or Firebase Cloud Functions (free quota) | Only needed once reminders must fire even if the app isn't open |
| WhatsApp/SMS premium notification channel | Your existing n8n + WhatsApp workflow | Could become a **paid-tier** differentiator later — reuse what you already built |

Recommendation: pick **one** stack early (Firebase is the lower-friction
choice given the eventual Flutter port) rather than running two in
parallel.

---

## 7. Universal incubator guidance (before going species-specific)

Ship a generic reference table as day-one content, independent of your
own birds:

- Incubation period range by species category (poultry, waterfowl,
  parrots/psittacines, etc.)
- Temp range: forced-air vs still-air incubators
- Humidity: days 1 to lockdown vs lockdown-to-hatch
- Turning frequency and the day turning stops (lockdown, typically
  last ~3 days)
- Candling schedule guidance

This becomes the default Species Profile seed data. Your specific
28-species Arabic parrot list from Dry-Down slots in later as a more
detailed override layer — no architecture change needed if the data
model is generic from the start.

---

## 8. Bilingual / RTL

- Arabic as default locale, RTL layout by default, English as toggle
  (consistent with your existing apps' pattern)
- PWA phase: simple JSON key-value i18n (same pattern as Dry-Down/Maliya)
- Flutter phase: `flutter_localizations` + `intl` — RTL is largely
  automatic once locale is set correctly, but test number formatting,
  date formatting, and icon mirroring (calendar arrows, chevrons)
  explicitly

---

## 9. Free vs. paid feature split

| Free | Paid |
|---|---|
| 1 incubator | Multiple incubators |
| Local-only reminders | Cloud-synced reminders across devices |
| Universal species library | Extended/custom species profiles |
| Basic calendar + fertility/hatch-rate (HOS, HOF) per batch | Cost-per-chick economics, export/reports, batch history analytics |
| Normal (batch-only) tracking mode | Detailed per-egg mode (Dry-Down weight tracking) |
| — | Hardware integration (DWIN/ESP32 live sync) |
| — | WhatsApp/SMS notification channel |

Keep the free tier genuinely useful (single incubator covers most
hobbyists) — it's your acquisition funnel, not a crippled demo.

---

## 10. Customer acquisition (Bahrain/GCC first, Arabic-first market)

- **Leverage ibirdbh's existing customer base** — cross-promote to
  people who already buy from your WooCommerce store; bundle with ring
  orders ("track this clutch's incubation free")
- **Community-first distribution**: Arabic bird-breeding Facebook
  groups, Instagram, Telegram channels — this audience is highly
  active there, more than app-store search initially
- **Content marketing**: short Arabic reels/tutorials on incubation
  best practices that naturally showcase the app's calendar/tracking —
  positions it as expertise-led, not just a product pitch
  - App becomes the tool that goes with the content, not the ad
- **Local partnerships**: poultry/parrot breeder associations,
  exhibitions, pet expos in Bahrain/GCC
- **Referral incentive**: e.g. an extra free incubator slot for
  referrals — word-of-mouth still dominates in niche hobbyist
  communities
- **ASO once Flutter ships**: Arabic-language keyword optimization on
  App Store/Play Store — most competing incubator apps are English-only,
  which is a real gap to exploit
- **Known English-language competitors** (HatchKeeper, Hatch Master) are
  hobbyist-hatchery batch/brooder trackers with no Arabic support and no
  Guidance branch (they log data but don't coach the breeder day-to-day)
  — the Guidance branch (§1a) plus Arabic-first is the clearest wedge

---

## 11. Suggested build order

1. **PWA MVP**: single incubator, universal species profiles, Normal-mode
   batch calendar + local notifications (lockdown & hatch-window alerts
   per §4), fertility/hatch-rate + cost-per-chick (§5), AR/EN toggle;
   Detailed per-egg mode (Dry-Down) as an opt-in toggle within a batch
2. **Multi-incubator support** + IndexedDB data model finalized
3. **Backend** (Firebase or Supabase, pick one) for cross-device sync
4. **Flutter rebuild**: same data model/logic, native push via FCM,
   publish iOS + Android
5. **Hardware bridge**: DWIN/ESP32 controller sync (premium)
6. **Growth**: community + ibirdbh cross-promotion + content marketing

---

## Open questions for next session

- Confirm: single unified app absorbing Dry-Down, or Dry-Down stays a
  separate module/import source?
- Firebase vs Supabase — final call
- Which universal species categories to seed first (beyond parrots)?
