# AGENTS.md

Handoff notes for anyone (human or agent) working on this repo.

## What it is

DistroPicker: a quiz that recommends a Linux distro, plus Explore (distro cards), a Wiki of terms, an install guide and a community leaderboard with comments.

- Live at **https://distropicker.com** (`CNAME`).
- Hosted on GitHub Pages, built from `main` at `/` (no Actions workflow). A merge to `main` is a deploy.
- Public repo, MIT license.

## Files

| File | What |
|---|---|
| `index.html` | The whole site: CSS, HTML and all JS inline. |
| `bgm.mp3` | Background music (mute button top of page). |
| `favicon.png`, `preview.png` | Icon and social preview image. |
| `CNAME`, `robots.txt`, `sitemap.xml` | Pages domain and SEO. |
| `.mcp.json` | Supabase MCP server for this project (`wyhsdixvfnemobdvwiai`). |

## Inside `index.html`

The main `<script>` near the bottom holds, in order:

- `wikiTerms` — Wiki entries and tooltip text.
- `quizQuestions` — 9 questions, `id` 1–9. Answers are stored as `answers[id] = value`.
- `distros` — the distro data (27 entries). Each has `base`, `immutable`, `release`, `desktop`, `difficulty`, `useCase`, `gpuSupport`, `laptopFriendly`, `ramRecommended`, `bootloader`, `brandColor`, etc.
- `supabaseClient` — Supabase client (publishable key, meant to be public).
- Startup in the `DOMContentLoaded` handler.
- Quiz: `renderQuestion`, `selectOption`, `showResults`, `generateExplanation`.
- **Scoring:** `calculateRecommendation()` adds points per distro for each answer (experience, device, GPU, RAM, immutable, base, desktop, release, use case) and picks the highest. On a tie the **later** distro in the array wins (`reduce` with `>`). `calculateAlternatives()` takes the first 3 distros with the same base or difficulty.
- Leaderboard (`loadVotes`, `voteForDistro`, `renderLeaderboard`): one vote per user, changeable every 6 months.
- Distro modal with comments, auth, profile, passkeys, avatar upload.

## Online features (Supabase)

- Tables used: `votes`, `comments`, `profiles`; storage bucket `avatars`.
- Sign-in is username + password, or passkey (`signInWithPasskey`). Login looks up the profile by username, then signs in with `<user id>@distropicker.local`. Sign-up sends a random `@distropicker.local` email. Whatever maps that to the user id lives in Supabase, not in this repo.
- The database schema and security rules (RLS) also live only in Supabase. Use the MCP server in `.mcp.json` to look.

**Offline behaviour (PR #13):** if the Supabase script is blocked or fails, `supabaseClient` is `null`. The quiz, Explore, Wiki, navigation and tooltips render first; `initAuth` and `loadVotes` run afterwards inside their own `try` and skip Supabase when there's no client. The distro modal still opens without comments. Sign-in, vote and comment buttons still need Supabase.

## Testing locally

- Quiz, Explore and Wiki: just open `index.html` in a browser.
- With a server (needed for passkeys): `python3 -m http.server` (or `npx serve`), then http://localhost:8000.
- Syntax check: pull the inline script out and run `node --check` on it.
- Check scoring without a browser (prints distro count and the pick):

```bash
node -e '
const h=require("fs").readFileSync("index.html","utf8");
const js=[...h.matchAll(/<script>([\s\S]*?)<\/script>/g)].pop()[1];
const data=js.slice(js.indexOf("const wikiTerms"),js.indexOf("// Supabase"));
const fn=js.slice(js.indexOf("function calculateRecommendation"),js.indexOf("function calculateAlternatives"));
const answers={1:"beginner",2:"laptop",3:"nvidia",4:"16gb",5:"no",6:"any",7:"any",8:"any",9:"gaming"};
console.log(new Function("answers",data+fn+"return [distros.length, calculateRecommendation().name]")(answers));'
```

Change `answers` (values come from `quizQuestions` options). This relies on the `// Supabase` comment right after the data.

## Rules

- Keep it one static `index.html`: no build step, no framework, no npm deps. Libraries load from a CDN with a pinned version (supabase-js is pinned).
- Escape anything user-written (usernames, comments) with `escapeHtml` before putting it in HTML.
- Clickable `div`s need `role="button"` and `tabindex="0"`; one global keydown handler turns Enter/Space into a click (PR #11). Escape closes all modals.
- Nothing that runs before the Supabase `try` in startup may call Supabase, or a blocked CDN blanks the page again.

## Gotchas from past PRs

- **#12 scoring bugs:** compare the **user's** answer to the distro field (it once checked `distro.release === 'semi'` alone, so every semi-rolling distro got +15). Treat `'any'` on either side as a match. Check `gpuSupport` is right: wrong NVIDIA flags give −40. The progress bar counts from question 1, not 0.
- When adding a distro, fill every field the scoring reads, or it silently scores low.
- `README.md` still says "Google sign-in"; auth is now username/password and passkeys.
- Console logs still say "DistroMatch" (old name).
