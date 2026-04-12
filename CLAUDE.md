# Project: Van Huynh's Personal Blog

Hugo + PaperMod blog at congvan.github.io. GitHub username: CongVan. SSH alias: github.com.me.

## Blog Content Rules

### Privacy & Confidentiality
- **NEVER** use real company names from previous employers (e.g., no "Persona Studios", "Antler VC", "Galaxy Education", "Prime Commerce", "FPT Telecom")
- **NEVER** copy production code from the octopus/persona codebases into blog posts
- **NEVER** reference internal file paths, repo names (octopus-be, octopus-ai), or internal tooling names
- Use generic descriptions: "a VC firm", "an enterprise client", "the platform I built", "a SaaS company"
- All code in blog posts must be **original mock code** that illustrates the pattern, not copied from production
- OK to keep personal metrics: 250K+ interactions, 20+ clients, team of 7 — these are career achievements
- OK to describe **architectural patterns** (Agenda Engine, state machine, gRPC streaming, event-driven) — concepts aren't proprietary
- OK to reference tech stack generically: Node.js, Python, PostgreSQL, Redis, Firebase, gRPC, OpenAI

### Writing Style — Sound Like a Human, Not an AI
- **Simple words.** Write like you're explaining to a colleague at a whiteboard, not writing a textbook. If a 5th-grader can't understand the sentence structure, simplify it.
- **Short sentences.** One idea per sentence. Break long sentences into two.
- **Straightforward.** Say what you mean. No hedging ("it's worth noting that..."), no filler ("in order to", "it should be noted"), no throat-clearing.
- **Goal-focused.** Every section answers one question. If a paragraph doesn't serve the topic's goal, cut it.
- **First person, practitioner voice.** "I built this" not "the system was designed." Share what you did, what broke, what you learned.
- **No "you" / "your" addressing the reader.** These posts share experience, not step-by-step instructions. Use "I" for personal decisions ("I built", "I ran into"), "we" for team decisions, and third person for the system ("the backend", "the AI service"). Change "your backend" → "the backend", "you can configure" → "I configure" or "the system is configured". Exception: "you" / "your" inside **quoted dialogue** (AI prompts, user messages, conversation examples) is fine — that's not addressing the reader.
- **No AI-sounding phrases.** Avoid: "it's important to note", "leveraging", "in today's landscape", "comprehensive", "robust", "utilize", "facilitate", "delve into", "it's worth mentioning."
- **Show, don't tell.** Code and diagrams over explanation. A 10-line code block beats 3 paragraphs of description.
- **Code-heavy** but using mock/illustrative examples — never production code.
- Frame as "a platform I built" or "in my experience..."

### Post Dates & Ordering
- **Always use the real current date** when writing or updating a post. Run `date +"%Y-%m-%dT%H:%M:%S%z"` to get it — never fabricate or reuse a date from the series plan.
- **`date`** frontmatter = the date the post was originally written (don't change on edits)
- **`lastmod`** frontmatter = the date of the most recent meaningful update. Update this every time you edit the post's content (typos don't count; content changes do).
- **Post lists are ordered by `lastmod` descending** via `hugo.toml` config (`frontmatter.lastmod` + `[taxonomies]` settings). The most recently updated post appears first on /posts/ and the homepage.
- When adding `lastmod` to an old post for the first time, set it to the date you're making the edit.

### Diagram Style
- SVG with dark/light mode via CSS `prefers-color-scheme`
- **Simple colors only:** use 2 tones — primary text + muted. No colored backgrounds, no rainbow boxes.
- Monochrome: `#dadadb` / `#888` (dark), `#1a1a1a` / `#999` (light)
- **Transparent background** — `.bg { fill: none; }` so SVGs blend with the page
- Boxes: no fill, just border strokes
- Font: Fira Mono, consistent with site

### Architecture Context (for accurate mock code)
- **Backend**: Node.js/TypeScript, Express, PostgreSQL (Prisma), Redis, BullMQ, Firebase Firestore
- **AI Service**: Python, gRPC streaming, OpenAI API with tool calling, Langfuse tracing
- **Communication**: Backend sends ChatPayload via gRPC → AI service streams EventResponse back
- **Agenda Engine**: State machine with item types (Convey, Gather, Debate, Q&A, Cart, API, Schedule, Select, Quiz, Contact)
- **Each agenda item**: Python class with lifecycle hooks (on_start, on_user_response, on_tool_call, on_end, should_end)
- **Real-time**: Redis Streams for chat chunks, Redis Hashes for conversation state, Postgres as the durable store (flushed on agenda completion). Frontend consumer (SSE reading from Redis) is a separate future post.
- **Channels**: Webchat, Widget, WhatsApp, Messenger — unified via adapter pattern

### Blog Series: "Building a Conversational AI Platform"
- 12 posts, 1 per day
- See `content/posts/_series-plan.md` for the full schedule
- Each post links back to Day 1 (flagship)
- Tags: AI Research, Backend, Frontend, DevOps

### Blog Workflow — Always Use the `claude-blog` Plugin Skills
When writing or updating any blog post, use the full `claude-blog` plugin pipeline instead of just writing markdown. Pick the skills that match the job. Never ship a post that wasn't touched by at least `blog-write` (or `blog-rewrite`) **plus** `blog-seo-check` **plus** `blog-analyze`.

**For a new post** (`/blog write <topic>`):
1. `claude-blog:blog-strategy` — if the post is part of a new series or cluster, confirm positioning first
2. `claude-blog:blog-outline` — SERP-informed outline with section word counts and gap analysis
3. `claude-blog:blog-write` — write the full draft using the chosen template
4. `claude-blog:blog-chart` — generate any inline SVG charts for chart-worthy data (already fits our monochrome diagram style)
5. `claude-blog:blog-image` — generate hero/section images via Gemini when available
6. `claude-blog:blog-schema` — add JSON-LD `BlogPosting` / `Person` / `FAQPage` schema
7. `claude-blog:blog-seo-check` — post-writing SEO validation (title, meta, headings, links, OG tags)
8. `claude-blog:blog-factcheck` — verify every statistic against its cited source
9. `claude-blog:blog-geo` — AI citation readiness audit (passage citability, entity clarity)
10. `claude-blog:blog-analyze` — final 100-point quality score. Must hit ≥80 before publishing.

**For updating an existing post** (`/blog rewrite <file>` or content edits):
1. `claude-blog:blog-rewrite` — optimize the existing post (adds answer-first formatting, sourced stats, anti-AI-pattern detection)
2. `claude-blog:blog-seo-check` — re-validate SEO
3. `claude-blog:blog-factcheck` — re-verify stats
4. `claude-blog:blog-analyze` — re-score. Score must not drop.
5. Bump the `lastmod` frontmatter to today's date (run `date +"%Y-%m-%dT%H:%M:%S%z"`)

**For site-wide maintenance**:
- `claude-blog:blog-audit` — periodically (monthly) run a full-site health assessment
- `claude-blog:blog-cannibalization` — detect keyword overlap across posts
- `claude-blog:blog-calendar` — refresh the editorial calendar

**For distribution**:
- `claude-blog:blog-repurpose` — adapt the post for Twitter/X threads, LinkedIn articles, YouTube scripts when publishing
- `claude-blog:blog-audio` — optional audio narration for longer posts
- `claude-blog:blog-google` — track post performance via GSC/GA4 post-publication

**Exceptions that don't need the full pipeline:**
- Frontmatter-only changes (e.g. fixing `lastmod`, adding tags)
- Pure diagram or layout fixes where post content is unchanged
- Typo fixes (still bump `lastmod` if changed)

Even for these small edits, run `claude-blog:blog-analyze` on the final output as a sanity check if the changes touch more than one paragraph.

## Technical

### Commands
- `hugo server -D` — local dev with drafts
- `hugo --gc --minify` — production build
- `hugo new posts/my-post.md` — new post from archetype

### Deployment
- GitHub Actions auto-deploys on push to `main`
- Workflow: `.github/workflows/deploy.yml`
- Branch `enhance-git-profiles` for blog series drafting

### Custom Layouts
- `layouts/index.html` — custom portfolio homepage
- `layouts/posts/list.html` — posts page with tag filter chips (URL ?tag= params)
- `layouts/partials/home_info.html` — hero with avatar
- `layouts/partials/header.html` — search as icon in nav
- `layouts/partials/extend_head.html` — all custom CSS (Fira Mono font, responsive styles)
- `layouts/partials/extend_footer.html` — contact section in footer
