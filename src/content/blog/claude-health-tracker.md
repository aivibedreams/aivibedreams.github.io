---
title: "I Built a Health Tracker That Lets Me Chat With My Own Data"
description: "Claude Health is a personal health app with a twist — all your data lives in plain text files so your AI assistant can actually read and reason about it."
date: 2026-04-13
tags: [health, ai, claude, nextjs, typescript, fitbit]
---

Most health apps keep your data locked inside their own walled garden. Your Fitbit data lives in Fitbit's cloud. Your workout notes are in one app. Your body measurements are in another. You might export a CSV once a year if you're determined enough, but day to day? You can't really *talk* to your data.

**Claude Health** was built to fix that. It's a local web app that logs everything — Fitbit syncs, workouts, meals, body measurements — and stores it all as simple text files that an AI can read directly. Then you ask Claude Code to open the project folder and you just... talk to your data.

> "What patterns do you see between my sleep quality and my workout performance?"  
> "Am I making progress on my key lifts?"  
> "Flag any meals that might have caused my low energy days."

That's the dream. Here's how it got built.

---

## The Problem With Health Apps

Health tracking apps are genuinely useful, but they have a structural problem: your data is trapped inside them.

You can look at graphs, see trends, maybe get a weekly summary. But the analysis is pre-packaged. The app only shows you what its developers thought you'd want to see. If you have a specific question — or if you want to understand a *correlation* between two different things — you're stuck.

Exporting data is usually possible but painful. Even if you get the CSVs out, then what? Paste them into ChatGPT? Try to remember what you ate three Thursdays ago?

The alternative: what if you built a local app where the data was *always* in a format an AI could read, stored right there on your own machine?

---

## The Idea: Flat Files as the Interface

The key design decision behind Claude Health is deceptively simple:

**All data is stored as CSV, JSON, or JSONL files in a `data/` folder.**

No database. No API. No cloud sync required. Every time you log something, a line gets appended to a text file.

```
data/
  daily_metrics.csv       ← weight, energy, mood, stress, water
  workouts.jsonl          ← gym sessions with sets, reps, weights
  body_metrics.csv        ← body measurements over time
  fitbit_daily.csv        ← steps, calories, HR zones (from Fitbit)
  fitbit_sleep.csv        ← sleep hours, deep/REM/light breakdown
  meals.csv               ← meal log with photo references
  youtube_sessions.jsonl  ← home workouts from video
  diary.jsonl             ← analyses and notes
  profile.json            ← height, DOB, sex — static reference data
```

Claude Code can open the project directory and read any of these files directly. When you ask it to analyse your data, it reads the actual files — not a summary, not a cached snapshot. The real, current data.

This turns Claude into something more than a chatbot. It becomes a health analyst with access to your complete, longitudinal health record.

---

## What the App Does

### Daily Metrics & Fitbit Sync

The dashboard gives you a quick overview of today and lets you log your core daily vitals: energy level (1–10), mood, stress, water intake, and weight.

Fitbit data gets pulled in via OAuth2. Once you've connected your account (a one-time setup), clicking **Sync** pulls steps, calories, active minutes, heart rate zones, floors climbed, and sleep data for the last 90 days. It uses upsert logic — re-syncing updates existing rows rather than duplicating them.

Sleep data comes from Fitbit's v1.2 API, which gives you a breakdown of deep, REM, light, and awake minutes — the detail that matters for recovery analysis.

### Workout Tracking — Two Systems

There are two separate workout systems because gym sessions and home video workouts are fundamentally different things.

**Gym sessions** are logged as structured JSONL: each session records the exercises, sets, reps, and weight lifted. This format is ideal for progressive overload analysis — Claude can track whether you're lifting more over time on each exercise.

**Home workouts** use YouTube playlists. You paste in a playlist URL (like a DAREBEE bodyweight channel), click **Sync**, and the app uses `yt-dlp` to extract all the video titles and IDs. No YouTube API key needed, no quota to worry about.

From there you can browse the video library, build workout plans from your available videos, and run them through a built-in player. The player loads each video in sequence, shows a timer counting up, and lets you mark each one complete before moving on.

### Body Measurements

The metrics page handles two things:

**Profile** (one-time): height, date of birth, sex, and preferred units. This gets saved to `profile.json` and is used to calculate derived stats.

**Body measurements** (logged periodically): weight, body fat percentage, and seven circumference measurements — chest, waist, hips, both arms, both thighs. As you type, the page calculates your BMI and waist-to-height ratio live, colour-coded against longevity targets.

### The AI Diary

This is where the app really comes into its own. After you ask Claude Code to analyse your data, it can save its analysis directly to `data/diary.jsonl`. The Diary page in the app displays these entries with full markdown rendering — tables, code blocks, bullet points — in a clean two-panel layout.

The idea is that over time you build up a personal health journal of AI-assisted analyses. You can browse back through them, compare to where you are now, and track whether recommendations actually worked.

---

## Building It With Claude Code

The whole app was built collaboratively with Claude Code — not just the code, but the design decisions, debugging, and even the workout plans.

A few things that emerged from that process:

**The flat-file architecture came from the AI's needs, not mine.** I initially thought about using SQLite, but when I thought about how Claude would actually query the data, plain text files were clearly better. No schema to explain, no query language to learn. Claude just reads the files.

**Bugs that would have taken hours took minutes.** When Fitbit sync was producing all-zero data, Claude traced it to three separate root causes: wrong API endpoint format, wrong field names, and the sleep endpoint requiring a different API version (v1.2 vs v1.0). It fixed all three in one pass.

**Claude generated the workout plans.** After syncing 200+ videos from two YouTube playlists, I asked Claude to create 5 holistic workout plans. It read all the video titles, categorised them by type (warm-up, push, pull, squat, core, mobility, HIIT, cooldown), and created 5 balanced plans: push/core, lower body, mobility, HIIT combat, and pull/balance.

**The iteration speed is genuinely different.** The whole app — dashboard, Fitbit OAuth, workout player, diary, body metrics — was built across a few sessions. Features that would have been multi-day projects solo were done in hours.

---

## Technical Deep Dive

*This section is for developers who want to understand the implementation.*

### Stack

- **Next.js 15** (App Router) + TypeScript
- **Tailwind CSS** — dark theme, CSS custom properties
- **No database** — all I/O goes through a `lib/storage.ts` utility layer
- **Fitbit OAuth2** — PKCE flow, access/refresh tokens stored in `data/.tokens.json`
- **yt-dlp** — spawned as a child process, stdout parsed line-by-line for JSON metadata

### Key Storage Patterns

`lib/storage.ts` exposes four primitives:

- `csvAppend(file, row, headers)` — appends a row, writing headers on first write
- `csvRead(file)` — returns all rows as an array of objects
- `csvUpsert(file, row, headers, keyField)` — updates an existing row by key or appends if not found (used for Fitbit sync to avoid duplicates)
- `jsonlAppend(file, obj)` — appends a JSON object as a newline

This layer is the only thing that touches the filesystem. Routes import from here, not from `fs` directly.

### Route Exports Gotcha

Next.js App Router routes only allow HTTP method exports (`GET`, `POST`, etc.). Any helper function exported from a route file causes a build error. The solution is to put all shared types and helpers in `lib/` files:

- `lib/profile.ts` — `Profile` interface, `readProfile()`
- `lib/playlists.ts` — `Playlist`, `VideoInfo`, `readPlaylists()`, `writePlaylists()`
- `lib/workout-plans.ts` — `WorkoutPlan`, `readPlans()`, `writePlans()`

### Fitbit Sync Details

Fitbit's Activities API uses a date-range endpoint format: `/activities/{resource}/date/{start}/{end}`. Seven separate requests run for each sync:

1. `activities/steps` 
2. `activities/activityCalories`
3. `activities/minutesFairlyActive`
4. `activities/minutesVeryActive`
5. `activities/floors`
6. `activities/heart` — returns HR zones including resting heart rate
7. `sleep` — from v1.2 base URL, returns full sleep stage breakdown

All results are merged by date and upserted into `fitbit_daily.csv` and `fitbit_sleep.csv`.

### YouTube Playlist Sync

```ts
const proc = spawn('yt-dlp', [
  '--flat-playlist',
  '--dump-json',
  '--no-warnings',
  '--no-playlist-reverse',
  url,
])
```

stdout is collected line-by-line and each line parsed as JSON. The title and ID are extracted, de-duplicated (some playlists contain the same video twice), and written to `playlists.json`. If yt-dlp isn't installed, the error is caught and surfaced gracefully with installation instructions.

### CLAUDE.md as the AI's Context

The project includes a `CLAUDE.md` file that Claude Code reads automatically when you open the project. It contains:

- User profile (age, goals, dietary notes, wearable)
- A manifest of all data files and what they contain
- Longevity target ranges for each metric (resting HR, sleep %, body fat, waist-to-height ratio)
- Instructions for how to analyse the data
- Example prompts

This means every conversation starts with Claude already knowing the context. You don't re-explain your goals every time.

---

## What's Next

A few things on the list:

- **Progress charts** — visualising body metrics and Fitbit trends over time directly in the UI
- **Notification/reminder system** — prompts to log if a day is missed
- **Photo gallery** — proper image viewer for logged meals
- **Export to markdown** — generate a shareable health summary document

The core loop is working well though: log data, sync Fitbit, ask Claude to look at everything, save the analysis to the diary, repeat.

---

## Try It Yourself

The project is designed to be run locally. You'll need Node.js 18+ and `yt-dlp` installed for playlist sync.

```bash
git clone https://github.com/aivibedreams/claude-health
cd claude-health
npm install
npm run dev
```

For Fitbit sync you'll need to register a Personal app at [dev.fitbit.com](https://dev.fitbit.com/apps/new) and add your credentials to `.env.local`. Full setup instructions are in the README.

Then open Claude Code in the project directory and start asking questions.
