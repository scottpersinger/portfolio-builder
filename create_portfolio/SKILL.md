---
name: create_portfolio
description: >-
  Build an engaging single-page personal portfolio website for someone from
  their LinkedIn profile and/or existing personal site. Gathers their
  student/role status, work experience, projects, and personal info via the
  claude-in-chrome browser tools — including a real image for every job (company
  logo) and every project (demo-video thumbnail or screenshot), with videos
  embedded click-to-play. Generates a self-contained, animated dark-theme HTML
  file in reverse-chronological order, saves it to a folder, and serves it
  locally for preview. Use when the user asks to "create a portfolio", "make a
  personal site", "build a portfolio from a LinkedIn profile", or points at a
  LinkedIn/personal URL and wants a portfolio.
---

# create_portfolio

Turn a person's LinkedIn profile (and existing personal site, if any) into an
engaging, single-file portfolio page — where **every job and project carries a
real image**, and any demo videos are embedded.

## When to use

The user gives you a person (usually via a LinkedIn URL, sometimes a résumé or
an existing personal site) and wants a portfolio / personal site generated for
them. Works for students, new grads, and early-career engineers especially.

## Inputs to collect from the user

- **Who** the portfolio is for and their LinkedIn URL (and any existing personal
  site / résumé / GitHub).
- **Section order preference** (default below).
- **Where** to save it and whether to deploy — but you can proceed with sensible
  defaults and confirm at the end.

Default section order (reverse-chronological within each section):

1. **Student / current status** — hero: name, headline, school + program, a
   short intro, a few credential "chips" (scholarships, YC, awards).
2. **Experience** — newest first, **each with a company logo**.
3. **Projects** — newest first, **each with an image** (demo-video thumbnail or
   screenshot) plus real links (GitHub / Devpost / demo); embed demo videos.
4. **Education** — degree + school + standing (computed from their grad year, see
   step 3.5), then the **3 most interesting classes** from their program's real
   course catalog. Especially valuable for students / new grads — the main audience.
5. **Personal** — location, languages, hobbies, service/volunteering, and
   contact/social buttons at the very bottom.

Non-negotiable: **every job and every project must have an image AND an
outbound link.** A card with a bare text block looks unfinished — this is the
main thing that makes the page feel engaging rather than a résumé dump. If a
company/org link isn't in the source, look up its official site (see step 3).

## Procedure

### 1. Load browser tools

LinkedIn blocks automated fetching (`WebFetch` will fail), but the user is
usually logged in, so drive their real browser session. Load the
claude-in-chrome tools in ONE `ToolSearch` call:

```
select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__javascript_tool
```

Call `tabs_context_mcp` once first to get a valid tab. Create your OWN tab with
`tabs_create_mcp` for asset-gathering — the user may be browsing in the existing
tabs, so don't hijack them.

### 2. Scrape the source(s)

- Navigate to the LinkedIn profile and use `get_page_text` to pull the whole
  card. **Note:** 1st-degree connections render the full profile (About,
  Experience, Education, Volunteering, Honors, Skills, posts); 2nd/3rd-degree
  profiles only expose the top summary card. If you get a thin result, tell the
  user and ask for a résumé or the person's own site.
- If they have a **personal site**, navigate it too. Use `get_page_text` for
  copy and `read_page` with `filter: "interactive"` to extract the real
  destination URLs behind project links (GitHub, Devpost, YouTube demos) —
  LinkedIn `lnkd.in` links are shortened and not directly usable.
- Prefer the person's own descriptions over LinkedIn's auto-text.

### 3. Assemble the data

Collect, per section:

- **Status:** name, headline, school + degree/program, one-paragraph intro,
  2–4 credential chips.
- **Experience:** org, role, dates (reverse-chron), one-line description, and an
  outbound link for **every** entry. Flag standouts (e.g. "First intern").
  Don't skip a link just because the source didn't include one — **look up the
  org's official site.** Try the obvious domain, verify it with a quick
  `curl -sL -o /dev/null -w '%{http_code}' <url>` (a 200, or a 301/302 redirect
  to the real site, is good), and fall back to a `WebSearch` for the official
  homepage if the guess misses. A `403`/`401` usually just means the site blocks
  bots — it still works in a browser, so keep the link. Only omit a link if no
  official site genuinely exists (then say so). Link the specific org the person
  worked at; if it rolls up under a parent (e.g. a hospital under a health
  authority), the parent's site is fine.
- **Projects:** name, year, one-to-two sentence description, and every real
  link (GitHub / Devpost / demo). Note awards inline as a badge.
- **Personal:** location, languages, hobbies, volunteering/service, contact
  email, socials (LinkedIn / X / GitHub), a short personal blurb (their own
  words from the LinkedIn "About"/personal site read best), and **their profile
  photo** for the About section (see step 4 for how to grab it).

**When socials/contact are thin — go looking, then ask.** Some profiles (often
students) expose little beyond LinkedIn: no email, no GitHub, no X, no personal
site. The Connect section needs more than one lone LinkedIn button, so:

1. **Search for their other feeds.** Run `WebSearch` for the person on the
   platforms they're likely on — at minimum **Instagram and TikTok**, plus X and
   GitHub — e.g. `"<full name>" <school/company> instagram`,
   `"<full name>" tiktok`, `<full name> github`. Match carefully: confirm it's
   the same person (same name **and** corroborating details like school,
   city, photo, or an interest you already know) before adding a link — don't
   attach a stranger's account. When in doubt, treat it as unconfirmed.
2. **If you still can't find (or confirm) anything, STOP and ask the user** for
   the links they want included (they often have handles you can't surface).
   Don't ship a Connect section with a single link, and don't invent or guess
   handles. Add whatever they give you and move on.

### 3.5. Standing & standout coursework (the Education section)

Most people using this skill are **students or new grads**, so the Education
section carries real weight. Build it in two moves:

- **Compute their standing** from their graduation year and *today's date* — don't
  just restate "Class of 2027." For a standard 4-year degree, grad year − 4 = start
  year; compare against the current date to get the year they're *entering* and
  whether it's their final year. E.g. Class of 2027 as of mid-2026 → finished 3rd
  year, **entering 4th (final) year**. Phrase it like
  `Class of 2027 · entering 4th (final) year · <city>`. If the program isn't the
  usual 4 years (accelerated, co-op, master's, transfer), adjust or keep it vague
  rather than guessing wrong.
- **Look up their program's real course catalog** and pick the **3 most
  interesting classes**. Find the official calendar/catalog (search
  `"<university>" "<program>" course catalog` or the department site), read the
  actual course codes, titles, and descriptions — **never invent course codes or
  titles.** Choose upper-year / advanced electives appropriate to their standing
  (a 4th-year gets 3000–4000-level courses, not intro ones), and prefer the ones
  that **echo their experience and projects** (e.g. surface a security course for
  someone who shipped auth work, an ML course for someone building AI products) so
  the page reads as one coherent story. Write each blurb as one engaging sentence
  distilled from the real course description.

Frame the cards honestly as *coursework they're drawn to / excited about* rather
than claiming completion — these are highlights from their program, not a
transcript. If you genuinely can't find the catalog (obscure/international school,
paywalled), tell the user and ask them for a few courses instead of fabricating.

For the `.edu-lead` school tile, reuse a real crest (`images/logo_<school>.png`)
if you grabbed one for Experience; otherwise a `.logo-tile.mono` with the school's
initial and brand color reads clean (a tiny/blurry favicon looks worse than a
tidy mono tile).

### 4. Gather images & videos (the part that makes it engaging)

Create an `images/` folder in the output dir and download every asset **locally**
(don't hotlink — hotlinked logos/CDN images break and the page must work
offline). Do all downloads with `curl -sL -o images/<name> "<url>"`.

**Project media — check for a demo video first.** Most hackathon/side projects
have one. Visit each project's Devpost or personal-site page and extract media
with `javascript_tool` (strip query strings first, or the tool blocks the result
as "Cookie/query string data"):

```js
const clean = u => (u||'').split('?')[0];
JSON.stringify({
  images:  [...new Set(Array.from(document.querySelectorAll('img')).map(i=>clean(i.src)).filter(s=>s.includes('software_photos')))].slice(0,6),
  iframes: [...new Set(Array.from(document.querySelectorAll('iframe')).map(f=>clean(f.src)).filter(Boolean))]
})
```

- A YouTube `iframe` (`.../embed/<ID>`) means there's a demo video → record the
  `<ID>` for click-to-play, and grab its thumbnail:
  `https://img.youtube.com/vi/<ID>/maxresdefault.jpg` (clean 1280×720).
- No video → download the first gallery/screenshot image as `images/<slug>.jpg`.
- No media at all → use the template's `.card.graphic` variant (an SVG on a
  gradient) so the card still has visual weight.

**Job logos.** Clearbit's logo API is dead. Use Google's favicon service, which
reliably returns a square logo: `https://www.google.com/s2/favicons?domain=<domain>&sz=256`.
Save as `images/logo_<slug>.png`. Find each company's domain from its LinkedIn
link or a quick guess (`copperlane.ai`, `uwblueprint.org`, …). If a favicon
comes back tiny (16px) or generic, try the real domain or note it to the user.
**Watch for white logos:** a white/transparent logo vanishes on the white tile —
add `style="background:#101827"` to that one `.logo-tile`.

**Profile photo (for the About section).** This one's fiddly because the
`javascript_tool` filters out cookies, query strings, AND base64, and LinkedIn's
image URLs are signed + cross-origin (so `curl` on a clean URL and `canvas`
export both fail). The reliable path, on the LinkedIn tab:

1. Find the main top-card photo — the largest `<img>` whose `src` matches
   `/profile-displayphoto/` (there are ~12 on the page; sort by displayed width
   and take the widest, and confirm it sits at the top-left of the card).
2. `fetch(img.currentSrc, {cache:'force-cache'})` inside the page (it's cached,
   so it succeeds), get a `blob`, and `createImageBitmap(blob)` — because it came
   through a blob the canvas is **not** tainted.
3. Draw a centered square cover-crop to a 256×256 canvas and `toDataURL('image/jpeg',0.88)`.
4. You can't return the base64 (filtered), so **trigger a browser download** of
   it instead: create an `<a download="<name>.jpg" href="<dataurl>">`, `.click()`
   it — it lands in `~/Downloads`. Then `cp` it into `images/`.

Display it as a circular avatar in the template's `.about-lead` block. If this
whole dance fails, fall back to a monogram tile and tell the user.

### 5. Generate the page

Copy `assets/template.html` (next to this skill) and fill in the placeholders,
repeating the marked `.job` / `.card` / `.course` blocks per item (the template
now includes the Education section between Projects and About). Keep it a **single
self-contained HTML file** — inline CSS + JS, no external dependencies besides
the local `images/`. The template already provides the engaging layer: animated
gradient blobs, sticky glass nav, scroll-reveal, logo tiles, video cards with
click-to-play, and hover animations. Preserve reverse-chronological ordering and
the section order above. Re-theme by swapping the `--a1/--a2/--a3` accent trio.

### 6. Save it

Save to a real folder the user can keep — **not** the session scratchpad, which
is temporary. Default to `~/<firstname>-portfolio/` (with `index.html` +
`images/`) unless the user specifies a location. Confirm the path.

### 7. Serve and preview

`file://` URLs are blocked by the browser extension, so serve over HTTP:

```
cd ~/<firstname>-portfolio && python3 -m http.server 8900
```

Run it in the background, then `navigate` to `http://localhost:8900/` and
`screenshot` while scrolling to verify every section. Because of the scroll-
reveal animations, cards fade in as they enter the viewport — scroll a little
past a section and pause before screenshotting so it's fully revealed.
**Specifically check that every logo is visible** (catch white-on-white tiles)
and that each project image loaded. Give the user the local URL and the restart
command.

### 8. Offer next steps

- Tweaks: headshot, accent color (swap the `--a1/--a2/--a3` trio), wording,
  reordering.
- Permanent deploy: push to a GitHub repo + Vercel/Railway for a public URL, or
  point an existing domain at it. **Push the `images/` folder too** — the page
  references local assets, so it renders blank without them.

## Judgment calls to surface (don't silently guess)

- **Email:** prefer the address the person publishes on their **own site** over
  the LinkedIn one if they differ — but mention which you used.
- **Tone:** default to a recruiter-facing page. If the source has jokey "skills"
  or very personal lines, leave them out but tell the user they're easy to add
  back.
- **Thin LinkedIn data:** if only the summary card loaded, say so rather than
  shipping a sparse page.
- **Generic/low-res logos:** if a company favicon is tiny or a placeholder,
  flag it and offer to swap in a real logo the user provides.
