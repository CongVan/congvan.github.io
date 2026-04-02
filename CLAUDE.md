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

### Writing Style
- Code-heavy but using mock/illustrative examples
- Include SVG diagrams for architecture flows (dark/light mode compatible using CSS prefers-color-scheme)
- Frame as "a platform I built" or "in my experience building..."
- First person, practitioner voice — not academic or tutorial-style
- Short paragraphs, direct sentences

### Architecture Context (for accurate mock code)
- **Backend**: Node.js/TypeScript, Express, PostgreSQL (Prisma), Redis, BullMQ, Firebase Firestore
- **AI Service**: Python, gRPC streaming, OpenAI API with tool calling, Langfuse tracing
- **Communication**: Backend sends ChatPayload via gRPC → AI service streams EventResponse back
- **Agenda Engine**: State machine with item types (Convey, Gather, Debate, Q&A, Cart, API, Schedule, Select, Quiz, Contact)
- **Each agenda item**: Python class with lifecycle hooks (on_start, on_user_response, on_tool_call, on_end, should_end)
- **Real-time**: Firebase Firestore for state sync (NOT Socket.io), API streaming for chat responses
- **Channels**: Webchat, Widget, WhatsApp, Messenger — unified via adapter pattern

### Blog Series: "Building a Conversational AI Platform"
- 12 posts, 1 per day
- See `content/posts/_series-plan.md` for the full schedule
- Each post links back to Day 1 (flagship)
- Tags: AI Research, Backend, Frontend, DevOps

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
