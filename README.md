# Tekken Score Overlay

A lightweight scoreboard overlay for **OBS**, built for Tekken streams. It draws a
name + score bar near the top‑left and top‑right of the screen and updates **live**
from a small web‑based control panel. State is shared through a **Supabase** table,
so an operator can change names, scores, the score‑box colour, and overlay
visibility from any browser and have it reflected on stream instantly.

No build step and no framework — just static HTML/CSS/JS plus the Supabase JS
client loaded from a CDN.

![overlay position: score bars flank the game's round timer along the top edge](docs/preview.png)
<!-- optional: drop a screenshot at docs/preview.png, or delete this line -->

---

## Features

- **Live score bars** — `P1_Name P1_Score` top‑left, `P2_Score P2_Name` top‑right, in the [Anton](https://fonts.google.com/specimen/Anton) display font.
- **Control panel** (`lops.html`):
  - Editable P1/P2 **names** and **scores**.
  - **Push Changes** — writes to the overlay behind a confirmation modal; your text boxes are left untouched.
  - **Reset** — restores defaults: `Player 1 / 0`, `Player 2 / 0`, score box red, visibility on.
  - **Score box colour** picker (swatch + hex) — the score **number** auto‑switches between white and black for contrast.
  - **Overlay visible** toggle — slides the bars up and off‑screen via a CSS transition. Applies instantly (independent of Push).
- **Realtime sync** through Supabase, with a **4‑second polling fallback** so the overlay keeps updating even if the realtime socket drops.
- **Fixed‑width name plates** for left/right symmetry; names too long to fit **scroll like a marquee**.
- **Trapezoidal** bar shape; every position / size / colour value is a CSS variable at the top of `index.html`.
- **16:9 guard** — if the browser source isn't a 16:9 aspect ratio, the overlay shows a message asking you to fix the resolution.

---

## How it works

Two static pages share a **single row** (`id = 1`) in a Supabase table called `overlay`:

```
 lops.html  ──(UPDATE)──▶  Supabase `overlay` row  ──(SELECT + Realtime)──▶  index.html
 (control panel)              (shared state)                                   (OBS source)
```

- **`lops.html`** only writes. It reads the current values once on open to pre‑fill the boxes.
- **`index.html`** only reads. It fetches on load, subscribes to realtime changes, and also polls every 4 s as a safety net, then re‑renders the bars.

---

## Project structure

```
tekken-score-overlay/
├── index.html          # The OBS overlay (read-only display)
├── lops.html           # The control panel (writes to Supabase)
├── fonts/
│   └── Anton-Regular.ttf
└── README.md
```

---

## Setup

### 1. Create the Supabase table

In your Supabase project, open **SQL Editor** and run:

```sql
-- Single-row table (id is always 1)
create table if not exists public.overlay (
  id          integer primary key default 1,
  p1_name     text    not null default 'Player 1',
  p1_score    integer not null default 0,
  p2_name     text    not null default 'Player 2',
  p2_score    integer not null default 0,
  score_color text    not null default '#D61F26',   -- score BOX colour
  visible     boolean not null default true,
  constraint overlay_single_row check (id = 1)
);

-- Seed the one row
insert into public.overlay (id) values (1) on conflict (id) do nothing;

-- Row Level Security: allow the public anon key to read + write this table
alter table public.overlay enable row level security;
create policy "overlay anon read"   on public.overlay for select using (true);
create policy "overlay anon insert" on public.overlay for insert with check (true);
create policy "overlay anon update" on public.overlay for update using (true) with check (true);

-- Enable instant updates (the overlay also polls every 4s as a fallback)
alter publication supabase_realtime add table public.overlay;
```

### 2. Add your credentials

Copy your **Project URL** and **anon public key** from Supabase → **Settings → API**,
then paste them into the credentials block at the top of the `<script>` in **both**
`index.html` and `lops.html`:

```js
const SUPABASE_URL      = 'https://YOUR-PROJECT.supabase.co';
const SUPABASE_ANON_KEY = 'YOUR-ANON-KEY';
```

### 3. Add the overlay to OBS

1. Add a **Browser** source → **Local file** → select `index.html`.
2. Set the source **Width/Height** to a 16:9 resolution (e.g. `1920×1080` or `1280×720`).
3. The background is transparent, so it sits over your gameplay capture.

### 4. Open the control panel

Open `lops.html` in any browser (on the same or a different machine). Edit the
values and hit **Push Changes**.

> **Tip:** after editing a file, refresh the OBS browser source (right‑click →
> *Refresh*) and hard‑refresh the control panel (`Ctrl` + `F5`).

---

## Customisation

All layout knobs live in `:root` at the top of `index.html` (units are `vh`/`vw`
so they scale with any 16:9 resolution):

| Variable        | Default   | What it does                                            |
| --------------- | --------- | ------------------------------------------------------- |
| `--bar-top`     | `0vh`     | Gap from the top edge of the screen                     |
| `--center-gap`  | `13vw`    | Distance from screen centre to each bar's inner edge    |
| `--bar-height`  | `6.4vh`   | Thickness of the bars                                   |
| `--slant`       | `2.4vh`   | Trapezoid angle on the inner end (`0` = plain rectangle)|
| `--name-size`   | `3.4vh`   | Player‑name font size                                   |
| `--score-size`  | `4.2vh`   | Score‑number font size                                  |
| `--name-width`  | `20vw`    | Fixed black‑plate width; longer names scroll (marquee)  |
| `--accent`      | `#d61f26` | Dividers / ratio‑gate accent                            |
| `--bar-bg`      | near‑black| Name plate background                                   |
| `--name-text`   | `#ffffff` | Player‑name colour                                      |
| `--score-bg`    | `#d61f26` | Score box colour (overridden live by the colour picker) |
| `--score-text`  | `#ffffff` | Score number colour (set automatically for contrast)    |

Other constants in the script: `ROW_ID` (which row to read/write, default `1`) and
`POLL_MS` (fallback poll interval, default `4000`).

---

## Troubleshooting

- **"Please set the viewport resolution to a 16:9 aspect ratio"** — the browser
  source isn't 16:9. Set it to `1920×1080`, `1280×720`, etc. (This message is
  expected when previewing in a normal, non‑16:9 browser window.)
- **Console: `Could not find the table 'public.overlay' in the schema cache`
  (PGRST205)** — PostgREST's schema cache is reloading, which happens briefly after
  any DDL (`ALTER`, etc.). It clears on its own; the overlay keeps the last values
  and recovers on the next poll. To force it: run `NOTIFY pgrst, 'reload schema';`.
- **Push / toggle does nothing** — make sure the credentials are filled in and the
  SQL above has been run (the `score_color` and `visible` columns must exist).
- **Long name isn't scrolling** — it only scrolls when the text is wider than
  `--name-width`; increase font size or reduce `--name-width` to test.

---

## Security note

The **anon key is a public, client‑side key** — it's designed to be shipped in the
browser, so committing it here is expected. However, the RLS policies above allow
**anyone with the key to read and update** the score row. For a personal stream
overlay that's usually fine, but be aware that if this repo is public, anyone could
change your on‑stream score. If you want to lock writes down, restrict the `update`
/ `insert` policies (e.g. require an authenticated user) and keep `lops.html`
private, or move the key handling behind your own auth.

---

## Tech stack

- Plain HTML / CSS / JavaScript (no build step)
- [Supabase](https://supabase.com/) — Postgres + Realtime + REST
- [Anton](https://fonts.google.com/specimen/Anton) font by Vernon Adams, licensed under the [SIL Open Font License 1.1](https://openfontlicense.org/)

---

## Notes

This is a fan‑made streaming tool and is not affiliated with or endorsed by Bandai
Namco. "Tekken" is a trademark of its respective owner.
