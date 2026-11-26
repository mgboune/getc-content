---
name: getc-content
description: Generates GETC (Gakushin Eikou Training Center) slot-aware Burmese Facebook content and visual media (0800, 1200, 1400) for Hledan and Kamayut pages, with 1-day advance holiday cards, interactive Japanese multiple-choice quizzes, and 30s/40s Reels/TikTok 10s CapCut video clips. Supports flexible CLI arguments (--date, --slot, --regenerate / -force, or bare positional values).
---

# GETC Daily Content & Visual Engine (V4.0 — Slot-Aware & Daily Folder Pipeline)

Execute this skill when the user invokes `/getc-content` (with any combination of optional flags or positional arguments).

## Command Invocation Examples (All Supported)
- `/getc-content` -> Auto-detects today's date (Asia/Yangon `+06:30`) and current slot (`0800`, `1200`, or `1400`).
- `/getc-content 1400` or `/getc-content --slot 1400` -> Today's `1400` slot.
- `/getc-content 2026-11-26 1400` or `/getc-content --date 2026-11-26 --slot 1400` -> Specific date and slot.
- `/getc-content 2026-11-26 1400 --regenerate` or `/getc-content 2026-11-26 1400 -force` -> Force-regenerates markdown, images, and video clips for that slot with fresh topics/sources.

## Execution Rules

1. **Tool budget & speed**: Execute the entire pipeline in **at most 2–3 shell commands** (`prepare_context.py` -> LLM JSON generation -> `generate_posts.py` + `render_images.py`). Do NOT inspect or modify protected scripts (`chmod 444`).
2. **Script protection**: Never rewrite or `chmod` `prepare_context.py`, `generate_posts.py`, `render_images.py`, `categories.txt`, `sources.txt`, or `holidays.json`.
3. **Final Output Format**:
   - First, output **PART 1: COPY-READY CONTENT** verbatim from `/root/GETC/Marketing/FB_Posts/YYYY.MM.DD/SLOT-chat-ready.md` (with its fenced ` ```text ` / ` ```json ` blocks so the user can copy-paste cleanly).
   - Immediately after PART 1, append the **Execution & Diagnostic Report** printed by `render_images.py` (showing generated image/video paths, file sizes, AI providers used, font pairs used, and any diagnostics).

---

## Step 0: Parse Flexible CLI Arguments, Check Daily Folder Cache & Run Pre-Processor

Run ONE shell command to parse any order of `--date`/`-d`, `--slot`/`-s`, `--regenerate`/`-force`/`--force`/`-f`, or bare positional `YYYY-MM-DD` / `0800|1200|1400` tokens:

```bash
python3 - <<'PY'
import datetime, re, subprocess, sys
from pathlib import Path

raw_args = """$ARGUMENTS""".split()
tz_mm = datetime.timezone(datetime.timedelta(hours=6, minutes=30))
now_mm = datetime.datetime.now(tz_mm)

date_arg = None
slot_arg = None
force_flag = False

i = 0
while i < len(raw_args):
    tok = raw_args[i].strip()
    low = tok.lower()
    if low in ("--regenerate", "-regenerate", "regenerate", "--force", "-force", "-f", "force"):
        force_flag = True
    elif low in ("--date", "-d") and i + 1 < len(raw_args):
        date_arg = raw_args[i + 1].strip()
        i += 1
    elif low.startswith("--date="):
        date_arg = tok.split("=", 1)[1].strip()
    elif low in ("--slot", "-s") and i + 1 < len(raw_args):
        slot_arg = raw_args[i + 1].strip()
        i += 1
    elif low.startswith("--slot="):
        slot_arg = tok.split("=", 1)[1].strip()
    elif re.fullmatch(r"(0800|1200|1400)", tok):
        slot_arg = tok
    elif re.search(r"\d{4}[-./]\d{2}[-./]\d{2}", tok):
        date_arg = tok
    i += 1

if date_arg:
    m = re.search(r"(\d{4})[-./](\d{2})[-./](\d{2})", date_arg)
    date_dot = f"{m.group(1)}.{m.group(2)}.{m.group(3)}" if m else now_mm.strftime("%Y.%m.%d")
else:
    date_dot = now_mm.strftime("%Y.%m.%d")

if slot_arg not in ("0800", "1200", "1400"):
    hhmm = now_mm.hour * 100 + now_mm.minute
    slot_arg = "0800" if hhmm < 1100 else ("1200" if hhmm < 1330 else "1400")

day_dir = Path(f"/root/GETC/Marketing/FB_Posts/{date_dot}")
md_file = day_dir / f"{slot_arg}-chat-ready.md"
p1_png = day_dir / f"{slot_arg}-post1.png"
p2_png = day_dir / f"{slot_arg}-post2.png"

print(f"RESOLVED: date_dot={date_dot} slot={slot_arg} force={force_flag}")

if md_file.exists() and not force_flag:
    print("CACHE_HIT_MD: True")
    if p1_png.exists() and p2_png.exists():
        print("CACHE_HIT_MEDIA: True")
    else:
        print("CACHE_HIT_MEDIA: False — rendering missing media now...")
        subprocess.run(["python3", "/root/GETC/Marketing/render_images.py", date_dot, slot_arg], check=True)
else:
    cmd = ["python3", "/root/GETC/Marketing/prepare_context.py", date_dot, slot_arg]
    if force_flag:
        cmd.append("--force")
    subprocess.run(cmd, check=True)
PY
```

- **If `CACHE_HIT_MD: True` (and `force=False`)**: Read `/root/GETC/Marketing/FB_Posts/${date_dot}/${slot_arg}-chat-ready.md`, output PART 1 verbatim along with the media file summary, and stop immediately.
- **Otherwise**: Read `/root/GETC/Marketing/.daily_brief_${date_dot}-${slot_arg}.json` and follow Steps 1–3 below.

---

## Step 1: Verify Grounding & Slot Directives from `.daily_brief_*.json`

1. **Primary & Backup Sources**:
   - Use `cleaned_content_excerpt` directly whenever `fetch_status == "READY"`.
   - If `fetch_status == "NEEDS_AI_FETCH"`, check `backup_source_url` or run at most ONE targeted `web_search` on official `.go.jp` / Tier 2 domains. Never cite generic root homepages.
2. **Morning Breaking News (`0800` Slot Only)**:
   - Only when `slot == "0800"` and `news_radar.mode == "ACTIVE_MORNING_CHECK"`, run ONE `web_search` for urgent Japan immigration, visa, scholarship, or labor changes within the past 24–72 hours. Include **Post 3** (`GETC - Hledan`) ONLY if a verified urgent update exists.
   - In `1200` and `1400`, do NOT search for breaking news; set `news_monitoring.status = "SKIPPED_NON_MORNING_SLOT"`.
3. **Post 2 Mode (`QUIZ` vs `LESSON` vs `STANDARD`)**:
   - Check `post_briefs[1].post_mode` in `.daily_brief_*.json`.
   - When `post_mode == "QUIZ"`, include `quiz_data` in Post 2 JSON (`jlpt_level`, `question_jp`, `question_mm`, `options: {"A": ..., "B": ..., "C": ..., "D": ...}`, `correct_option`, `explanation_mm`). Do NOT reveal the answer in the caption; reveal the answer + clear explanation in **Comment 1**.
4. **1-Day Advance Holiday Card (`1400` Slot)**:
   - If `holiday_card_required == true` in `.daily_brief_*.json`, generate the `"holiday_card"` object in the batch JSON using `holiday_brief` (`country`, `name_local`, `name_mm`, `date_range_display`, `holiday_type`, `greeting_title`, `greeting_body`, `caption`, `image_prompt`). Follow `tone_guidance` strictly (warm blessings for religious/New Year festivals; dignified respect for Martyrs' Day; joyful cultural note for commemorative holidays).
5. **Best-of-4 Reels & TikTok Script (`1200` Slot)**:
   - Compare `morning_0800_posts_for_video_selection` + today's `1200` Posts 1–2, pick the single most engaging topic, and choose `duration_mode`: `"30s"` (3 clips x 10s) for concise tips/vocab OR `"40s"` (4 clips x 10s) for step-by-step guides.

---

## Step 2: Write Burmese Copy & Visual Prompts (V4 Editorial Standards)

### R1. Audience, Voice & Anti-Translationese
- **Target Audience**: Myanmar youth aged 18–35 planning to study or work in Japan, plus parents/guardians.
- **Voice**: Warm, natural, approachable senior-to-junior advisory Burmese (`...ပါတယ်`, `...လေ့ရှိပါတယ်`, `...နိုင်ပါတယ်`, `...ကြည့်ရအောင်`). Avoid stiff textbook/newspaper translationese (`...ဖြစ်ပါသည်`, `...ဆိုသည်မှာ`, `...လျက် ရှိပါသည်`, `...သတ်မှတ်ခြင်း ခံထားရပါသည်`).

### R2. 5 Rotating Opening Hooks (Strict Rule)
- **NEVER** start Post 1 or Post 2 with `"ဂျပန်နိုင်ငံ..."` or `"ဂျပန်မှာ..."`!
- Follow each post's `assigned_hook_style` from `.daily_brief_*.json` (`QUESTION_HOOK`, `SCENE_IMMERSION_HOOK`, `COMMON_MISTAKE_HOOK`, `JAPANESE_KEYWORD_HOOK`, or `STAT_OR_FACT_HOOK`).

### R3. Natural Bilingualism & Japanese Script Format
- Keep common everyday English terms in English (`Settings`, `Deadline`, `Internship`, `Part-time`, `Interview`, `Online`, `CV`, `Orientation`, `Data Science`, `Silent Mode`, `Quiz`, `comment`). Never use awkward Burmese phonetic transliteration (`ဆက်တင်`, `အင်တာဗျူး`, `ကွန်မန့်`).
- Format Japanese terms cleanly as `Japanese (Romaji) - Burmese meaning` (e.g., `マナーモード (Manā Mōdo) - ဖုန်း Silent Mode ထားခြင်း`).

### R4. Length, Soft-Sell Cross-Promo CTA & Comments
- **Length**:
  - Standard Posts: **150–250 Burmese words** (`~550–900` Myanmar characters).
  - Japanese Lessons: **80–150 Burmese words** (`~280–520` Myanmar characters) + natural Japanese examples.
  - Quiz Posts: **60–130 Burmese words** inviting readers to guess A, B, C, or D in the comments.
- **Soft-Sell Cross-Promo CTA**: Write a natural, varied soft-sell invitation (do NOT use repetitive robotic templates!):
  - Post 1 (`GETC - Hledan`) softly mentions `GETC - Kamayut (https://www.facebook.com/gakushineikou2)`.
  - Post 2 (`GETC - Kamayut`) softly mentions `GETC - Hledan (https://www.facebook.com/gakushineikou)`.
  - Never include any external URLs inside captions or comments.
- **Comments (2–3 per post) & Natural Teaser**:
  - Include a natural, varied sentence in the caption inviting readers to check `comment` (NEVER use the word `"Insider"` and NEVER repeat the exact same teaser sentence across posts).
  - Write 2–3 genuinely helpful comments per post (practical tip, checklist, or polite discussion question). For Quiz posts, Comment 1 reveals the correct option (`A/B/C/D`) and explains why in Burmese.

### R5. Clean Scene-Only AI Image Prompts (Zero Text in Prompt)
- Write every `image_prompt` and `broll_prompt` as a **pure photographic/artistic scene description in English** (16–35 words, Japanese modern aesthetic, warm natural lighting, clean negative space for typography).
- **NEVER** include words like `"with text"`, `"bold title"`, `"Kanji"`, `"Burmese"`, `"logo"`, or quoted text inside `image_prompt` or `broll_prompt`—Python renders all Japanese/Burmese typography and the soft-glow GETC logo directly!

---

## Step 3: Save JSON, Validate, Render Images/Clips & Print Report

Write the batch JSON to `/root/GETC/Marketing/.getc_batch_${date_dot}-${slot_arg}.json` with this structure:

```json
{
  "date": "YYYY-MM-DD",
  "slot": "0800|1200|1400",
  "news_monitoring": {
    "checked_at": "YYYY-MM-DD HH:MM +06:30",
    "status": "POST_CREATED|NO_QUALIFYING_NEWS|SKIPPED_NON_MORNING_SLOT",
    "summary": "Brief factual note.",
    "search_coverage": ["Official source checked"],
    "sources_checked": [
      {
        "title": "Exact page title",
        "url": "https://...",
        "published_or_updated": "YYYY-MM-DD or Date not stated"
      }
    ]
  },
  "posts": [
    {
      "post_number": 1,
      "page": "GETC - Hledan (https://www.facebook.com/gakushineikou)",
      "category": "Exact category from brief",
      "post_mode": "STANDARD",
      "topic": "Short topic label",
      "topic_type": "STATIC",
      "core_tags": ["tag-1", "tag-2", "tag-3"],
      "source_updated_date": "YYYY-MM-DD or Date not stated",
      "title": "Concise English title",
      "angle": "Specific angle",
      "audience_need": "Target reader need",
      "verification_status": "VERIFIED",
      "caption": "Full Burmese caption (150-250 words) starting with assigned hook style, soft-sell Kamayut link, and 4-6 hashtags.",
      "image_prompt": "Pure English scene-only background prompt without any text or logo.",
      "comments": [
        "Comment 1: Practical tip in natural Burmese.",
        "Comment 2: Checklist or discussion question in natural Burmese."
      ],
      "sources": [
        {
          "title": "Exact source title",
          "url": "https://...",
          "published_or_updated": "YYYY-MM-DD or Date not stated",
          "claim_supported": "Fact verified"
        }
      ]
    },
    {
      "post_number": 2,
      "page": "GETC - Kamayut (https://www.facebook.com/gakushineikou2)",
      "category": "Exact category from brief",
      "post_mode": "STANDARD|LESSON|QUIZ",
      "topic": "Short topic label",
      "topic_type": "STATIC",
      "core_tags": ["tag-1", "tag-2", "tag-3"],
      "source_updated_date": "YYYY-MM-DD or Date not stated",
      "title": "Concise English title",
      "angle": "Specific angle",
      "audience_need": "Target reader need",
      "verification_status": "VERIFIED",
      "caption": "Full Burmese caption with soft-sell Hledan link and 4-6 hashtags.",
      "image_prompt": "Pure English scene-only background prompt without any text or logo.",
      "quiz_data": {
        "jlpt_level": "JLPT N5",
        "question_jp": "Japanese question sentence with blank （　　）",
        "question_mm": "Short Burmese prompt",
        "options": {"A": "...", "B": "...", "C": "...", "D": "..."},
        "correct_option": "B",
        "explanation_mm": "Clear explanation in Burmese"
      },
      "comments": [
        "Comment 1 (If QUIZ: reveal correct option + explanation_mm; otherwise practical example).",
        "Comment 2: Extra vocabulary or discussion prompt."
      ],
      "sources": [
        {
          "title": "Exact source title",
          "url": "https://...",
          "published_or_updated": "YYYY-MM-DD or Date not stated",
          "claim_supported": "Fact verified"
        }
      ]
    }
  ],
  "holiday_card": null,
  "video_script": null
}
```
*(Omit `quiz_data` unless `post_mode == "QUIZ"`. Include `holiday_card` object only when `holiday_card_required == true` in `1400`. Include `video_script` with `duration_mode` `"30s"` or `"40s"` and 3–4 `clips` only in `1200`.)*

Then run validation, media rendering, and cleanup in ONE command:

```bash
python3 /root/GETC/Marketing/generate_posts.py --input /root/GETC/Marketing/.getc_batch_${date_dot}-${slot_arg}.json ${FORCE_FLAG}   && python3 /root/GETC/Marketing/render_images.py ${date_dot} ${slot_arg} ${FORCE_FLAG}   && rm -f /root/GETC/Marketing/.getc_batch_${date_dot}-${slot_arg}.json /root/GETC/Marketing/.daily_brief_${date_dot}-${slot_arg}.json
```
*(Pass `--force` for `${FORCE_FLAG}` when `force=True`, otherwise leave empty.)*

Finally, read `/root/GETC/Marketing/FB_Posts/${date_dot}/${slot_arg}-chat-ready.md`, display **PART 1: COPY-READY CONTENT** verbatim, and append the **Execution & Diagnostic Report** from `render_images.py` right below it.
