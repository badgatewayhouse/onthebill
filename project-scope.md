# Project Scope — Working Title: "On the Bill" (front-runner, not final)

*Naming shortlist also includes: Handbill, Openers, Doors. Decision deferred; check domains (.com, .show, .live) and trademarks before printing anything. Working slug until then: `onthebill`.*

*An Apps With Intent project. Drafted June 2026.*

## Next Priority: Design

Before any scaffolding, a deliberate design pass on **the card** — typography, photo crop behavior, genre tag treatment, the player-reveal interaction. The card is the thing the venue judges in the demo, the component every surface reuses (embed, flier, gallery, eventually widget), and the layout Edict #4 freezes. It is the single highest-leverage artifact in the project; code waits for it.

---

## North Star

A fan standing anywhere — a venue's website, a flier on a pole, a browse page for a city they're visiting — can answer *who's playing, where, when, and what do they sound like* in two clicks, without entering social media, without ads, without a subscription.

The product is not band pages. The product is the **event graph**: bands × venues × dates. Every visible surface (band profile, event flier, venue widget, city browse) is a *projection* of that graph. This is the single most important framing in this document.

---

## The Edict

All technical decisions are made with the stretch goals in mind. Concretely:

1. **Bands and venues are atoms; events are molecules.** Bands are the atomic *content* unit: they hold all media, exist independently of any event, and must be fully useful with zero shows booked. Events are the atomic *product* unit: every distribution surface — flier, QR, venue widget, city browse — is a render of an event or an event query. Two corollaries: (a) never denormalize atom data into molecules — fliers and widgets always render the band's current record, so updates propagate everywhere, including pages behind printed QR codes; (b) identity, claiming, renames, and dispute handling live on atoms, never on events. If a feature can't be expressed as atoms composed into molecules and projected, question the feature.
2. **Never host audio.** Playback is always a whitelisted third-party embed (Bandcamp first; provider field is pluggable). This is a permanent constraint, not a cost-saving phase. It keeps licensing, transcoding, and bandwidth permanently off the table.
3. **Embeds and permalinks are sacred.** Any URL given to a venue or printed on a flier (especially as a QR code) works forever. Embed and flier routes are statically rendered and CDN-cached so they survive a database outage. Slugs are never recycled. Renames leave redirects.
4. **Opinionated, fixed-geometry layouts.** One card design, known dimensions, responsive within them. No themes, no customization. This is the calm-design moat *and* what makes iframes viable on Squarespace/Wix/WordPress without height-resizing scripts.
5. **Structured data everywhere.** Every event page emits schema.org `MusicEvent` JSON-LD; every band page emits `MusicGroup`. SEO and Google event surfaces are a core venue-side value proposition, not an afterthought.
6. **Data is portable.** Bands and venues can export everything that's theirs. No lock-in mechanics, no engagement mechanics, ever.
7. **Roles are scoped, entry is delegated.** Curator (global), booker (scoped to venue), band (scoped to band). The schema assumes multiple writers per entity from the start, even while the pilot runs on a curator allowlist.
8. **Atoms are authored only by their owners.** Band records are created and edited exclusively by the band (curator acts as the band's proxy during the pilot, with their materials and consent). Bookers never create band records. An event may reference a band that doesn't exist yet via a *pending lineup slot* (name + invite link); the slot binds to the real record when the band completes their profile. The band table therefore contains only complete, consented, rights-cleared records — no stubs, ever.

---

## Data Model (v1)

```
bands         id, slug, name, city, blurb (≤140 chars),
              genre_tags text[] (controlled vocab, max 3),
              music_provider enum('bandcamp','youtube','soundcloud'),
              music_ref (canonical params, never raw embed HTML),
              created_by, updated_at

band_photos   id, band_id, storage_path, width, height,
              credit (photographer), position (1–2 enforced)

venues        id, slug, name, city, address, geo point,
              website, created_by

events        id, slug, venue_id, starts_at timestamptz,
              door_time, price_text, status enum('draft','published','cancelled'),
              flyer_style (reserved), created_by

event_lineup  event_id, band_id nullable, position int,
              role enum('headliner','support'),  -- openers are first-class
              invited_name, invite_token, claimed_at
              -- band_id XOR invited_name: a slot is either bound to a
              -- real band or pending. Pending slots render as plain text
              -- (no media); the invite link lets the band complete their
              -- profile, which binds the slot and re-renders all surfaces.

users         supabase auth
roles         user_id, role enum('curator','booker','band'),
              scope_type, scope_id               -- booker→venue, band→band
```

Notes:
- `music_ref` stores parsed canonical identifiers (e.g. Bandcamp album/track id + subdomain), validated server-side against a provider whitelist. Embed HTML is generated at render time from these params.
- `events.slug` is human-shareable (`/e/2026-07-12-the-broadberry-…`) because it gets printed as a QR.
- Geo on venues now, even though nothing queries it until city browse. It's one column; backfilling later is a chore.

---

## MVP — "Useful in Weeks"

Delivery sequence: **Demo → Pilot → Fast follows.** Render path first, entry path second.

### Phase 0 — Demo (pre-sign-off)

Tangible artifacts for the venue pitch, no forms required — data seeded by script:

- **Band cards** (page + embed) and **flier cards** (event page + embed) for a handful of real local bands and one or two real upcoming shows.
- Per-band consent text before mocking anyone up (Edict #8 holds from day zero — and that text doubles as the band-side buy-in conversation).
- A one-pager per card for the venue: bare link, iframe snippet, screenshot. The pitch recipient and the website editor are often different people; the screenshot is what gets forwarded.

### Phase 1 — Pilot build

**Deliverable:** one real venue's month of shows, live, with every band hearable, plus a printable QR flier per show.

### In scope

1. **Schema + Supabase setup** as above, with RLS policies for curator/booker roles.
2. **Entry UI** (curator-only in v1; allowlisted Supabase auth): create/edit venues, events, lineups, and band records (curator proxying bands via the invite-link flow — the same form that becomes self-serve in S1). The lineup picker searches existing bands; an unmatched name becomes a pending text-only slot with a generated invite link. Forms can be rough — one internal user. Booker role ships with S2's widget, not before. Image upload with server-side resize/compress (cap ~400KB, strip EXIF), photographer credit field required.
3. **Band profile page** — `/b/{slug}`: name, photo, tags, blurb, player, upcoming shows (a projection of the event graph — proof the model works). `MusicGroup` JSON-LD.
4. **Event page / digital flier** — `/e/{slug}`: lineup in order (openers included), venue, date, doors, price, one tap to hear each band. This page *is* the flier — designed to be screenshotted, shared as a link, and beautiful on a phone. `MusicEvent` JSON-LD.
5. **Band embed** — `/embed/b/{slug}`: zero-chrome card, `frame-ancestors *`, fixed geometry, statically rendered, revalidated on publish. Click-to-load the audio player (no nested iframe until interaction).
6. **QR codes** — auto-generated per event and per band, downloadable as SVG/PNG from the entry UI, sized for print. Band slaps it on the physical flier; the pole flier becomes the EF.
7. **Gallery** — `/bands`: public, statically-rendered scrollable grid of the same band-card component the embeds use. Click-to-load players only (never eager-load a grid of Bandcamp iframes). Alphabetical or recently-added sort; no algorithmic ordering, no infinite-scroll mechanics — a calm, finite list. `/shows` (upcoming flier cards) follows immediately after, same pattern. This is the embryo of S4's city browse and the standing sales artifact.
8. **Minimal analytics** — impressions and player-load clicks, counted server-side, tagged by surface (`embed:band` / `page:event` / `page:band` / `gallery`) and referrer class (venue-site / QR / direct). This answers the pilot's real question: not *whether* fans press play, but *on which surface*.

### Explicitly out (MVP)

- Band self-serve auth and editing (phase 2 — curator/booker entry covers the pilot)
- Venue lineup widget (stretch — but `/embed/v/{slug}` route namespace is reserved and the query already exists)
- Browse/discovery site
- Generated print-layout fliers (the event page is the flier for now)
- Search, follows, notifications, anything resembling engagement mechanics (most of these are *permanently* out per the philosophy)

### Stack

Next.js (App Router, static rendering for all public/embed routes, on-demand revalidation on publish) · Vercel · Supabase (auth, Postgres, RLS) · images in Supabase Storage initially with a clean seam to move to Cloudflare R2 if egress grows · sharp for the upload pipeline · `qrcode` lib for QR generation. Cost at pilot scale: ~$0.

---

## Stretch Goals (in intended order)

**S1 — Fast follows on verified buy-in: band entry form + venue flier-creation form.** Neither is new construction — both are the curator's rough forms gaining auth scoping and polish. (a) *Band self-serve* (load-bearing per Edict #8 — the only band-creation path once the curator stops proxying): invite-link claim flow from pending slots, plus curator-approved cold signup for impersonation/name-collision control. (b) *Flier creation for bookers* — the booker role arriving early, demand-pulled: lineup picker + pending slots + date/price/doors, **nothing else**. No layout options, no colors, no themes (Edict #4). The moment a venue asks to customize the flier, the answer is no — opinionated layout is the product.

**S2 — Venue widget.** `/embed/v/{slug}`: one permanent iframe a venue installs once, showing upcoming published events. The venue's website is touched exactly once, ever; everything after is data entry in our dashboard. Pure projection of the event graph — no new schema. Pitch: "embed this and your shows appear in Google" (the `MusicEvent` JSON-LD on event pages does the SEO work).

**S3 — Ad-hoc flier generator.** Arbitrary combination of bands + venue + date → rendered flier in our house style, as a page and as a print/social export (satori/resvg or Playwright render to PNG at poster and story dimensions). `flyer_style` column comes alive. This is the "host the EFs" goal, generalized: every flier is just an event render, including hypothetical/unbooked combinations for bookers planning a bill.

**S4 — City browse.** Public, read-only: pick a city, see venues, see upcoming events, hear every band. The traveling-fan product. The MVP's `/bands` and `/shows` galleries *are* this product for one city; S4 is adding geo facets and curated city-by-city rollout to keep quality high. Zero new write paths.

**S5 — Link-rot sentinel.** Scheduled job HEAD-checks every `music_ref`; failures flag to curator dashboard, never silently break a venue page.

*(S-numbers above supersede earlier drafts: self-serve moved to S1 because Edict #8 makes it the only band-creation path after the pilot.)*

---

## Pilot Plan

Curator-driven: the curator (Stuart) does all data entry, proxying bands through the invite-link flow so the S1 form gets exercised on real content. No booker role needed in v1 — curator-only auth; the roles table waits in the schema.

1. Pick one venue with (a) a website someone can edit, (b) a booker willing to paste what they're handed — both already identified in the local network.
2. Enter one month of shows: every band, openers included, with Bandcamp links. Bands without materials stay as pending text-only slots.
3. Run **both surfaces** as parallel experiments:
   - **Per-band embeds** handed to the venue for their existing event pages — tests the atom in context.
   - **Event pages as fliers** — linked from the venue site and printed as QR posters for at least two shows — tests the molecule as a destination.
4. Analytics tag every impression and player-load with surface (`embed:band` / `page:event` / `page:band`) and referrer class (venue-site / QR / direct). The pilot's output is not "did people press play" but **which surface earns plays** — that answer decides whether S2 (venue widget) or S3 (flier generator) is the real product.
5. Secondary signals: does the booker ask for next month unprompted; do invited bands complete profiles from the invite link alone, and how long does the form take them.
6. Decision gate: healthy play rates → build S1 (band self-serve) plus whichever of S2/S3 the surface data points to. Crickets on both surfaces → the problem may be real but the surface wrong; reassess before building more.

## Known Risks (accepted, monitored)

- **Bandcamp dependency** — ownership turbulence post-Songtradr; mitigated by pluggable provider field. YouTube fails the no-ads criterion; acceptable as fallback only.
- **Reliability contract** — broken embeds on venue sites are reputation-fatal in a small scene; mitigated by static rendering + CDN (Edict #3).
- **Photo rights** — required credit field + curator review during pilot; formal claim/takedown flow lands with S1 self-serve.
- **Unresponsive bands** — some pending lineup slots will never be claimed and render as text-only forever. Accepted: it degrades gracefully, accurately reflects the scene, and the visible gap is itself the recruitment pressure.
- **Staleness** — breakups, deleted Bandcamp pages; mitigated by S5 sentinel and curator flagging.

---

## Appendix: Decision Log

*Append-only. One line per settled decision, dated. The spec above always reflects current state; this log records how it got there. Promote to `DECISIONS.md`/ADRs when entries need real rationale or the log outgrows a screen.*

- **2026-06-10** — Never host audio; playback is whitelisted third-party embeds (Bandcamp first, provider pluggable). Permanent constraint.
- **2026-06-10** — Bands/venues are atoms (content unit), events are molecules (product unit); all surfaces are projections of the event graph. No denormalizing atom data into molecules.
- **2026-06-10** — Band records are authored only by bands (curator proxies during pilot). Bookers never create band records.
- **2026-06-10** — Unknown bands on a bill become pending lineup slots (text-only, invite link) rather than stub records. Events publish immediately; unclaimed slots stay text-only forever, accepted.
- **2026-06-10** — Opinionated fixed-geometry card; no themes or customization, ever. Venue flier form is lineup + date/price/doors, nothing else.
- **2026-06-10** — YouTube fails the no-ads criterion; fallback only. Bandcamp is the primary provider despite ownership risk.
- **2026-06-10** — Embeds/permalinks are sacred: statically rendered, CDN-cached, slugs never recycled.
- **2026-06-10** — Delivery sequence: Demo (seeded data, band + flier cards, venue one-pagers) → curator-driven pilot → fast follows (band entry form + venue flier form) on verified buy-in.
- **2026-06-10** — Pilot runs both surfaces (per-band embeds vs. event-page fliers/QR) with per-surface, per-referrer analytics; the output is *which surface earns plays*, deciding venue-widget vs. flier-generator priority.
- **2026-06-10** — QR codes carry source params (`?s=qr-<poster>`) for per-poster attribution.
- **2026-06-10** — Gallery (`/bands`, then `/shows`) ships in MVP as public index reusing the card component; calm finite list, click-to-load players, no algorithmic ordering.
- **2026-06-10** — Card design precedes all code.
- **2026-06-10** — Name: "On the Bill" is the front-runner (shortlist: Handbill, Openers, Doors); not final pending domain/trademark checks.
- **2026-06-10** — Decision log lives as this appendix until it outgrows the format; spec stays present-tense, log is append-only.
