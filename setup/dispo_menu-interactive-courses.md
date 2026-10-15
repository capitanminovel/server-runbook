# dispo_menu: interactive courses in Education (2026-10-14, dev only, main 289b148)

## What it is
Clickable courses at the top of the Education tile, next to the uploaded trainings: lessons made of short text,
callouts (tip / rule / myth), flip cards, a tap-to-match game, quizzes with instant "why", customer role-play
(pick the best reply), and a live "menu challenge" built from TODAY's menu (which product stands out for a terpene,
from the Compare data). Passing the final quiz (80%) earns the course's badge. Progress is saved per login; admins
see the team's progress (who, lessons done, final score, date finished).

Courses (content checked 2026-10-14): Customers & Minnesota Rules (342.27 ID check, visibly intoxicated, per-
transaction limits 2 oz / 8 g / 800 mg; 342.64 no health claims), Product Types & Timing (onset/duration, start low go
slow), Terpenes 101, Reading Labels & Lab Reports (Total THC = THCA x 0.877 + THC), Indica/Sativa/Hybrid (Smith 2022),
Using the Dispensary Tool.

## How it's set up
- Content is static, one file per course: `apps/admin/src/courses/content/<slug>.ts`, listed in `courses/index.ts`.
  Block types in `courses/types.ts`; activities in `courses/blocks.tsx`; drawn SVG icons in `courses/icons.tsx`;
  player `pages/CoursePage.tsx`; Education row `courses/CoursesRow.tsx`.
- Progress API `apps/api/app/routes/courses.py` (table `course_progress`): GET /progress (mine; team for admins),
  POST /{course}/lessons/{lesson} (own progress only, ids validated, max 50 lessons), GET /menu-items (the public
  menu's whitelisted Compare data for the challenge). All need login.
- Why static content and not uploads: no uploaded HTML/JS (SCORM) runs in the app -- uploads stay PDF/images/video.

## Limits / follow-up
- Progress is per LOGIN: a shared employee login shares progress.
- Rules content is not legal advice; re-check when Minnesota law/OCM rules change and update customers-and-rules.ts.
- Kiosk/customer menu don't show courses (staff only).
- Ideas: course cover art by the owner, more courses (product care/storage, cannabinoids beyond THC, store procedures).

## Also this session: "Reported, most likely" ratio (`apps/api/app/ai/ratio.py`)
Vote of the store text + pages about THIS strain (never parent pages); a source stating two different ratios doesn't
vote; a tie shows nothing and tells the admin. Saved on Generate/Redo (`strains.reported_ratio`). Not measured; COA
terpenes can't produce a ratio (labels don't track chemistry). Super Boof: tie 50/50 vs 70/30 -> none shown.

## Also: demo vs dev research, measured (17 strains, free research step, 2026-10-14)
| | Demo (Oct 5 code, 9 sites) | Dev (new code, 12 sites) |
|---|---|---|
| Avg pages about the strain | 2.1 | 3.1 |
| Strains with 0 pages | 6 | 4 (3 of them get parent pages) |
| Breeder pages / parent info / disagreements | 0 / 0 / 0 | 3 / 5 / 2 |
| Median research time | 12 s | 25 s |
Same code with dev's 12 sites gave 3.06 vs 3.12 pages: most of the page gain is the 3 added sites; the code adds
breeder/parent info and the wrong-plant / disagreement checks.

## Device guides: Puffco (2026-10-15)
- New section on the Education page, "Device guides: Puffco": Peak Pro, Peak (New 2024 + Original), Proxy (New + OG),
  Pivot. File: `apps/admin/src/courses/content/puffco.ts`. A course's optional `section` field puts it under its own heading.
- **Video block** (`type: 'video'`): plays the official Puffco video inside the lesson, with an "open in a new tab" link.
  **Why the privacy players:** YouTube links play through `youtube-nocookie.com` and Vimeo with `dnt=1`, so opening a
  lesson doesn't set tracking cookies on staff browsers. **Why `referrerPolicy` on the iframe:** YouTube now refuses
  embeds that send no referrer, and the demo site's `Referrer-Policy: same-origin` header would otherwise strip it.
- **Facts come only from puffco.com support pages** (listed at the top of puffco.ts), not from memory. The 12 video
  links were checked to be official Puffco uploads (YouTube/Vimeo oEmbed titles).
- Vimeo blocks headless test browsers ("couldn't verify the security of your connection") -- a bot check, not a bug;
  real browsers play fine. Don't try to get around it.
- To add another device brand: a new `content/<brand>.ts` with `section: 'Device guides: <Brand>'`, list it in `courses/index.ts`.
