# Docmate

Docmate is a voice-first health guardian for Indian families. A family member
speaks a sentence in their own language — English, Hindi, Hinglish, Bengali or
Tamil — and Docmate transcribes it, pulls out the structured facts (symptoms,
vitals, sleep, mood, medicines, food and water), runs them through a versioned
rule store, and turns the result into a short, plain-language card. Over time it
builds a personal health score for each person and a family score for the
household, watches every log for early risk, and speaks back in the user's
language. Severity is never guessed: it comes from a reviewed set of deterministic
rules, and the copy on every card is chosen from fixed templates.

---

## Quick start

```bash
npm install
npm run dev
```

Then open <http://localhost:3000>.

The database is a single SQLite file created on first boot at
`data/docmate.db`, and a rich 90-day demo family is seeded the first time:

- **Family demo account** — phone `9876543210` (Rohan + his father and mother,
  with ~90 days of history, a family space, sharing grants, a doctor link and a
  consultation).
- **Doctor demo account** — phone `9000000001` (signs into the doctor web view).

Login is by 6-digit phone OTP. **No SMS provider is configured out of the box, so
the OTP is shown in the Simulation Console** at `/console` (it is also written to
the server log). Enter any *new* phone number and Docmate runs the full
onboarding flow from scratch instead of loading the demo family — the demo seed
is only written while no real account exists, so your own data is never
overwritten.

Requires Node 22+ and a browser with the Web Speech API (Chrome or Edge) for the
voice path; the typed-entry fallback works everywhere.

---

## Demo script (about two minutes)

1. **Sign in** with the seeded family account `9876543210`. Read the code from
   the Simulation Console and enter it.
2. On the dashboard, look at the **two score rings** (the current person and the
   family) and the **KPI strip** below them. Switch between Rohan, Father and
   Mother with the person switcher — every tile re-renders from that person's data
   only.
3. Press the centre microphone and **record a voice journal** ("I slept five
   hours and I feel stressed"). If the mic is not available, use the **"or type"**
   fallback and type the same sentence.
4. **Read the result card**: the verdict, the personal "why" line, the guardrail
   line ("Checked against doctor-reviewed rules. Not a diagnosis.") and the
   action buttons. Note the transcript, extracted values, distress score, rule id
   and log id are all linked to this one log.
5. Open the **timeline**, find the new entry, and **correct a chip** (for example
   change the sleep hours). The correction re-runs the rule engine for the person.
6. Trigger SOS: either **say a red-flag phrase** ("I have chest pain") or press
   the **SOS** button. The screen turns red with a **15-second countdown**, and
   cancelling it logs a cancelled event plus one follow-up.
7. Open the **Simulation Console** (`/console`) to see the rows the adapters
   emitted: the OTP inbox, the **SMS / push / call / payment** outbox, the job
   queue and the current driver per service.
8. In the **Doctor** tab, **connect a doctor** and generate the auto-built
   **30-day summary PDF** (one page, real PDF, only the consented categories).
9. Open the **Family** folder: family name, family score ring, member cards and
   insights. Flip a **privacy toggle**, then open the **access log** to see the
   audit trail that change writes.
10. **Switch the language to Hindi** from the language picker and replay a log —
    the UI, the card and the spoken reply follow the new language.

---

## Architecture

One Next.js 15 app (App Router, TypeScript, Tailwind) is both the frontend and
the backend. SQLite (`better-sqlite3`) is the only datastore; there is one
process and one command.

```
Browser (mobile-first PWA)
  Web Speech API            real-time ASR: en-IN, hi-IN, bn-IN, ta-IN + Hinglish
  Web Audio API             real acoustic features from the recorded buffer
  SpeechSynthesis API       spoken replies in the user's language
  Service Worker            offline shell cache + offline recording queue
  fetch /api/v1/*           JSON API

Next.js Route Handlers (Node runtime)
  auth        OTP + HMAC-signed HTTP-only session cookie
  scope       resolvePerson(personId, accountId) -> ownership | grant, else 403
  engine      extract · rules · score · verdict · predictions · alerts · sos
  adapters    sms · push · call · payment · tts   (simulated drivers, swappable)
  store       better-sqlite3 + schema.sql + access audit
```

### Folder map

```
public/                    manifest.webmanifest · sw.js · icon.svg · icon-maskable.svg
src/
  app/
    layout.tsx             root layout (metadata, manifest link, theme colour)
    providers.tsx          app context (session, persons, language, toasts)
    api/v1/…               route handlers (auth, onboarding, persons, family,
                           doctor-links, consultations, console, analytics)
  components/
    shell/                 AppShell · MicButton · ResultCard · PersonSwitcher ·
                           SosOverlay · LanguagePicker · ServiceWorker
    charts.tsx  ui.tsx     ScoreRing, HeartbeatLine, KpiTile, Card, Sheet, toggles
  db/
    schema.sql             the whole SQLite schema (PRD 13.2)
  lib/
    db.ts                  connection, boot(), all/one/run/insert, id + date helpers
    session.ts  otp.ts     signed-cookie sessions, phone OTP with lockout
    scope.ts  audit.ts     person scope guard + access audit trail
    nlu/
      lexicon.ts           multilingual red flags, symptoms, mood, stress, vitals
      extract.ts           deterministic transcript -> structured JSON
    rules/
      engine.ts            the evaluators (pure over one person's data)
      rules.json           the versioned rule store (source of truth for seed)
    analysis.ts            DB -> engine bridge; scores, features, baselines
    score.ts  verdict.ts   health score maths, constrained card writer
    rules (store)          seeded into the `rules` table on boot
    alerts.ts  sos.ts      notification policy, SOS state machine
    predictions.ts         trend / recurrence / adherence forecasts
    pdf.ts                 one-page 30-day summary (pdf-lib)
    adapters/              sms · push · call · payment · tts drivers + registry
    seed.ts                rules, doctor directory, 90-day demo family
    i18n/                  index.ts + en · hi · bn · ta JSON
  tests/                   score · extract · rules · isolation
```

### Health score

```
H = 100 * sum(w_i * c_i) / sum(w_i)  -  P        clamped to 0..100
F = 0.7 * mean(H_shared) + 0.3 * min(H_shared)
```

Components, weights and windows: Vitals 0.30 / 14 days · Sleep 0.20 / 7 days ·
Mood and voice 0.20 / 7 days · Activity 0.15 / 7 days · Medicine adherence 0.15 /
7 days. The score renormalises over only the components that have data; fewer than
two components with data shows "Starting" (no number). The penalty `P` comes from
unresolved alerts in the last 7 days — L2 = 3, L3 = 8, L4 = 20 — capped at 25.
The displayed score is smoothed with a 3-day EMA; a fresh L4 alert bypasses
smoothing. A single sharer means the family score is simply that person's score.

### SOS state machine

```
start -> countdown (15 s, client-owned clock) -> escalate -> resolved | cancelled
```

Escalation sends SMS + push and places one automated call to the emergency
contacts in priority order. The event row is written and the full-screen SOS opens
before any network call, so weak connectivity degrades to queued messages rather
than a blocked screen. Acoustic distress alone can never trigger SOS.

### No LLM decides severity

Severity is produced **only** by the versioned rule store (`rules.json` seeded
into the `rules` table), never by a model. Extraction is a deterministic
multilingual lexicon/grammar engine with no network calls. Card copy is chosen
from templates and is structurally unable to change a severity, name a medicine
or give a dose. Every alert and card records its `rule_id`, `rule_version` and
`evidence_json`, which is what makes each one auditable.

---

## Going live on each external service

Every third-party wire sits behind an adapter. The active driver is chosen purely
by whether the corresponding key is present, so **dropping a key into `.env.local`
flips the service from simulated to live with no code change**:

```
SMS_API_KEY=...        emergency + invite SMS
PUSH_API_KEY=...       alert notifications
CALL_API_KEY=...       automated SOS call
PAYMENT_API_KEY=...    tele-consult booking
TTS_API_KEY=...        cloud text-to-speech (browser TTS is the fallback)
```

With no key, the simulated driver records the exact payload it would have sent
into the `outbox` table, which the Simulation Console renders. The Console's
adapter-status panel shows which driver is active per service.

---

## Testing

```bash
npm test           # vitest: score · extract · rules · isolation
npx tsc --noEmit   # full type check
npm run build      # production build
```

- `score.test.ts` — golden cases for the score maths, penalty cap, EMA smoothing
  and the family formula.
- `extract.test.ts` — symptom, vital, sleep and stress extraction across English,
  Hindi, Bengali, Tamil and Hinglish.
- `rules.test.ts` — the 9 seeded rules, a level-4 BP-01 emergency, the audit
  fields on every outcome, and the `min_data` gate.
- `isolation.test.ts` — cross-account reads are refused, a family grant opens only
  the shared categories, and revoking the grant closes access again.

---

## Privacy

Every health-bearing table carries `person_id`, and every person-scoped read goes
through the scope guard, so a request for a person you do not own or that has not
been shared with you returns **403** with the standard error envelope. Cross-person
reads are written to the `access_audit` trail, which the profile owner can see.
Journal audio, journal text and cycle data are **never** shared with family —
even a full family grant exposes only score, alerts, vitals, reports and family
insights. A doctor link is scoped by category and duration and is enforced and
audited on every access. No health content is written to logs.
