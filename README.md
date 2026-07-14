# From Prompt to Playlist: The Engineering Behind AI Playlist

**TL;DR** Company XY shipped a feature that turns "songs for a rainy Sunday morning that slowly wake me up" into a personalised playlist in under two seconds, for millions of listeners. The surprising decision at its core: the LLM is never allowed to name a song. This repo holds the full engineering story from my interviews with the team, the plan I'd use to put it in front of engineers worldwide, the same story reshaped per platform, and the dashboard that tells leadership whether it worked.

**Author:** [Gift Ojeabulu] · assessment

---

## Table of Contents

- [Why this repo exists](#why-this-repo-exists)
- [Task 1 · The Article](#task-1--the-article)
  - [The problem, stated honestly](#the-problem-stated-honestly)
  - [The platform at a glance](#the-platform-at-a-glance)
  - [Architecture: a pipeline, not a model](#architecture-a-pipeline-not-a-model)
  - [Latency: the two-second religion](#latency-the-two-second-religion)
  - [Reliability: degrade, never fail](#reliability-degrade-never-fail)
  - [Experimentation: measuring a vibe](#experimentation-measuring-a-vibe)
  - [The trade-offs ledger](#the-trade-offs-ledger)
  - [What you can take away](#what-you-can-take-away)
- [Task 2 · The Amplification Plan](#task-2--the-amplification-plan)
- [Task 3 · Content Repurposing](#task-3--content-repurposing)
- [Task 4 · The Six-Week Dashboard](#task-4--the-six-week-dashboard)
- [Assumptions, and why I'm comfortable with them](#assumptions-and-why-im-comfortable-with-them)

---

## Why this repo exists

This is my submission for the Engineering Relations Specialist assessment. The brief asked for four things: the engineering story behind Company XY's new AI Playlist feature, a plan to get it in front of engineers worldwide, the same story reshaped for different platforms, and a dashboard that tells leadership whether any of it worked.

I've written all of it the way I'd write for a real engineering audience, because the assessment deserves the same standard as the job. The engineers quoted are composites I've named for readability, since the brief asks me to assume the interviews happened; every technical claim they make is one I'd defend. Assumptions are flagged inline and collected [at the bottom](#assumptions-and-why-im-comfortable-with-them) with my reasoning, because engineers will check. They should.

All diagrams are plain text. They render identically on GitHub, in a terminal, and in a code review diff, and they'll still work in five years. Nothing to render, nothing to rot.

---

## Task 1 · The Article

*How Company XY turned free-form language into personalised playlists for millions of listeners, in under two seconds.*

When Company XY launched AI Playlist, the pitch fit in one sentence. Type "songs for a rainy Sunday morning that slowly wake me up" and get a playlist that feels hand-made for you. Behind that single text box sits some of the most disciplined systems engineering the company has shipped in years. I spent time with the engineers who built it to understand how, and more importantly why, it works the way it does.

### The problem, stated honestly

The team's founding insight was that this is not a chatbot feature. Nobody wants a conversation with their music app. They want thirty great songs, quickly, and they want the playlist to know them.

"An LLM can describe a great playlist. It cannot retrieve one," Tunde Adeyemi, the staff engineer who led the intent-parsing work, told me. Ask a general-purpose model for songs and you get hallucinated tracks, misattributed artists, and no awareness of your listening history. The existing recommender has the opposite problem. It knows your taste intimately and understands nothing about "music my Nigerian dad would approve of at a barbecue."

So the real engineering problem was a marriage: language understanding from LLMs, retrieval and personalisation from recommendation systems the team has run in production for years, joined under a latency budget the LLM call alone could blow.

### The platform at a glance

Before we walk the pipeline stage by stage, here's the full service topology a request travels through, from any client to a finished playlist.

```
                                          AI Playlist Platform

┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                 Client Applications                                       │
│                           Mobile App • Desktop • Web Player                               │
└───────────────────────────────────────────┬──────────────────────────────────────────────┘
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │   API Gateway   │
                                   │ Auth • Routing  │
                                   └────────┬────────┘
                                            │
                                            ▼
                            ┌────────────────────────────────┐
                            │ Prompt Validation & Safety     │
                            │ • Rate limiting                │
                            │ • Prompt sanitization          │
                            │ • Content safety               │
                            └───────────────┬────────────────┘
                                            │
                                            ▼
                             ┌──────────────────────────────┐
                             │     LLM Intent Parser        │
                             │ Converts prompts into:       │
                             │ • Mood                       │
                             │ • Genre                      │
                             │ • Energy                     │
                             │ • Activity                   │
                             │ • Language                   │
                             └───────────────┬──────────────┘
                                             │
                                    Structured Intent
                                             │
                 ┌───────────────────────────┼────────────────────────────┐
                 ▼                           ▼                            ▼
      ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
      │ User Profile API │        │ Embedding Store  │        │ Music Metadata   │
      │ Listening History│        │ Vector Features  │        │ Catalog Service  │
      └────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
               │                           │                           │
               └───────────────────────────┼───────────────────────────┘
                                           │
                                           ▼
                             ┌──────────────────────────────┐
                             │ Candidate Generation Service │
                             └───────────────┬──────────────┘
                                             │
                                             ▼
                             ┌──────────────────────────────┐
                             │ Recommendation & Ranking     │
                             │ • Collaborative Filtering    │
                             │ • Similarity Models          │
                             │ • Personalization            │
                             └───────────────┬──────────────┘
                                             │
                                             ▼
                             ┌──────────────────────────────┐
                             │ Business Rules Engine        │
                             │ • Explicit content           │
                             │ • Regional licensing         │
                             │ • Diversity controls         │
                             └───────────────┬──────────────┘
                                             │
                                             ▼
                             ┌──────────────────────────────┐
                             │ Playlist Assembly Service    │
                             └───────────────┬──────────────┘
                                             │
                                             ▼
                             ┌──────────────────────────────┐
                             │ Monitoring & Analytics       │
                             │ Logs • Metrics • Traces      │
                             └───────────────┬──────────────┘
                                             │
                                             ▼
                                   Personalized Playlist
```

Notice what sits around the interesting parts: safety checks before the LLM ever sees a prompt, a business rules engine after ranking, monitoring wrapped around everything. The glamorous components live inside a very unglamorous, very deliberate frame.

### Architecture: a pipeline, not a model

Conceptually, the topology above collapses into a four-stage pipeline governed by one principle. Use the LLM only where genuine language understanding is required, and lean on cheap, proven systems everywhere else.

```
  User prompt (free text)
          │
          ▼
  ┌───────────────────────────────┐
  │ STAGE 1 · Intent Parsing      │──── cache hit? skip straight to Stage 2
  │                               │
  │ Distilled LLM emits a         │
  │ constrained JSON intent:      │
  │ mood • genre • tempo • era    │
  │ activity • language •         │
  │ negative constraints          │
  │                               │
  │ Never track names.            │
  └──────────────┬────────────────┘
                 │ structured intent
                 ▼
  ┌───────────────────────────────┐
  │ STAGE 2 · Candidate Retrieval │
  │                               │
  │ Vector ANN search over        │
  │ precomputed track embeddings  │
  │ + hard filters: tempo, era,   │
  │ language, region              │
  │                               │
  │ ~100M tracks ─▶ ~5K           │
  │ candidates, tens of ms        │
  └──────────────┬────────────────┘
                 │ candidates
                 ▼
  ┌───────────────────────────────┐
  │ STAGE 3 · Personalised        │
  │ Ranking                       │
  │                               │
  │ Existing rankers rescore      │
  │ against the user taste vector │
  │                               │
  │ Tunable blend:                │
  │ prompt intent ◀─────▶ taste   │
  └──────────────┬────────────────┘
                 │ ranked tracks
                 ▼
  ┌───────────────────────────────┐
  │ STAGE 4 · Assembly &          │
  │ Guardrails                    │
  │                               │
  │ Sequencing (energy arcs)      │
  │ artist spacing • dedupe       │
  │ licensing • safety filtering  │
  └──────────────┬────────────────┘
                 │
                 ▼
  Playlist, streamed track by track
```

**Stage one: prompt understanding.** An LLM translates the free-form prompt into a structured intent object: genres, moods, tempo range, eras, activities, language, artist references, and negative constraints such as "no sad songs." Here's the decision I keep coming back to: the model never names tracks. It emits a constrained JSON schema, and that single choice eliminates hallucination as a failure class. The LLM may only describe what kind of music, never which music.

**Stage two: candidate retrieval.** The intent object is embedded and used to query a vector index over the full catalogue, drawing on the embedding store, the metadata catalogue, and the user profile service shown in the topology above. (I'm assuming an approximate nearest neighbour index such as HNSW over precomputed track embeddings, the standard approach at this scale.) Structured fields become hard filters; the semantic embedding handles everything fuzzy. This stage narrows roughly one hundred million tracks to a few thousand candidates in tens of milliseconds.

**Stage three: personalised ranking.** Candidates are rescored by the existing collaborative filtering and similarity models against the user's taste profile. The defining decision here is a tunable blend between prompt intent and personal taste. Weight taste too heavily and every prompt returns your usual rotation. Weight the prompt too heavily and the playlist feels generic, identical for every user. "We argued about that blend for weeks," Priya Raman, the senior ML engineer who owns the ranking layer, told me. "In the end we stopped arguing and made it a parameter. The A/B tests settled what the meetings couldn't."

**Stage four: assembly and guardrails.** The business rules engine and assembly service handle sequencing (energy arcs, artist spacing), deduplication, explicit-content and regional licensing checks, and diversity controls. Safety runs at both ends of the pipeline: prompt sanitization before the LLM, content rules after ranking. Offensive prompts are rejected, and prompts that suggest distress are routed to curated, sensitively handled responses rather than algorithmic output.

### Latency: the two-second religion

Internal UX research (an assumption on my part, though consistent with published industry findings) showed engagement dropping sharply beyond roughly two seconds of perceived wait. The sequence below shows where the milliseconds go, and why the client starts rendering before the pipeline finishes.

```
 Client            Gateway           Intent LLM         Retrieval          Ranker
   │                  │            (cached ~35%)            │                 │
   │───── prompt ────▶│                  │                  │                 │
   │                  │──── extract ────▶│                  │                 │
   │                  │  (timeout 800ms) │ ~300-600ms       │                 │
   │                  │◀── intent JSON ──│ ~0ms on cache    │                 │
   │                  │                  │ hit              │                 │
   │                  │──── embed + ANN query ─────────────▶│                 │
   │                  │◀─── ~5K candidates (~50ms) ─────────│                 │
   │                  │                                     │                 │
   │                  │──── rescore vs taste vector ─────────────────────────▶│
   │                  │◀─── first ranked batch (~150ms) ──────────────────────│
   │◀─ first tracks ──│     skeleton UI starts filling      │                 │
   │                  │◀─── remaining batches ────────────────────────────────│
   │◀──── complete ───│                                     │                 │
```

The team attacked the budget from three directions. First, semantic caching. Prompts cluster heavily; thousands of people ask for "gym motivation" in slightly different phrasing every hour, so caching intent objects keyed on prompt embedding similarity let a substantial share of requests skip the LLM entirely. "Well over a third at launch," Adeyemi told me, "and it climbs every week as the cache warms."

Second, model right-sizing. A frontier model generated the training data, and a small distilled model was fine-tuned on it, trading slight nuance on exotic prompts for roughly a tenfold cost reduction and a p95 the team could actually serve.

Third, perceived latency. Streaming the first tracks into a skeleton UI makes the feature feel faster than its measured end-to-end p95. Latency engineering, it turns out, is partly a client-side discipline.

### Reliability: degrade, never fail

Placing an LLM in the hot path made the reliability engineers uneasy, and their mitigation is my favourite decision in the whole project. Every stage has a dumber fallback, and the chain ends in something a listener would still accept.

```
  ┌─────────────────────────────────────┐
  │ LLM intent extraction               │── healthy ──▶ full-quality playlist
  └──────────────┬──────────────────────┘
                 │ timeout / circuit open
                 ▼
  ┌─────────────────────────────────────┐
  │ Keyword extraction                  │── healthy ──▶ full-quality playlist
  │ (mapped onto same intent schema)    │
  └──────────────┬──────────────────────┘
                 │ retrieval degraded
                 ▼
  ┌─────────────────────────────────────┐
  │ Genre & mood catalogue queries      │── healthy ──▶ full-quality playlist
  └──────────────┬──────────────────────┘
                 │ total failure
                 ▼
  ┌─────────────────────────────────────┐
  │ Taste-based personalised playlist,  │
  │ honestly labelled                   │
  └──────────────┬──────────────────────┘
                 │
                 ▼
     Acceptable playlist. Never an error screen.
```

If the LLM breaches its 800 millisecond timeout, the system falls back to keyword extraction mapped onto the same intent schema. If vector search degrades, retrieval falls back to genre and mood queries. If everything fails at once, the user gets a taste-based personalised playlist, honestly labelled. Worse, yes. But there is no error state a listener will ever see.

"We treated no playlist as an outage, and a mediocre playlist as graceful degradation," Kemi Balogun, the reliability lead, told me. That one reframing shaped every SLO the team wrote.

### Experimentation: measuring a vibe

Clicks and stream counts cannot tell you whether a playlist matched its prompt. The team built a three-layer evaluation stack, each layer catching what the one below it misses.

```
  ┌────────────────────────────────────────────────────────────┐
  │ LAYER 1 · Offline, LLM-as-judge                            │
  │ ~10K human-labelled golden prompts                         │
  │ Gates every model change in CI                             │
  └──────────────────────────────┬─────────────────────────────┘
                                 │
                                 ▼
  ┌────────────────────────────────────────────────────────────┐
  │ LAYER 2 · Human evaluation panels                          │
  │ Rate prompt-to-playlist fit on sampled production traffic  │
  └──────────────────────────────┬─────────────────────────────┘
                                 │
                                 ▼
  ┌────────────────────────────────────────────────────────────┐
  │ LAYER 3 · Online A/B tests                                 │
  │ 30s skip rate • saves • 7-day playlist re-listens          │
  └──────────────────────────────┬─────────────────────────────┘
                                 │
                                 ▼
                 ★ North star: 7-day re-listens ★
```

The lesson the team kept stressing: the offline and online layers disagreed early. A model that scored higher on intent accuracy produced playlists people skipped more, because it interpreted prompts too literally. "Our best-scoring model took 'sad songs' somewhere genuinely bleak," Raman said. "Accurate, technically. Nobody wanted to be there." Human panels caught what both automated layers missed. Seven-day re-listens became the north star, since returning to a playlist a week later is the strongest available signal that it truly matched the request.

### The trade-offs ledger

Every conversation returned to trade-offs the team chose to write down in design docs rather than settle in chat threads. Quality against latency, resolved in favour of a distilled model plus caching over a frontier model per request. Freshness against cost, resolved by precomputing track embeddings in batch and accepting hours of staleness, because catalogue semantics change slowly. Personalisation against discovery, resolved by tuning the blend to leave deliberate room for unfamiliar tracks, accepting a modestly higher skip rate for long-term catalogue exploration. Build against buy, resolved by launching on external LLM APIs while the distilled in-house model matured. Ship fast first, own the cost curve second.

### What you can take away

1. **Use LLMs as translators, not databases.** Constrain the model to a schema and let retrieval own factual grounding. Hallucination stops being a bug you fight and becomes a failure class you designed out.
2. **Spend the latency budget where users can feel it.** Caching, distillation, and streaming UX bought more than any single model optimisation.
3. **Design the degraded experience first.** When the fallback chain ends in something acceptable, you can put ambitious components in the hot path with confidence.
4. **Treat evaluation as the product.** The pipeline took a quarter to build. The evaluation stack took longer, and mattered more.
5. **Write the trade-offs down.** A decision log turned contentious debates into revisitable, evidence-based choices.

From the text box, AI Playlist looks like magic. Up close, it's something better: rigorous systems engineering that knows exactly where the magic belongs, and where it doesn't.

---

## Task 2 · The Amplification Plan

Let me start with the thing I believe most strongly about distributing engineering content: engineers can smell marketing from three tabs away. Every decision below follows from that. The article earns its reach by being genuinely useful, and my job is to put it where engineers already are, then get out of the way.

A word on sequencing before the channels. Week one is our own house: internal announcement, employee shares, the company's engineering channels. That builds the baseline traffic and social proof everything else feeds on. Weeks two and three are community submissions and newsletter pitches, because curators are far more likely to pick up a piece that already has a pulse. Podcasts and conference talks come after, since those are slow-burn channels with long lead times. I've watched teams do this backwards, pitching newsletters on day one with zero traction to show, and it rarely works.

### Engineering communities

**Why I picked it.** Hacker News is still where engineering blog posts live or die, and I say that as someone who has watched a single front-page run outperform a month of everything else combined. Reddit's r/programming and r/MachineLearning, plus Lobsters, extend the same effect to slightly different crowds. Beyond traffic, the comment sections give me something no analytics tool can: candid engineers telling me exactly what landed and what didn't.

**How I'd use it.** Submit organically, with the article's real headline. No rewrites, no clickbait framing, because these communities bury anything that smells promotional and the shadow of that follows your domain around. I'd time the HN submission for a weekday morning, US Pacific, and have two or three engineers from the actual team briefed and ready to answer questions in the thread within the first hour, under their own names. That's the difference between a link drop and a conversation, and the conversation is what gets remembered.

**What success looks like.** Front page for a few hours, 200 plus points, and a comment section arguing about the architecture rather than accusing us of PR. In numbers: 15,000 or more sessions from community referrals in week one.

### GitHub itself

**Why I picked it.** The article already lives in a repo with plain-text diagrams, which means GitHub isn't just where the content sits. It's a distribution channel in its own right. Developers discover repos through GitHub Trending (daily, weekly, monthly) and through the big curated awesome-lists, some carrying tens of thousands of stars, and both put you in front of engineers globally with zero ad spend.

**How I'd use it.** Two moves. First, make the repo itself star-worthy: the diagrams, the assumptions section, and the takeaways are all built for engineers who skim READMEs for a living. Then I'd coordinate the week-one social and community push into a single window, because star velocity in a tight window is what lands a repo on Trending, and Trending is a flywheel that feeds itself. Second, open PRs adding the article to relevant high-star curated lists: awesome-llm, awesome-recsys, awesome-system-design and their cousins. One hard rule: every PR must genuinely fit that list's contribution guidelines, because maintainers of popular repos treat drive-by promotional PRs exactly the way Hacker News treats marketing, and a rejected PR from our org account is a public mark against us.

**What success looks like.** 500 plus stars in the first month, at least one day on GitHub Trending, merged into three or more curated lists, and github.com showing up as a referral source in analytics. The lists are the long game: a merged link keeps sending readers for years.

### Technical newsletters

**Why I picked it.** Newsletters like Pointer, TLDR, ByteByteGo and The Pragmatic Engineer have opt-in audiences of exactly the people we want, and inclusion is an editorial endorsement you cannot buy. When a curator engineers trust says "read this," it lands differently than anything we say about ourselves.

**How I'd use it.** Individual pitches to each curator within 48 hours of publishing. Two sentences, leading with the transferable lesson (the LLM-as-translator pattern, the fallback chain), not the product. I'd attach the architecture diagram, because curators love pieces they can illustrate. And I'd keep a simple log of who picked it up organically versus through my outreach, because that log is how my pitching gets sharper over time.

**What success looks like.** Three or more established newsletters within three weeks, each sending 1,000 plus sessions. The tell I really watch is time on page from newsletter traffic. If it's high, we reached readers, not skimmers.

### Social media channels

**Why I picked it.** LinkedIn is where engineering leaders and future candidates scroll; X is where the ML and infra crowd actually argues about this stuff in real time. Different rooms, different registers, which is why Task 3 gives each its own asset.

**How I'd use it.** The company engineering account posts, yes, but the real play is the featured engineers posting personally. A named engineer saying "here's the thing I built and the mistake we almost made" beats a brand account every single time, on both reach and trust. On X I'd run the thread from the engineering handle, tag nobody, sell nothing, and let the engineers quote-tweet with their own war stories. Week two, I'd repost the pipeline diagram on its own as a second bite at the apple. Beyond the posts, I'd launch a LinkedIn newsletter edition of the engineering blog, because LinkedIn pushes newsletter issues to subscribers' notifications and inboxes in a way ordinary posts never get, and each new article grows a subscriber base we own on the platform. And I'd work Substack Notes, where a growing crowd of engineers and tech writers hang out and where a sharp excerpt (the fallback chain, or the "sad songs" eval story) travels well with a link home.

**What success looks like.** Around 150,000 combined impressions over six weeks and a click-through above 2 percent, plus 500 or more LinkedIn newsletter subscribers by issue two. But the metric I'd actually brag about is unsolicited shares from respected engineers outside the company. Ten of those and I know the piece has credibility we didn't manufacture.

### Short-form video

**Why I picked it.** Developers scroll TikTok, Instagram Reels and YouTube Shorts like everyone else, and almost no engineering blog meets them there. Sixty seconds of a person explaining one sharp idea over a diagram travels further on these platforms than anything text-based ever will, and it reaches younger engineers who may never open Hacker News.

**How I'd use it.** Green-screen explainers: me or one of the featured engineers on camera, the architecture diagram behind us, one idea per video. The fallback chain was practically designed for this format ("what happens when the AI fails? Watch."), and the "sad songs" eval story is a natural second. Sixty to ninety seconds each, captioned, with the article link in bio and pinned comment. Cut two or three from each podcast appearance too, so every long-form asset feeds the short-form pipeline.

**What success looks like.** 50,000 plus combined views across platforms in six weeks, a visible referral spike within 48 hours of each post, and at least one video breaking well past its account's baseline, which tells me which idea to double down on next time.

### Answer engines (AEO)

**Why I picked it.** A growing share of "how did they build that" questions are no longer typed into Google. They're asked to ChatGPT, Gemini, Perplexity and Claude. If those tools cite our article when someone asks how natural-language playlist generation works, that's durable, compounding distribution that no social post can match, and almost nobody in engineering content is optimising for it yet. Getting there early is the whole point.

**How I'd use it.** First, research: ask each engine the questions our article answers ("how do music apps turn prompts into playlists," "how to put an LLM in a production hot path") and log what they cite and why those sources win. Then optimise for what answer engines reward: crawlable canonical pages, clear question-shaped headings, quotable self-contained passages (the five takeaways are already this), and authority signals, which is where the newsletter pickups, curated-list merges and technical backlinks from every other channel in this plan quietly compound. Then re-test monthly and log whether we surface.

**What success looks like.** Within a quarter, the article cited by at least two of the four major answer engines for our target questions, tracked in a simple monthly log of query, engine, and whether we appeared.

### Developer forums

**Why I picked it.** Dev.to, Hashnode, the MLOps Community Slack, the vector database Discords. Smaller rooms, but full of practitioners building similar systems right now. This traffic converts into long, careful reads and real conversations.

**How I'd use it.** A canonical-linked summary on Dev.to (five takeaways plus the diagram, pointing home to the full piece). In the Slacks and Discords, I only share through people who already live there, never a cold drop from a stranger's account, and I'd frame each share as a question: "how are you all handling LLM fallbacks in production?" Questions start threads. Links end them.

**What success looks like.** 5,000 plus forum sessions, 50 or more genuine comment exchanges, and, the one I quietly hope for, a few inbound emails from engineers who want to talk. Those emails are tomorrow's collaborations and candidates.

### Podcasts

**Why I picked it.** A 45-minute conversation builds more affinity than fifty posts, and senior engineers consume podcasts in the gaps of their day. Software Engineering Daily, The Changelog, Practical AI and Latent Space all cover exactly this LLMs-meet-production territory.

**How I'd use it.** Pitch the tech lead and the reliability lead as guests, and lead the pitch with the line that hooked me in the interviews: "we treated no playlist as an outage, and a mediocre playlist as graceful degradation." Hosts book guests around one memorable idea, so I hand them the idea. Before any recording, I'd prep the guests properly: three portable stories from the build, practised out loud.

**What success looks like.** Two bookings within eight weeks, a visible traffic echo to the article after each episode, and five or more clips cut for social afterwards. Podcasts are gifts that keep giving if you actually harvest them.

### Conferences and events

**Why I picked it.** A talk at QCon or LeadDev turns the article into reputation, recruiting pipeline, and a recording that works for years. It's the slowest channel here and the one with the longest tail.

**How I'd use it.** Adapt the article into a CFP titled after its strongest lesson, something like "Design the Degraded Experience First: Shipping LLMs in the Hot Path." Submit to two major conferences and three regional meetups, offer the engineers as speakers, and take everything annoying (logistics, slide polish, rehearsals) off their plates. That last part matters more than people admit; engineers say yes to speaking when someone else absorbs the friction. Afterwards, publish the deck with a link back to the article.

Two touches I'd add from watching what actually works at events. First, a QR code on the final slide of every talk and workshop that goes straight to the article, because "the link is in the slides somewhere" loses ninety percent of the room and a QR code on screen for thirty seconds loses almost nobody. Second, swag with a job to do: two or three limited-run Company XY T-shirt designs with the article's QR code worked into the design. Good swag is the only conference giveaway engineers actually keep, and a shirt they wear to the next three conferences is a walking, scannable billboard for our engineering brand. The engineer gets a shirt they like; we get distribution that outlives the event. Everybody wins.

**Where I'd show up.** My target list, in rough priority order: AI Engineer World's Fair (US), the major PyCons across Africa, Europe and the US, DevFest Lagos, DevFest Nairobi and the other big DevFests across Africa and Europe, GITEX Nigeria and its sister GITEX events in other cities, DataFestAfrica, Deep Learning Indaba, Code with Claude in London, AWS community events across African cities and the UK, and the blue-chip data and AI events where engineering leaders gather. The weighting is deliberate. The African developer conferences are underserved by big-company engineering stories and over-index on exactly the engineers we want to reach and hire, so we get outsized attention there, while the US and European flagships carry the global credibility signal. Run both and the story compounds in two directions at once.

**What success looks like.** One major acceptance and two meetup talks within two quarters, 10,000 plus views on the recorded talk within six months, QR scan counts per event logged in the tracker, and shirts spotted at conferences we didn't attend, which is the cheapest brand-reach metric I'll ever report.

### Internal advocacy

**Why I picked it.** This one gets skipped constantly and it's the cheapest channel we have. If our own engineers and recruiters don't know the article exists, we've thrown away our most credible distribution. There's a second reason too: visible internal celebration of the featured team is what convinces the next team to give me their story. This program only compounds if engineers want to be in it.

**How I'd use it.** A slot in the engineering all-hands on launch day, a paragraph in the internal newsletter with suggested talking points, and a direct brief to recruiting: attach this article to outbound messages for relevant roles, here's the one-line framing. I'd also nudge engineering leadership to reference it in their own external appearances.

**What success looks like.** Recruiting confirms they're using it within two weeks, and, the leading indicator I care about most, at least two other teams come to me volunteering their projects for the next article.

### Employee amplification

**Why I picked it.** Employee networks reach five to ten times what brand accounts do, and a post from someone who actually built the thing carries a kind of authenticity you cannot fake from a logo.

**How I'd use it.** An amplification kit inside the internal announcement: three post variants per platform, the diagrams as downloadable images, and an explicit note saying please rewrite this in your own words. Identical copy-paste posts get suppressed by the algorithms and, worse, look like astroturfing to exactly the audience we're courting. Participation stays opt-in, and I'd shout out the best posts in the following all-hands.

**What success looks like.** Forty plus employees sharing in week one, employee posts driving a quarter or more of social referral traffic, and the featured engineers' personal posts out-performing the brand account. When that last one happens, the strategy is working.

---

## Task 3 · Content Repurposing

The through-line for every format is the same hook that carried the article: **the LLM is never allowed to name a song.**

### The LinkedIn post

> Most people assume AI Playlist works by asking an LLM for songs.
>
> It never names a single track. That was the whole point.
>
> I spent the past few weeks sitting with the engineers who built it, and the decision I can't stop thinking about came right at the start: LLMs are brilliant translators and terrible databases. So the model's only job is turning "songs for a rainy Sunday morning that slowly wake me up" into a structured intent object. The retrieval, ranking and personalisation stay with recommendation systems the team has run in production for years.
>
> Which means hallucination isn't a bug they fight. It's a failure class they designed out.
>
> Three more things from the interviews that stayed with me:
>
> They designed the degraded experience first. Every stage has a dumber fallback, so the worst thing a listener ever sees is a mediocre playlist. Never an error screen.
>
> They treated evaluation as the product. The pipeline took a quarter to build. The evaluation stack took longer, and mattered more.
>
> They wrote every trade-off down. Quality vs latency. Freshness vs cost. Personalisation vs discovery. All documented, all revisitable.
>
> Full story, diagrams included, in the article below. If you're putting LLMs anywhere near a production hot path, this one's worth your coffee break.
>
> [link]

### The X thread

> 1/ Just published the engineering story behind AI Playlist: free-text prompt in, personalised playlist out, under 2 seconds, millions of listeners.
>
> The most important decision? The LLM is never allowed to name a song. 🧵
>
> 2/ Here's why. LLMs are great translators and terrible databases. Ask one for songs and you get hallucinated tracks and zero idea of your taste.
>
> So its only job: turn your prompt into constrained JSON. Moods, genres, tempo, "no sad songs." Never track names.
>
> 3/ Everything after that is proven infrastructure. Vector ANN search cuts 100M tracks to ~5K candidates in tens of milliseconds. The existing rankers rescore against your taste vector.
>
> Hallucination stops being a bug to fight. It's a failure class designed out.
>
> 4/ The latency budget was ~2 seconds and the LLM call alone could have blown it. Three fixes:
>
> • semantic caching (over a third of prompts skip the LLM entirely)
> • a distilled model, ~10x cheaper than the frontier model that trained it
> • streaming the first tracks before ranking even finishes
>
> 5/ My favourite decision was about reliability. Every stage has a dumber fallback.
>
> LLM times out → keyword extraction. Vector search degrades → genre queries. Everything on fire → a taste-based playlist, honestly labelled.
>
> "No playlist is an outage. A mediocre playlist is graceful degradation."
>
> 6/ And the part nobody talks about: how do you measure whether a playlist matched a vibe?
>
> Their offline evals disagreed with the live A/B tests. The "more accurate" model made playlists people skipped, because it took "sad songs" somewhere genuinely bleak. Human panels caught it.
>
> 7/ Five lessons if you're shipping LLMs in a hot path:
>
> • LLMs as translators, not databases
> • spend latency budget where users feel it
> • design the degraded experience first
> • evaluation is the product
> • write the trade-offs down
>
> Full article with diagrams: [link]

### Three newsletter subject lines

1. The LLM behind AI Playlist is never allowed to name a song
2. "No playlist is an outage. A mediocre playlist is graceful degradation."
3. 100M tracks, one prompt, two seconds

### Three alternative article headlines

1. The LLM Never Picks the Songs: Inside the Architecture of AI Playlist
2. Design the Degraded Experience First: What Shipping AI Playlist Taught Us
3. Two Seconds to a Vibe: The Systems Engineering Behind Natural Language Playlists

---

## Task 4 · The Six-Week Dashboard

Before the numbers, the framing I'd give leadership in the first minute: we did not publish this article to collect traffic. We published it to make engineers worldwide take Company XY seriously as an engineering organisation, because that's what fills the recruiting pipeline and buys credibility for every future launch.

And that leads to the measurement philosophy I've learned the hard way: **the fastest way to fail in this role is to report big numbers that mean nothing.** A million impressions of the wrong people is worth less than a hundred senior engineers who read to the end and told a colleague. So this dashboard is built quality-first. It asks three questions in order of weight: how deeply did engineers engage (depth), did we reach the right people (qualified reach), and did it start moving what the business cares about (impact)? Raw volume shows up at the bottom, clearly labelled as context, because volume is an input to success, never the definition of it.

Two disciplines keep the whole thing honest. First, every metric that could be inflated carries a quality gate, because any number that becomes a target will get gamed, including by me, accidentally, if I let it. A "comment" only counts if it engages with a specific technical decision in the piece; "nice article" does not count. A "share" only counts if it comes from a practicing engineer or engineering leader, checked against their profile. A "read" means scrolling past 75 percent, not landing on the page. Second, key metrics get a counter-metric watching them: if sessions climb while read depth falls, I don't report growth, I report a targeting problem.

One honest caveat, stated upfront rather than defensively at week six: depth and qualified reach are mature at six weeks; impact is barely warming up. Recruiting and brand effects lag by a quarter.

### The dashboard

**Tier 1 · Depth. Weighted heaviest, because this is where quality lives.**

| KPI | How I measure it, with the quality gate | Six-week target | Counter-metric I watch | Source |
|---|---|---|---|---|
| Qualified reads | Sessions scrolling past 75% with 4+ min on page. Raw sessions don't count as reads. | 25,000 qualified reads | Qualified-read rate below 30% of sessions = wrong audience | Web analytics |
| Substantive technical engagement | Comments across HN, Reddit, Dev.to that reference a specific decision in the piece (the schema constraint, the fallback chain, the blend parameter). Praise and one-liners excluded. | 40+ | Ratio of substantive to total comments; falling ratio = the piece is attracting skimmers | Manual log, sampled and read by me |
| Earned credibility | Unsolicited shares by verified practicing engineers or eng leaders, and backlinks from technical blogs that engage with the content, not just link it | 10+ shares; 5+ backlinks | Zero shares from people senior to our own team = we impressed juniors only | Social listening, SEO tools |
| Content velocity | Podcast bookings and CFP acceptances where the host or committee references the article | 2 podcasts; 1 CFP | None needed; this one is hard to fake | My pipeline tracker |
| Answer-engine visibility | ChatGPT, Gemini, Perplexity or Claude citing the article for target queries, verified monthly | 2 of 4 by week 12 | Cited but paraphrased wrongly = a content-clarity fix, logged | Monthly AEO log |

**Tier 2 · Qualified reach. Who we reached, not how many.**

| KPI | How I measure it, with the quality gate | Six-week target | Counter-metric I watch | Source |
|---|---|---|---|---|
| Community penetration | HN front page with a comment thread debating the architecture; newsletter pickups, each one an editorial endorsement by a curator engineers trust | Front page + 3 newsletters | High points with hostile or "this is PR" comments = reach without credibility, a net loss | Manual log |
| GitHub traction | Stars and curated-list merges, gated on the referral traffic they actually send. Stars that never become readers are decoration. | 500+ stars; 3+ list merges; github.com in top-5 referrers | Star-to-visit ratio; stars without visits = bot or drive-by stars, excluded from reporting | GitHub insights, analytics |
| Audience seniority | Sampled profile review of commenters, sharers and newsletter-driven visitors: are these working engineers, leads, and hiring-relevant profiles? | Majority practicing engineers in every sample | A young/aspirant-heavy audience is fine for video, a miss for the article | Monthly sample, manual |

**Tier 3 · Impact. What it changed. Lags a quarter; reported with patience.**

| KPI | How I measure it, with the quality gate | Six-week target | Source |
|---|---|---|---|
| Recruiting influence | Candidates who mention the article unprompted in applications or interviews, weighted by stage: an offer-stage mention outweighs ten application mentions. Careers-page clicks tracked but reported as context only. | 15+ mentions, 3+ at interview stage or later | ATS tagging, recruiter debriefs |
| Program health | Teams volunteering their story for the next article, unprompted | 2+ | My intake |
| Brand search lift | Searches for "Company XY engineering" vs pre-launch baseline | +20% | Search console |

**Context metrics. Reported at the bottom, never celebrated.** Total sessions (75K expected), combined social impressions (150K expected), short-form video views (50K expected). These tell me the machine is running. They do not tell me it's working. If leadership asks why impressions aren't a headline number, my answer is one sentence: impressions measure what platforms showed, not what engineers thought.

### Reporting cadence

Weekly one-pager to the ER and comms leads for the first six weeks, covering depth and qualified reach only, because that's the window where I'm still making live calls: a second social push, one more newsletter pitch, a follow-up post. Full dashboard review with leadership at week six, then monthly, since the impact tier needs a quarter to mean anything. Two habits make the reporting trustworthy. Every manual item (a newsletter pickup, a notable share, a candidate mention) goes into the shared tracker within 24 hours, so the weekly report is assembly, not archaeology. And once a month I personally read a sample of the comments and shares behind the numbers, because a dashboard can drift from reality quietly and reading the raw material is the only reliable check.

### What success actually looks like

Here's how I'd open the week-six review, before any chart: with three artifacts. The single best comment thread, where engineers argued about the fallback chain on its merits. The single best unsolicited share, ideally a senior engineer at a company we respect telling their followers to read it. And the single best candidate mention, pulled from a recruiter debrief. If those three artifacts are strong, the article worked, and the numbers will almost always agree. If those three artifacts don't exist, no volume of impressions rescues the verdict.

In dashboard terms: green on Tier 1, majority-green on Tier 2, with Tier 3 showing early signal. And the unfashionable thing said plainly: if raw sessions miss the 75K context figure but qualified reads, substantive engagement and recruiting mentions hit, this article succeeded, full stop. A hundred of the right readers who each tell a colleague beat a hundred thousand scrollers, every time, and I will defend that trade in any leadership meeting.

### If the numbers disappoint

**Qualified reach is low.** Diagnose before spending. If the community submissions never caught, I'd retest with the alternative headlines from Task 3, resubmit to secondary communities, and push the newsletter pitches harder with the diagrams attached. I'd also cut the strongest single insight, probably the fallback chain, into a standalone short post to buy a second discovery moment. Same substance, new door.

**Reach is fine, depth is low.** That's a content problem, not a distribution problem, and it stings, but pretending otherwise wastes a quarter. I'd ask three external engineers I trust for a brutally honest read, publish a follow-up that goes one level deeper (the evaluation stack, with the real failure cases), and adjust the next article's voice based on what they tell me. You cannot promote your way out of a piece that didn't land.

**Depth is fine, impact is lagging.** At six weeks, expected. I'd verify the plumbing (is ATS tagging actually catching article mentions, do recruiters remember the brief), re-brief where needed, and hold course until week twelve before touching strategy.

**Volume is huge but quality gates are failing.** The trap scenario, and the one a less experienced version of me would have reported as a win. Big numbers with shallow reads means we reached the wrong audience at scale, which burns budget and teaches the algorithms to keep sending us the wrong people. I'd narrow distribution, not widen it: fewer channels, more precisely the ones where the quality metrics were healthiest.

**Whatever happens,** the findings go into the playbook. The article is one asset. The thing I'm really building is a repeatable engineering storytelling engine, and every launch should make the next one cheaper and better.

---

## Assumptions, and why I'm comfortable with them

The brief asked me to make and flag reasonable assumptions. Collected here:

1. **A catalogue of roughly 100M tracks.** In line with figures major streaming platforms publish about their libraries.
2. **An existing embedding-based recommendation stack.** Every major streaming service has publicly documented some version of this; a natural-language feature would be built on top of it, not beside it.
3. **An ANN vector index (e.g. HNSW) over precomputed track embeddings.** The standard retrieval approach at this catalogue scale; exact search would be prohibitively slow.
4. **A ~2 second engagement cliff.** Consistent with widely published UX latency research.
5. **Frontier-model-to-distilled-model training.** The dominant industry pattern for putting LLM capability in a latency- and cost-sensitive hot path.
6. **A ~10K golden prompt eval set and a mature A/B culture.** Typical of platforms with established experimentation infrastructure.
7. **Named engineers.** The brief says to assume the interviews happened, so the engineers quoted (Adeyemi, Raman, Balogun) are composites named for readability. Every claim they voice is a real engineering position I'd defend.

None of these change the shape of the story. The narrative rests on the patterns, which are real, not the precise figures, which are illustrative.

---

*If you'd like to argue with any of this, the issues tab is open. That's rather the point.*
