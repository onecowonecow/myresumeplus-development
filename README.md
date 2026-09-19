# MyResumePlus

A hiring platform for asynchronous video interviews. A company writes a question set, sends candidates a link, and reviews the recorded answers on its own schedule. Each client organization runs on its own subdomain.

This repo is a writeup, not the source. The application lives in a private repo. The architecture, the permission model, and the product decisions described here are mine; most of the implementation was delegated to Claude Code and Codex.

---

## The vision

Two sentences had to stay true through every phase.

A hiring team should be able to see and hear a candidate without putting a meeting on anyone's calendar. Scheduling is the expensive part of a first round screen, and it costs the candidate more than it costs the company. A student takes a bus somewhere and loses an afternoon to a conversation that lasts twenty minutes.

An organization's hiring data should never be reachable from another organization. Resumes, recordings, and interviewer notes are some of the most sensitive material a company holds about people who do not work there yet.

The phases below are ordered by which of those two sentences was under the most strain at the time. Each one says what I built, and what I decided when the two sentences pulled against each other.

---

## Phase 1: the core loop

Before anything else, one candidate had to record an answer and one admin had to watch it. Everything in this document after Phase 1 is decoration if that loop is broken.

Candidates get in through an emailed one time code. No password, no account to create, no profile to maintain after the interview ends. A candidate applying to six companies should not end up with six logins.

That choice looks like a convenience feature. It is really a data decision. An account implies a durable record of a person across companies, and I did not want to hold one. The candidate exists inside the organization that invited them and nowhere else.

---

## Phase 2: questions

The platform generates interview questions from a candidate's resume, and keeps a bank of more than fifty generic questions as a fallback for when the model is rate limited or a resume fails to parse.

The fallback is the part worth explaining. An interview that cannot start because an API is busy is worse than an interview with slightly less tailored questions, so the generic bank is always reachable and the generated set is treated as an upgrade. Generated questions also get deduplicated, because a model asked for eight questions about one resume will happily produce the same question three ways.

The model writes questions. It does not score anyone, rank anyone, or recommend anyone. That line was drawn at the start and has not moved. Automated scoring of a recorded human is a different product with different obligations, and I did not want to build it by accident, one convenience feature at a time. The hiring team reads the resume, watches the answers, and decides.

---

## Phase 3: video

Video is where the honest engineering constraints live.

A five question interview recorded at whatever bitrate a modern phone browser picks is enormous, and every megabyte lands on a candidate's upload connection. A bitrate limiter set at 1.4 Mbps holds each response under a 20MB ceiling, with most answers landing near 17MB. Capture runs through WebRTC and MediaRecorder in the browser, so nothing gets installed.

Uploads are chunked to Cloudflare R2 and deferred to a single screen at the end of the interview rather than fired off after each answer. That ordering is deliberate. A candidate on hotel wifi who loses a connection halfway through question three should not lose question three. The recording stays local until the interview is done, then goes up in one controlled pass with the failure surface in one place the candidate can see.

There is a fairness argument underneath the compression number too. If quality floats with connection speed, then the candidate on a worse connection submits a worse looking interview, and a reviewer reads that as something about the person. Holding everyone to the same ceiling removes a signal that should never have been in the room.

Reviewers can pull individual responses or download every answer at once, including audio only versions for anyone who would rather listen than watch.

---

## Phase 4: permissions

Most role systems make you invent a new role every time one person needs one extra capability. Teams of fifteen end up with twelve roles that differ by one checkbox each, and nobody can say what any of them do.

So permissions here work the way Discord's do. An owner defines one default permission set, with each capability set to allow, deny, or default. Every admin inherits that set. When an individual admin needs something different, the owner overrides that single capability on that single admin, and the override resolves above the role.

Three states instead of two is what makes it work. A capability left on default follows the org set and keeps following it when the org set changes later, while an explicit allow or deny stays put. The common case stays one role, and the exception stays an exception rather than becoming a thirteenth role.

Drawing this was harder than building it. The flowchart for the admin hierarchy is the one that took the most revisions, because every arrow you draw forces you to answer what happens when two rules disagree, and you cannot leave that ambiguous on a diagram the way you can leave it ambiguous in your head.

---

## Phase 5: multi-tenancy

This is the phase that exists entirely to serve the second sentence of the vision.

The platform is single instance, multi-tenant. One deployment and one Neon Postgres database serve every client, and each organization reaches it through its own wildcard subdomain, so a company gets something like `company.myresumeplus.com` provisioned from a central owner dashboard without anyone standing up infrastructure by hand. Six tables hold users, candidates, question sets, interview responses, and the one time code state, and every query is scoped to the acting tenant through Drizzle.

Single instance is the cheaper and more maintainable choice. It also means a scoping mistake is not a bug, it is one company reading another company's candidates. I took that trade knowingly, and the consequence is that tenant scoping belongs in the query layer rather than at each call site. Anything that depends on every future developer remembering to add a filter is a design that has already failed.

The subdomain is doing quiet work on the candidate side as well. A candidate clicking into an interview sees the hiring company's name in the address bar rather than a vendor's. The infrastructure is shared; the experience should not advertise that.

---

## Phase 6: billing

Stripe subscriptions gate how many candidates an organization can run and how many question sets it can keep. Plans start at $9 a month.

Billing is wired into the same permission system rather than living beside it. A plan tier is a statement about what an account is allowed to do, which is the definition of a permission, and splitting that across two systems produces the failure everyone has seen: an account that downgrades and keeps its old capabilities because the second system never heard about it. One place decides what you can do, and subscription state is one of its inputs.

---

## Phase 7: hardening and documentation

The one time code flow is rate limited to one code per email address every twenty seconds, and the limit is keyed to the address rather than to a cookie. Clearing storage, opening a private window, or switching devices does not reset it, which is the entire point. Sessions run on JWTs, input is validated with Zod at the boundary, and every query goes through Drizzle as a parameterized statement.

Documentation ran alongside all of this rather than after it. There are twenty pages at [myresumeplus.com/docs](https://myresumeplus.com/docs), plus a nine page flowchart set covering candidate registration and login, admin registration and login, the admin portal, interview creation, and the admin hierarchy, each with a worked example.

I write the flowcharts before the implementation on anything with branching state. Prose lets you describe a permission system you have not finished thinking about. A diagram does not, because an unlabeled arrow is visibly an unanswered question.

---

## Stack

TypeScript throughout. React with Vite, Tailwind, shadcn/ui, and Wouter on the front end; WebRTC and MediaRecorder for capture. tRPC between client and server, Express behind it, Drizzle ORM over Neon Postgres. Cloudflare R2 for recordings. Stripe for subscriptions, Resend for one time code delivery, Zod for validation, Vitest for tests. Diagrams in Mermaid.js and Figma. Built by directing Claude Code and Codex.

---

## Status

An organization signs up, gets its subdomain, builds a question set, imports candidates a hundred at a time from a spreadsheet, and reviews recorded answers. Permissions, billing, and the video pipeline are live. The docs site is up. 
