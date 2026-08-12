# Family Agent

A local AI agent that runs on your Intel NUC and:

1. **Combines calendars** — pulls Google Calendar and Apple (iCloud) Calendar
   events from every family member into one unified family calendar.
2. **Notifies everyone** — sends each person Telegram reminders for their own
   events, plus a daily digest of the whole family's agenda.
3. **Recommends vacation activities** — when it spots a trip/vacation on
   anyone's calendar, asks Claude for detailed, locally-specific activity
   recommendations and sends them to whoever's going.
4. **Plans healthy meals** — generates a weekly meal-prep plan per person,
   tailored to their dietary goals/restrictions and how busy each evening
   actually is that week.

It's built as a Python service today, with an eye toward wrapping it in a
proper app (mobile/web dashboard) later — the sync/notify/recommend logic is
already decoupled from Telegram, so a different front-end can reuse it.

## How it's built

- **Windows Scheduled Task** keeps it running in the background on the NUC
  (see "Run it as a background service" below).
- **Google Calendar** via OAuth (`google-api-python-client`) — read-only.
- **Apple Calendar** via iCloud **CalDAV** (`caldav` + `icalendar`) — the NUC
  isn't a Mac, so there's no local EventKit access; this is the standard
  third-party way to read iCloud calendars, using a per-person
  app-specific password.
- **Telegram** for notifications — free, works identically on iOS/Android,
  and each family member just messages a bot.
- **Claude API** (`claude-opus-5`) for the vacation and meal-prep
  recommendations, called through the official `anthropic` Python SDK.
- **SQLite** tracks what's already been sent so restarts never spam anyone.
- **APScheduler** runs the sync/notify loop on an interval.

```
config/family.yaml     <- who's in the family, their calendars, dietary info
.env                    <- secrets (API keys, app passwords)
src/family_agent/
  calendars/            <- Google + Apple sync, merging into one list
  notify/telegram.py     <- push notifications + /today, /week commands
  recommend/             <- Claude-powered vacation & meal-prep generation
  storage/db.py           <- SQLite dedupe/cache
  scheduler.py            <- what runs on each sync cycle
  main.py                 <- entry point
scripts/
  authorize_google.py            <- one-time OAuth per family member
  run.ps1                        <- launches + auto-restarts the agent
  register_windows_task.ps1      <- installs it as a background task
```

---

## 1. Prerequisites

- Python 3.11+ on the NUC (Windows). Get it from python.org and check "Add
  to PATH" during install.
- A Telegram account for each family member.
- A Google account and (for Apple users) an iCloud account per member.
- An Anthropic API key: https://console.anthropic.com/settings/keys

## 2. Install

```powershell
cd family-agent
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
```

## 3. Create the Telegram bot

1. In Telegram, message **@BotFather** → `/newbot` → follow the prompts.
   BotFather gives you a token like `123456789:AAExampleTokenTextGoesHere`.
2. Copy `.env.example` to `.env` and set `TELEGRAM_BOT_TOKEN`.
3. Have **every family member** open a chat with the new bot and send
   `/start`. That's what lets the bot message them.
4. Find each person's chat ID: with the bot token set, run:

   ```powershell
   curl "https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates"
   ```

   Each `/start` shows up with `"chat":{"id": 123456789, ...}` — that
   number is what goes in `telegram_chat_id` in the config below.

## 4. Google Calendar setup

Google requires an OAuth "app" (free, just your own project) before anyone
can grant calendar access:

1. Go to https://console.cloud.google.com/ → create a new project (e.g.
   "Family Agent").
2. **APIs & Services → Library** → enable **Google Calendar API**.
3. **APIs & Services → OAuth consent screen** → User type **External** →
   fill in an app name/support email → under **Test users**, add every
   family member's Gmail address (this keeps the app private to your
   family without needing Google's app-review process).
4. **APIs & Services → Credentials → Create Credentials → OAuth client ID**
   → Application type **Desktop app** → download the JSON.
5. Save that file as `config/google_client_secret.json`.

Each member then authorizes once:

```powershell
.venv\Scripts\python scripts\authorize_google.py Sam
```

This opens a browser for **Sam** to sign into **Sam's own Google account**
and grant read-only calendar access. It writes a personal token file (path
set by `token_file` in the config) — repeat per member. It only needs a
browser once; after that the agent refreshes tokens on its own.

## 5. Apple (iCloud) Calendar setup

For each family member who uses Apple Calendar:

1. Go to https://appleid.apple.com → sign in → **Sign-In and Security** →
   **App-Specific Passwords** → generate one, name it "Family Agent".
2. Put it in `.env` under the variable name referenced by
   `icloud_app_password_env` for that person (see `.env.example`).

No app registration is needed on Apple's side — CalDAV with an app-specific
password is Apple's standard sanctioned way for third-party tools to read a
personal iCloud calendar.

## 6. Configure your family

```powershell
copy config\family.example.yaml config\family.yaml
```

Edit `config/family.yaml`: add each member's name, Telegram chat ID, which
calendars they use, and their dietary goals/restrictions/allergies (used for
the meal-prep plans). See the comments in the file for the schema.

## 7. First run (foreground, to check everything works)

```powershell
.venv\Scripts\python -m family_agent.main
```

You should see it sync calendars and log activity to `logs/family_agent.log`.
Message the bot `/today` from any family member's Telegram to sanity-check
the combined agenda. Stop it with Ctrl+C once you're satisfied.

## 8. Run it as a background service

In an **elevated** ("Run as Administrator") PowerShell prompt:

```powershell
powershell -ExecutionPolicy Bypass -File scripts\register_windows_task.ps1
```

This registers a Scheduled Task (`FamilyAgent`) that starts when you log
into the NUC and keeps itself running (`scripts\run.ps1` restarts it
automatically if it ever crashes). Start it immediately without rebooting:

```powershell
Start-ScheduledTask -TaskName FamilyAgent
```

If your NUC is set to auto-login, this means the agent effectively starts on
boot. To view/stop it: Task Scheduler → Task Scheduler Library → `FamilyAgent`.

---

## Notes on cost

Claude is only called when something new needs recommendations — a newly
detected trip, or once a week for meal plans (both are cached in SQLite so
the same trip/week never re-triggers a call). Calendar sync itself doesn't
touch the Claude API at all, so cost stays low even syncing every 10 minutes.

## Tuning

All of this lives in `config/family.yaml`:

- `sync_interval_minutes` — how often calendars are re-checked.
- `notifications.minutes_before_event` — e.g. `[60, 10]` for a heads-up and a
  final reminder.
- `notifications.daily_digest_time` — when the whole-family agenda goes out.
- `recommendations.vacation_keywords` — what marks an event as a trip.
- `recommendations.meal_prep_day` / `meal_prep_time` — when the weekly plan
  is generated and sent.

## Roadmap ideas (not built yet)

- A small web/mobile dashboard reading the same SQLite DB + calendar sync
  layer, instead of (or alongside) Telegram.
- Two-way sync (currently read-only by design, to avoid ever double-booking
  someone's real calendar).
- Push the merged family calendar back out as its own subscribable calendar
  feed (ICS) so it shows up natively in each person's calendar app too.
