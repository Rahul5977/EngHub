# Interview prep — Student-Counselor (AWS-heavy)

> Three parts. **Part 1** is the project's own Q&A (from `README.md` §15) with follow-up chains added —
> answer the bold question, then be ready for the indented follow-ups, which is how a real interviewer
> actually drills. **Part 2** is generic AWS depth that isn't project-specific but a serverless-on-AWS
> interview will hit anyway — know these cold, then bridge back to the project. **Part 3** is real
> incidents from actually building and deploying this thing — these are your best "tell me about a bug"
> answers because they're true and nobody else has them.
>
> Numbers to have cold before you walk in: **360,975** dataset rows, **11,261** live cutoffs, **121**
> institutes, **5 lakh** target students, **50k** concurrent spike, **5k req/s** predictor burst,
> **13.2%** median forecast error (was 39.6%), **92.9%** band coverage vs 80% target, **9** bounded
> contexts, **≈$0/mo** idle, **$20–100/mo** peak season.

---

## Part 1 — Project Q&A with follow-ups

### A. Openers

**1. Walk me through the predictor end-to-end — from a rank typed in to a result on screen.**
Filter to competition set (seatType/gender/quota) → quota selection (AI → HS-if-home-state → OS) →
exam mapping (Advanced rank for IIT, Main rank for NIT/IIIT/GFTI) → ratio buckets (Safe ≤0.80, Target
≤0.80–1.10, Reach >1.10, window 0.2–1.6) → sort by closing-rank ascending → forecast layer supplies the
headline chance %.
   - *Follow-up: where does each step physically run — edge, Lambda, DB?* CloudFront serves ~90%+ of
     repeat slices; on a miss, Lambda does everything (filter+sort is an in-memory array op); DynamoDB is
     only touched once per cold start to load the snapshot, never per request.
   - *Follow-up: what's the actual data structure the snapshot is held in — array, map, index?* It's the
     parsed cutoff rows kept as a flat array in module scope; the predictor is a linear filter+sort over
     ~11k rows, which is fast enough (single-digit ms) that no secondary index was built. Be honest this
     is a scale ceiling (see Q30/Q46 below) — an index by (institute, program) would be the first move if
     the dataset grew 10–100×.
   - *Follow-up: what happens on the very first request after a deploy, before any snapshot is loaded?*
     Cold start pays one bulk `Query` against `CUTOFF#<activeVersion>`, deserializes, caches in module
     scope; every subsequent warm invocation on that execution environment reuses it for free.

**2. Resume says 11,261 cutoffs / 121 institutes — why is that small, and does size matter?**
That's the *served snapshot* — one year, final round, active version. The full historical corpus behind
the forecasts is 360,975 rows across 2020–2025, all rounds. The architectural insight is that the served
dataset is small **and immutable within a round** — that's precisely why compute-not-query works at all.
   - *Follow-up: what if next year's dataset grows 50×?* Module-memory caching degrades — that's the
     documented ADR-008 reversal trigger: add Redis/DAX, or shard the snapshot per institute-type so each
     Lambda only loads what it needs.

**3. 50k concurrent on result day — what breaks first, and how do you know?**
Nothing before the origin — ~90% of reads die at CloudFront. The earliest failure signal is a **cache
hit-ratio drop**, not CPU or memory. After that: Lambda account concurrency (raised pre-season via limit
increase), then DynamoDB — which is on-demand and barely touched on the read path anyway.
   - *Follow-up: how would you actually detect the hit-ratio drop before users notice?* CloudWatch
     `CloudFrontCacheHitRate` metric on an alarm, checked against the leading-indicator claim in Q41 — if
     I can't point to a concrete alarm/dashboard I built, say so honestly rather than invent one.
   - *Follow-up: what's your mitigation if hit ratio DOES drop mid-spike?* Cache-key normalization is the
     preventive control (fewer distinct keys → fewer misses); the reactive control is `stale-while-revalidate`
     absorbing the miss storm so the origin sees one request per key, not 50k.

**4. Why serverless? Defend it against a Fargate service behind an ALB.**
Scale-to-zero matches 10 idle months; spikes are predictable *in time* (JoSAA publishes the round
schedule) but not worth owning standing capacity for; small team, 2-month product. Concede honestly:
containers win at steady 24×7 load, long-lived connections, WebSocket fan-out — this product doesn't have
those on the hot path (video is explicitly carved out to a managed SFU, §16 in Part 2 covers the general
Lambda-vs-Fargate tradeoff).
   - *Follow-up: quantify it — what would Fargate cost you for the same idle 10 months?* Even a minimal
     always-on ECS service (say 1 task, 0.25 vCPU/0.5GB) is a non-zero monthly floor × 10 idle months;
     Lambda's is $0. That's the whole argument in one line.
   - *Follow-up: if the product weren't seasonal — steady year-round traffic — would you still pick Lambda?*
     No — at steady high volume, Lambda's per-request pricing eventually crosses over a provisioned
     alternative; the crossover point is exactly what you'd model before deciding. Say that instead of
     dogmatically defending serverless everywhere.

### B. Caching & performance

**5. Predictor output is personalized (rank differs per student) — how is it cacheable at a CDN?**
The result is a *pure function* of (rank, category, home state, filters), not of user identity. Inputs
are normalized into a stable cache key so many students collapse onto the same entry.
   - *Follow-up: what does "normalized" concretely mean — do you bucket ranks?* Sorted/canonicalized
     filter sets into a stable hash; the point of normalization is collapsing near-duplicate requests
     (different filter *order*, same filter *set*) onto one key, not lossy rank-bucketing that would
     change the answer.
   - *Follow-up: does this leak information between students?* No — the response contains no PII, it's a
     computed college list; caching it doesn't expose one student's rank to another because the cache key
     is derived from, not equal to, identity.

**6. How do you invalidate cache when a new round publishes?**
Immutable versioned snapshots (`CUTOFF#<version>`) + an atomic `activeVersion` pointer flip + one CDN
invalidation of `/predict/*` and `/analysis/*`. Rollback = flip the pointer back.
   - *Follow-up: is the pointer flip itself atomic under concurrent readers?* Yes by construction — reads
     always resolve "what's active" via a single config item; a reader either sees the old or the new
     version, never a torn mix, because nothing about an in-flight snapshot mutates.

**7. What stops a cache stampede at the invalidation moment, at peak traffic?**
`stale-while-revalidate`: the edge keeps serving the just-invalidated stale entry to everyone while
exactly one request revalidates in the background. Origin sees one miss per key, not 50k.
   - *Follow-up: what if revalidation itself is slow — does everyone wait?* No — SWR's whole point is
     nobody waits on the revalidating request; they get stale content immediately, and the next request
     after revalidation completes gets fresh content. Slow revalidation extends the stale window, it
     doesn't cause a pile-up.

**8. Why Lambda module memory instead of Redis/ElastiCache? When would you add it?**
ADR-008: ~11k rows fit trivially in process memory; ElastiCache has a real fixed floor (~$12+/mo minimum
node) that buys nothing at this dataset size. Add it only if profiling at real scale shows cold-start
snapshot loads are the bottleneck.
   - *Follow-up: what's the actual failure mode module memory has that Redis wouldn't?* Per-execution-
     environment cache — a burst of fresh cold starts (e.g. Lambda scaling out hard on a spike) means many
     concurrent environments each pay their own DynamoDB load, simultaneously. Redis would be one shared
     cache all of them hit instead. This is the honest cost of the choice, not a hidden flaw.
   - *Follow-up: DAX vs standalone ElastiCache Redis — which would you pick if you added a cache layer?*
     DAX is DynamoDB-API-compatible (near drop-in, no app-level cache logic) but only helps DynamoDB reads
     specifically; standalone Redis is general-purpose and could also cache computed result slices, not
     just raw rows. Given the predictor computes over the whole dataset anyway, Redis is the more useful
     of the two here.

**9. What's your cold-start story, concretely?**
ARM64/Graviton (~20% cheaper per ms, comparable-or-better latency), esbuild bundles under 5MB (tree-shook
AWS SDK v3), lambdalith-per-context (few enough functions to stay warm), scheduled provisioned concurrency
only for the Jun–Jul window, pre-warmed ahead of published round-result timestamps.
   - *Follow-up: how do you know WHEN to pre-warm — is that automated?* JoSAA publishes the round-result
     schedule in advance; the documented mechanism (architecture.md §8.4) is EventBridge-scheduled jobs
     that ramp provisioned concurrency up before a known spike and down after, gated by a `season: on|off`
     config flag. Be honest about whether this specific automation is built vs. still a documented plan —
     check `progress.md` before claiming it's live.
   - *Follow-up: SnapStart — did you consider it, why/why not?* (This is a real gap — see Part 2 Q17. Node
     Lambda SnapStart didn't exist/wasn't GA when this was designed for Node runtimes the way it is for
     Java; worth knowing the current state before the interview since AWS ships fast.)

**10. A cold Lambda has to load 11k rows from DynamoDB — what does that do to p99, and how would you measure it?**
It IS the known p99 tail, bounded to a single bulk read of one partition key (`CUTOFF#<version>`) — cheap
relative to a scan, but still real latency added only on cold paths. Measure via CloudWatch: split p50 vs
p99 by presence of `Init Duration` in the Lambda log line (that field only appears on cold starts).
   - *Follow-up: is that measurement actually wired up, or is it the plan?* If not built, say: "the metric
     exists in CloudWatch natively (Init Duration is logged automatically); building the split-by-cold-
     start dashboard query is on the hardening-phase list, not yet built." Precision here signals honesty
     over a rehearsed answer.

**11. Estimate the memory footprint of the snapshot and per-request CPU cost.**
~11k rows × a few hundred bytes ≈ single-digit MB; a request is filter+sort over an in-memory array —
microseconds to low milliseconds of CPU. This is *why* misses are cheap and horizontal scaling is trivial
— the compute is dominated by network/serialization, not the actual work.
   - *Follow-up: at what row count does this napkin math break down?* Once the array no longer fits
     comfortably in a Lambda's memory tier cheaply, or once linear filter+sort stops being sub-ms — ballpark
     100k–1M rows depending on shape, which is roughly 10–100× current scale. That's the number to have
     ready if pushed for a concrete threshold.

### C. System design & architecture

**12. Why one Lambda per bounded context, not per-route nano-lambdas or one monolith?**
The middle ground: few enough functions to stay warm (nano-lambdas cold-start constantly at low individual
traffic), separate enough for independent deploys/IAM surface/blast radius (a monolith couples an
unrelated service's bug to everything).
   - *Follow-up: what's the actual blast-radius argument — walk through a concrete failure.* A bug in the
     `notifications` handler (e.g. a bad template causing an unhandled exception) throttles/errors only
     that Lambda; `predictor` and `payments` keep serving. In a single mega-lambda, the same bug could
     degrade the account-level concurrency pool shared by every route.

**13. How do services talk to each other?**
They don't, synchronously. Domain writes emit events onto EventBridge → fan out through SQS (+DLQ) to
consumers. Why: a spike or slowdown in one consumer (e.g. notifications) must never cascade backpressure
into the producer (e.g. booking).

**14. Design the booking flow — what are the failure modes?**
Saga: `REQUESTED → ACCEPTED → CONFIRMED (paid) → LIVE → ENDED → RATED` (+ `DECLINED`/`CANCELLED`). Payment
only happens *after* the mentor accepts. Failure modes: double-submit (Idempotency-Key header on create),
payment-webhook replay (ledger keyed on provider event id, condition-fails a duplicate), mentor no-show
(state timeouts / auto-refund trigger).
   - *Follow-up: what if the payment webhook never arrives at all — booking stuck at ACCEPTED forever?*
     This is the honest gap to name if you don't have a concrete answer: a reconciliation job (mentioned in
     the payments-payouts LLD as a daily diff against the Razorpay settlement report) is the intended
     backstop, not a per-booking timeout. Say what's designed vs. what's actually built.

**15. Where would you add a queue you don't have, and where would a queue be the wrong tool?**
Wrong: the predictor read path — it's latency-bound and cacheable, a queue only adds latency for no
correctness benefit. Right: anywhere write bursts could exceed a downstream's comfortable rate (already
true for notifications and analytics via EventBridge→SQS).

**16. Why isn't video on Lambda?**
Media is long-lived, stateful, latency-sensitive — the opposite of what Lambda is good at (short,
stateless, bursty). A managed SFU (100ms / Chime SDK) carries media; Lambda only mints join tokens and
handles webhooks/recording events.
   - *Follow-up: is the video/booking-payment path fully built and deployed, or still boilerplate?* Per
     `progress.md`, payments code is interfaces + boilerplate with `// TODO(owner)` markers for the actual
     Razorpay logic — verify current state before claiming "it's live" in an interview; overclaiming here
     is the easiest way to get caught.

**17. How would you take this multi-region?**
Honest answer: you mostly don't need to — CloudFront is already global edge, only the origin is regional.
If forced: DynamoDB global tables to `ap-south-2`, Route 53 failover; the genuinely hard part is Cognito,
which doesn't replicate user pools cleanly across regions.
   - *Follow-up: what's the actual RPO/RTO story today, single-region?* PITR gives point-in-time restore
     within the region (RPO measured in seconds, RTO is however long a restore + redeploy takes — not a
     tested/drilled number here, and it's honest to say so rather than invent an SLA).

### D. DynamoDB & data modeling

**18. Why DynamoDB over Postgres — what did you give up?**
Known, few access patterns + scale-to-zero + Lambda-friendly (no connection pool problem). Gave up:
ad-hoc queries (recovered via Streams → S3 → Athena) and multi-item transactions beyond what
`TransactWriteItems` natively covers (max 100 items/4MB per transaction, single-region).
   - *Follow-up: give me a query you literally cannot do without Athena.* "Which colleges had the biggest
     closing-rank swing year over year across all quotas" — that's a scan/aggregate across the whole
     history corpus, not a single-item access pattern DynamoDB indexes for; Athena over the S3 lake is the
     right tool.

**19. Walk me through the bookings table's GSIs — why is one date-partitioned?**
GSIs by student, by mentor, and a day-partitioned GSI (`gsi3-byday` in the target design, `gsi1-status`
pattern reused for mentors' time-ordered queues) so an admin console at high daily booking volume never
scans, and no single logical partition grows unbounded forever.
   - *Follow-up: what's the actual partition key shape, and why does day-granularity avoid a hot partition
     better than month or year would?* Finer granularity spreads write/read load across more distinct
     partition key values as volume grows — day-level means a busy season doesn't concentrate all of a
     year's writes onto one key; month/year granularity would eventually re-create the hot-partition
     problem this was built to avoid, just delayed.

**20. How do you avoid hot partitions on result day specifically?**
The hot *read* path never touches DynamoDB — the snapshot lives in Lambda memory. Writes are per-user
keyed (naturally spread across the keyspace). The one shared key — the cutoff snapshot itself — is read
once per cold start, not once per request, so it's never actually "hot" in the DynamoDB sense.

**21. How is the payments ledger idempotent?**
Append-only events keyed `ACCT#<id> / EVT#<ts>#<providerEvtId>` + a GSI on `providerEvtId`. A replayed
webhook writes the identical key → conditional-write failure → no double credit. Balance = fold over the
event stream, never a mutable counter.
   - *Follow-up: what actually enforces the conditional write — walk through the DynamoDB call.* A
     `PutItem` with `ConditionExpression: attribute_not_exists(pk)` (or the SK) — DynamoDB rejects the
     second write atomically server-side; no read-then-write race window exists because the check and the
     write are one atomic operation.

**22. What's your backup/recovery story?**
PITR on all tables (continuous, restorable to any second in the last 35 days); the catalog table is
trivially rebuildable from the committed, checksummed CSV corpus (reseed, not restore); the analytics lake
is append-only S3, inherently durable.
   - *Follow-up: what's NOT covered — what would actually hurt if lost?* A PITR restore creates a *new*
     table, so recovering in place means restoring to a new table then repointing the app — that cutover
     step isn't free, and isn't drilled/tested here; naming that gap is more credible than claiming a
     seamless story.

### E. The algorithm & statistics

**23. Why sort closing-rank-ascending by default, not chance-descending?**
JoSAA allots the *highest choice you clear*, so the result list should read like a choice list you'd
actually submit. Chance-descending floats your weakest, safest backups to #1 — actively harmful advice
disguised as a "better" default. This is the strongest product-sense answer in the whole project — lead
with it if asked "tell me about a UX decision you made for correctness, not polish."

**24. Is the chance % a real probability? Defend it.**
Two-tier honesty by design: where a forecast band exists, yes — `P(admit) = Φ((R̂ − rank)/σ)`, backtested
at 92.9% actual band coverage against an 80% target (slightly conservative, i.e. safe-leaning). Where no
band exists, the ratio-based fallback is explicitly labeled a monotone communication aid, not a
probability — and the API tells the UI which one it got via `chanceBasis`.
   - *Follow-up: 92.9% vs 80% target — is that a good result or a sign of a bug?* Good, with a caveat:
     bands run a bit wide (conservative/safe), which is the right failure direction for this product (better
     to under-promise a safe seat than over-promise a reach), but it also means the bands could be tightened
     for more useful discrimination — a real, honest tradeoff to volunteer, not just cite the number.

**25. Why can't you compare a rank of 10,000 in 2020 with one in 2025 directly?**
The candidate pool grew ~0.87M → ~1.3M over that window — the same rank means a different percentile of
the field each year. Ranks are converted to percentiles per year, trend-fit in logit space (unbounded,
symmetric, safe for extrapolation), then rescaled by the *projected* target-year pool size.

**26. Why a median-of-six ensemble instead of one regression, or an LSTM?**
≤8 data points per series — a deep model is theater, it'd just memorize noise. Six cheap classical
estimators split across two complementary spaces (percentile-space for pool-growth correction,
absolute-log-rank-space for ultra-elite seats whose absolute cutoff barely moves) hedge two distinct
failure modes; the median is robust to any single estimator going wild on a short, noisy series.

**27. How do you handle COVID years and the EWS quota's introduction?**
Explicit anomaly weighting: 2020/2021 get weight ×0.45 in the ensemble; EWS series drop all pre-2019 rows
entirely (the quota category didn't exist yet, so there's no honest history to weight down — it has to be
excluded, not downweighted) and are flagged `limitedHistory` downstream so the UI/consumer knows to trust
the band less.

**28. How did you validate the forecast — and what surprised you?**
Hold-out-a-year backtest: forecast 2024 using only ≤2023 data, compare to actual. The surprise: filling
the 2021–2023 data-completeness gap cut median error from 39.6% to 13.2% — a bigger lever than any
modeling sophistication tried. That's the headline "what did you learn" story for this whole component.

**29. What's a preparatory rank, and why did it almost poison the model?**
IIT `P`-suffixed ranks belong to a *separate* rank list (a distinct competition track), not a modifier on
the regular rank. Third-party mirror data had silently stripped the `P` flag on 715 rows, making
SC/ST/PwD cutoffs at IITs look roughly 100× better than reality. Caught by cross-checking the official
archive against the mirror corpus and finding the disagreement.

**30. Where does the model fail today?**
Sparse series (brand-new programs, thin PwD sub-pools), institute renames silently splitting one series'
history into two shorter, worse-forecast series, home-state quota accuracy for GFTIs (thinner curation
than IIT/NIT data), and it only forecasts the *final* round — round-to-round drift within a season is
unmodeled even though the raw all-rounds corpus exists to support it later.

### F. Data engineering & the scraper

**31. The source has no API — how did you get the data?**
Reverse-engineered the ASP.NET WebForms postback chain: `__VIEWSTATE`/`__EVENTVALIDATION` tokens echoed
back at each step of a cascading-dropdown session, one HTTP round-trip chain per (year, round, institute
type) partition.
   - *Follow-up: what specifically makes ASP.NET WebForms harder to scrape than a normal REST-backed page?*
     Every "request" is actually a full page postback carrying hidden state fields that must be captured
     from the previous response and replayed verbatim — you can't just hit an endpoint with query params;
     the session's server-side view state has to be faithfully round-tripped or the server rejects the
     next step.

**32. How do you make a 140-request scrape reliable against a flaky government site?**
Per-type manifest tracks completed partitions (resumable — a crash mid-run costs nothing already done),
exponential-backoff retries, sha256 recorded at fetch time and re-verified at build time, a politeness
delay between requests, and rounds discovered live from the year dropdown rather than hardcoded.

**33. Why commit the data to git instead of S3, or fetch it at deploy time?**
The data is immutable once a round publishes — cold, append-only history is the ideal shape for a
versioned, diffable, checksummed git artifact, and the worst possible shape for a live runtime dependency
on a slow, frequently-down site with no API. It's also only 3.6MB gzipped for six years — S3 would add a
moving part for no real benefit at this size.

**34. How do you know your scrape is actually correct?**
Cross-validated against an independently-sourced mirror corpus: 61,460 overlapping rows, zero rank
disagreements — and the diff process *found real defects* in the mirror source (a mislabeled round, the
stripped-P-flag bug in Q29), which is what a good validation process is supposed to surface, not just
confirm agreement.

**35. Two colleges renamed across years — why is that hard, and what's the fix?**
Forecast series identity is keyed on institute+program name; a rename silently splits one continuous
history into two short, individually-worse-forecast series. Fix: a curated alias crosswalk applied at
parse time, mapping historical names onto the canonical current name before series are built.

### G. Auth, security & correctness

**36. Why Google-only login?**
Many users are minors — no password custody, no password-reset flow, no credential-stuffing attack
surface to defend. Cognito Hosted UI keeps the entire OAuth handshake out of the app's own code.
   - *Follow-up: is native email/password actually gone, or just hidden?* Just hidden — `PASSWORD_LOGIN=false`
     toggles the UI; the Cognito native auth path still exists underneath, unused. Know this — claiming
     it's fully removed when it's a feature flag is the kind of detail an interviewer will probe.

**37. How is RBAC enforced — where could a client bypass it?**
`custom:role` claim in the verified JWT; hierarchy-aware middleware server-side (`superadmin ⊇ admin ⊇
student`); per-admin `permissions[]`/`custom:scopes` for finer gating. Frontend role-gating is UX only —
every admin route independently re-checks server-side, so hiding a button client-side is not a security
control.
   - *Follow-up: how do scope changes reach a session that's already logged in?* They land on the user's
     *next* token — silent refresh covers password/SRP sign-in, but the Google Hosted-UI implicit flow has
     no refresh token, so a "sign in again" banner is the actual mechanism (see Part 3 for why this bit the
     owner personally after Phase 11 deploy).

**38. A Razorpay webhook arrives twice, out of order, or forged — walk through each.**
Forged: HMAC signature verification against the shared webhook secret rejects it before any processing.
Duplicate: ledger idempotency key (Q21) makes a replay a no-op. Out of order: the booking state machine
only advances on legally-defined transitions; an event that doesn't match the current state is a no-op,
not a corruption.

**39. How do you prevent user enumeration at signup?**
Cognito's `preventUserExistenceErrors` setting plus identical response shapes regardless of whether the
email/account already existed — an attacker can't distinguish "wrong password" from "no such account."

**40. Where are your secrets?**
SSM Parameter Store (SecureString) for the Google OAuth client secret and the Calendar service-account
JSON — never in code, never in a committed `.env`. Per-Lambda IAM grants are least-privilege: each
function can only read/write its own service's table(s), nothing account-wide.
   - *Follow-up: SSM vs Secrets Manager — why SSM here?* SSM SecureString parameters are free; Secrets
     Manager charges per secret per month plus API calls. At this scale/budget (the whole point is
     near-$0 idle cost) SSM is the right call for values that don't need automatic rotation. The payments
     LLD in `architecture.md` actually documents Secrets Manager for payment secrets specifically — know
     that the project uses *both*, deliberately, for different sensitivity/rotation needs, not one
     blanket choice.

### H. Operations, cost & testing

**41. What's on your dashboard, and which single metric pages you first?**
API 5xx rate, Lambda errors/throttles, p95 latency, DynamoDB throttles — but the *leading* indicator, the
one that degrades before anything else turns red, is CDN cache hit ratio.

**42. How do you keep an AWS bill near zero, mechanically?**
No idle-cost services by default (no NAT gateway, no ElastiCache, WAF off outside hardening/production
posture, no provisioned concurrency off-season), on-demand billing everywhere it's an option, plus a $10
AWS Budget alarm as the tripwire that catches drift.
   - *Follow-up: name one AWS service in this stack that is NOT free at low usage, and what you did about
     it.* Cognito — free at low MAU but roughly $1k+/mo at 3-lakh MAU (per architecture.md §11.1); the
     documented mitigation is a planned migration to Firebase Auth or a self-managed Google-JWT path before
     hitting that threshold — not yet executed, current posture is "revisit before the MAU number gets
     close." Saying the honest current status beats claiming it's solved.

**43. How would you load-test the result-day spike before the season?**
k6/Artillery replaying the spike shape up to 5k rps on the predictor with a realistic write mix layered
in; watch SLO adherence, cache hit ratio, DynamoDB throttles, cold-start p99, and cost per 100k requests —
cost is a first-class metric here, not an afterthought, because the whole design is optimized for it.

**44. What do your tests cover, and what's deliberately untested?**
Pure packages (predictor, forecast/stats logic) are unit-tested directly. Services are integration-tested
against DynamoDB Local on isolated per-test tables. A "deployed e2e" pattern runs against the live API
with throwaway Cognito users, purged afterward. Deliberately untested: the scraper's HTML parsing beyond a
set of golden partitions — validated instead by checksums and cross-source agreement (Q34), which is a
better fit for "does this reflect a flaky external site correctly" than a snapshot test would be.

**45. You deploy a bad cutoff dataset at peak — recovery, step by step?**
Snapshot versions are immutable and never mutated in place, so recovery is: flip `activeVersion` back to
the prior good version, issue one CloudFront invalidation, done in seconds. The bad version stays around
(not deleted) for forensics. This is the entire payoff of the immutable-versioned-snapshot design — say
that explicitly, don't just describe the mechanics.

### I. Trade-offs to volunteer before they're asked

- **Different consistency models for different flows, on purpose.** Predictor reads are eventually
  consistent by design (cached, versioned snapshot) — that's fine because the underlying data doesn't
  change mid-round. Payments are strongly consistent (conditional writes, append-only ledger) — that's
  non-negotiable because money is involved. Naming this contrast unprompted is a strong signal you
  understand CAP trade-offs aren't a single dial for a whole system.
- **The in-memory snapshot is a scale ceiling, not a permanent design.** At ~10–100× more rows it stops
  working cleanly; the documented escape hatch is Redis/DAX (reversing ADR-008) or sharding the snapshot.
- **Single region is an accepted risk**, appropriate for a domestic, seasonal product — the mitigation is
  "CDN + fast rebuild," not active-active multi-region, and that's a deliberate cost/complexity call, not
  an oversight.
- **`chanceBasis` duality** — shipping a statistically calibrated number and a heuristic fallback number
  side by side, honestly labeled, rather than pretending one method covers every case. This is a good
  example of "I chose to expose complexity to the API consumer rather than hide it dishonestly."

---

## Part 2 — Generic AWS depth (bridge every answer back to a project decision)

These aren't in the README because they're not project-specific claims — but "AWS-heavy" interviews test
whether you understand the primitives, not just that you can name-drop services. For each, know the
mechanism, then have one sentence connecting it to a real choice made in this project.

**1. DynamoDB: how does partitioning actually work, and what's a hot partition mechanically?**
Each table is split into partitions by hashing the partition key; a partition has a physical throughput
ceiling. A "hot partition" is many requests hashing to the same key concentrating load past that ceiling,
even if the table's *aggregate* provisioned/on-demand capacity has headroom. Adaptive capacity
auto-mitigates *some* skew, but a single key that's simply too hot (like the old single-item cutoff
approach would've been) it can't fully absorb. → *Project link:* this is exactly why the cutoff snapshot
is read once per cold start into memory rather than per-request — avoiding turning `CUTOFF#<version>` into
a hot key at 50k concurrent readers.

**2. On-demand vs provisioned DynamoDB capacity — when does provisioned actually win?**
On-demand scales automatically and has no idle floor, at a per-request price premium versus provisioned.
Provisioned (with auto-scaling) is cheaper at high, steady, predictable volume where you can right-size a
floor. → *Project link:* seasonal+spiky load is the textbook case *for* on-demand — a provisioned floor
sized for peak would sit mostly idle 10 months a year; sized for idle, it'd throttle at the spike.

**3. GSI vs LSI — what's the real difference, and why did this project only use GSIs?**
An LSI shares the base table's partition key and must be created at table-creation time, capped at 10GB
per partition key value. A GSI has its own independent partition+sort key, created any time, its own
provisioned/on-demand throughput. GSIs are far more flexible for access patterns discovered after initial
design — which is normal in an evolving product — so LSIs are rarely the right default choice.

**4. DynamoDB Streams vs Kinesis Data Streams for capturing table changes — why Streams here?**
DynamoDB Streams is purpose-built, free-ish (pay only for GetRecords calls), directly tied to table item
changes with before/after images. Kinesis Data Streams needs more setup and is better when you need
longer retention, fan-out to many independent consumers, or non-DynamoDB event sources feeding the same
pipeline. → *Project link:* Streams → EventBridge → S3/Athena is the whole analytics lake; no Kinesis
needed because there's exactly one source (DynamoDB) and the consumer count is small.

**5. TransactWriteItems — what does it actually guarantee, and what are its limits?**
All-or-nothing atomic writes across up to 100 items / 4MB, within a single region, single account. It's
NOT a general distributed transaction across services — it can't span two different AWS accounts or
regions. → *Project link:* useful for e.g. "hold a booking slot AND create the booking row" atomically;
the booking↔payment saga across services still has to be an eventually-consistent EventBridge flow because
that spans two different Lambdas/tables, which a single transaction can't do anyway.

**6. Lambda execution environment lifecycle — what actually happens on a cold start vs warm invocation?**
Cold: AWS provisions a new microVM/environment, downloads the code package, runs any module-scope
(top-level) code once — this is exactly where the "init phase" duration comes from — then executes the
handler. Warm: the same environment is reused for a subsequent invocation; module-scope state (like a
cached DB connection or, here, the cutoff snapshot) persists across invocations on that environment for
free. → *Project link:* this IS the mechanism behind ADR-008 — the snapshot cache isn't a special Lambda
feature, it's just a normal module-scope variable exploiting environment reuse.

**7. What actually determines how many concurrent Lambda execution environments you get?**
Account-level concurrency limit (default 1,000, raisable via support ticket) shared across all functions
in the account unless you set reserved concurrency to carve out a guaranteed slice for a specific
function, or set it low to protect a downstream from being overwhelmed. Provisioned concurrency
pre-initializes a set number of environments so those specific invocations skip the cold-start path
entirely. → *Project link:* the plan is pre-season limit increase + reserved concurrency on `predictor`/
`auth`/`planner` so a runaway or noisy-neighbor function elsewhere in the account can't starve the hot
path.

**8. VPC-attached Lambda cold starts — why does this project deliberately avoid a VPC on the hot path?**
A VPC-attached Lambda needs an ENI (elastic network interface) to reach VPC resources; ENI attach/detach
used to add real cold-start latency (this has improved significantly with Hyperplane ENIs, but it's still
one more moving part and one more thing that can fail). This project's Lambdas talk to DynamoDB, S3,
EventBridge — all reachable over the public AWS API without a VPC — so there's simply no reason to pay
that complexity tax. A VPC would only be needed for something like RDS in a private subnet, which this
design deliberately avoids on the hot path anyway (§2.4 — Aurora is analytics-only, off the hot path).

**9. API Gateway HTTP API vs REST API — what do you actually give up?**
HTTP API is ~70% cheaper per million requests and lower latency, but has a smaller feature set: no
request/response transformation templates (VTL), no usage plans/API keys, fewer authorizer types (though
JWT authorizers — exactly what's needed here — are supported natively). REST API is the fuller-featured,
pricier option. → *Project link:* JWT authorization is the only auth need on this API, so HTTP API's
feature set is a complete match, not a compromise.

**10. How does a Cognito JWT authorizer on API Gateway actually validate a token, mechanically?**
API Gateway fetches and caches the user pool's JWKS (public signing keys) from a well-known Cognito URL,
verifies the JWT signature against the matching key (by `kid` in the token header), checks standard claims
(expiry, issuer, audience), and — critically — never calls Cognito synchronously per request once JWKS is
cached; verification is local/cryptographic. → *Project link:* this is why role/scope checks read claims
already embedded in the verified token (`custom:role`, `custom:scopes`) rather than making a second lookup
call per request — the token IS the source of truth for the request's lifetime, which is also exactly why
scope changes only take effect on the *next* token (Q37 follow-up).

**11. Cognito User Pools vs Identity Pools — which does this project use, and why does that distinction matter?**
User Pools handle authentication (who are you) and issue JWTs. Identity Pools (Federated Identities)
exchange a User Pool token for *temporary AWS credentials* to call AWS services directly from a client.
This project only needs the former — the frontend never calls AWS services directly, it calls this
project's own API, which itself holds IAM permissions server-side. Identity Pools would only matter if,
say, the browser needed to PUT directly to S3 with scoped temporary creds — which for document uploads
this project does via *presigned URLs* minted server-side instead (see Part 3), not Identity-Pool
federated credentials.

**12. EventBridge vs SQS vs SNS — when is each the right primitive?**
SNS: pub/sub fan-out, push-based, no built-in retry/backoff/ordering story of its own (though it can
target SQS for that). SQS: a durable queue, pull-based, gives you visibility timeouts, DLQs, and
at-least-once delivery with per-message retry. EventBridge: a schema-aware event *bus* with content-based
routing rules — good for "many different event types, route each to the right consumer(s) by attribute,"
which SNS topics can approximate but less natively. → *Project link:* EventBridge is the router (rules
decide which SQS queue a `booking.confirmed` vs `mentor.approved` event lands in), SQS is the buffer that
protects each consumer from bursty producers, and the DLQ on each queue is what turns a silent failure
into a visible, replayable one.

**13. SQS visibility timeout — what is it, and what breaks if you set it wrong?**
When a consumer receives a message, it becomes invisible to other consumers for the visibility timeout
duration while being processed; if the consumer crashes or is too slow and doesn't delete the message in
time, it becomes visible again and gets redelivered. Set too short: a slow-but-successful consumer gets
its own work redelivered to another consumer — duplicate processing. Set too long: a genuinely failed
message sits invisible (effectively lost) for a long time before retry. → the reason consumers here need
to be idempotent regardless (a redelivered `payment.captured` must be a safe no-op via the ledger's
provider-event-id key), because visibility-timeout tuning alone never *guarantees* exactly-once.

**14. CloudFront cache behaviors and invalidation cost — why can't you just invalidate constantly?**
Cache behaviors let you route by path pattern to different origins/TTL policies. Invalidations are billed
per path pattern past a free monthly allowance, and — more importantly for a spiky product — an
invalidation storm right at peak is itself a self-inflicted cache-miss storm. → *Project link:* invalidation
is deliberately rare (only on `cutoffs.publish`, a handful of times a season), not per-write, which is the
whole reason the versioned-snapshot design exists instead of invalidating per data change.

**15. S3 presigned URLs — what do they actually authorize, and what's the security model?**
A presigned URL is a time-limited, cryptographically signed URL granting the bearer of the URL the
specific action (GET/PUT) the signer's IAM identity was allowed to perform, for that specific key, until
expiry. The server signs it using its own credentials — the client never sees any AWS credentials at all.
→ *Project link:* mentor document uploads use short-lived presigned PUTs (5 min) with a *server-minted* S3
key (so a client can never choose or guess another user's key), and admin document previews use a 3-min
presigned GET, audited per access. This is the correct pattern versus, e.g., handing the browser real IAM
credentials via an Identity Pool (Q11) — smaller blast radius, no credential exposure at all.

**16. S3 storage classes and lifecycle rules — how would this project use them?**
Standard for actively-served objects; lifecycle transitions (to Infrequent Access / Glacier) or expiry
rules for objects with a known decay pattern. → *Project link:* the mentor-docs bucket's lifecycle rule
expires `rejected`-tagged documents after 90 days and non-current object versions after 30 days —
compliance-driven data minimization for sensitive documents of (often) minors, not a cost optimization
primarily, though it happens to also save storage cost.

**17. Lambda SnapStart — relevant here or not?**
SnapStart (originally Java-only, later extended) snapshots an initialized execution environment so cold
starts resume from a saved snapshot instead of re-running init from scratch. Worth knowing current runtime
coverage before the interview — AWS has been expanding this — since a live "have you considered SnapStart
for your Node cold starts" question deserves a current, not stale, answer. Don't guess; check AWS's
current docs if this comes up and you're unsure.

**18. CDK — what's the difference between `synth` and `deploy`, and what is CloudFormation drift?**
`cdk synth` compiles the TypeScript app into a CloudFormation template (pure, no AWS calls, reviewable/
diffable like the README claims). `cdk deploy` submits that template as a CloudFormation stack update.
Drift is when real infrastructure state diverges from what the last-deployed template says (someone
changed something out-of-band, e.g. via console) — CloudFormation can detect but not auto-heal drift.
→ *Project link:* this is exactly the discipline behind "stacks are diffable/reviewable" — a `cdk diff`
before every deploy is what makes an unreviewed accidental change visible before it ships.

**19. IAM policy evaluation — walk through how AWS decides allow vs deny for a request.**
Default deny. Collect all applicable policies (identity-based on the principal, resource-based on the
target, any permission boundaries, SCPs if in an AWS Organization). An explicit `Deny` anywhere always
wins. Otherwise, at least one explicit `Allow` across all applicable policies is required, or the request
is denied by default. → *Project link:* "least privilege, per Lambda" means each function's execution
role's identity policy only allows the actions/resources it actually needs (its own table's ARN, not
`dynamodb:*` on `*`) — the default-deny baseline is what makes an *omitted* grant a safe failure mode
(function can't do the thing) rather than an accidental over-grant.

**20. AWS Well-Architected Framework — the six pillars, and where does this project make its most visible trade-off?**
Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability.
The most visible, deliberate trade-off here is Cost Optimization pulling hard against Reliability/Performance
headroom — e.g. no Redis, no multi-region, WAF off outside hardening — all conscious, documented,
reversible choices given the product's actual usage shape, not oversights. A good interview answer names
the pillar you *deprioritized* and why, not just the ones you satisfied.

---

## Part 3 — Real incidents (your best "tell me about a bug" material)

These actually happened building and deploying this system — pulled from the project's own changelog/ADRs
and deployment history. Real, specific, and yours; use them whenever asked "tell me about a hard bug" or
"tell me about a production incident" instead of a generic made-up story.

**The Amplify custom-domain vs CloudFront alias fight (~2 days lost).**
Attaching a custom domain (`app.kodexa.in`) to a new Amplify deployment kept failing with "aliases …
points to another CloudFront distribution." Root cause: CloudFront refuses to attach an alias to
distribution X while DNS still resolves that hostname to a *different*, stale distribution Y (left over
from an earlier torn-down Amplify app) — and Amplify only attempts the association **once**
(`SDK Attempt Count: 1`), then freezes; it does not auto-retry once the old distribution releases the
alias. Compounding trap: every delete+recreate of the domain association mints a **new** target
CloudFront distribution, so the DNS CNAME value has to be re-pointed each time — while the cert-validation
CNAME stays stable. The fix was to stop churning delete/recreate (which resets the race every time),
point DNS at the current association's target, wait ~30–90 minutes for the old distribution to actually
release the alias, then nudge a retry via `update-domain-association` with the same settings.
→ *What this demonstrates:* diagnosing an asynchronous, eventually-consistent AWS resource-ownership
conflict from an opaque error message, and — importantly — recognizing that the "obvious" fix (delete and
recreate again) was actively making it worse, not better.

**HTTP API routes silently land in the wrong CDK stack.**
Service stacks add routes via `httpApi.addRoutes(...)`, but the `HttpApi` construct itself is owned by the
foundation stack — so the actual `AWS::ApiGatewayV2::Route` CloudFormation resources synthesize into the
**foundation** stack, not the service stack that logically owns the route. Forgetting this means deploying
only the service stack after adding a new endpoint silently does nothing — the route never gets created,
and the failure mode is "404, looks like the deploy didn't work" with no obvious link back to "you deployed
the wrong stack." Fix: always redeploy `foundation` alongside the owning service stack after adding routes.
→ *What this demonstrates:* real understanding of how CDK constructs can straddle stack boundaries in
non-obvious ways, and a habit of documenting infra gotchas so they don't cost the same debugging time
twice.

**Google implicit-flow sessions can't silently pick up new permission scopes.**
After deploying Phase 11 (adding admin permission scopes carried in the JWT as `custom:scopes`), the
owner's own already-logged-in session didn't gain the new `superadmin` scopes automatically. Root cause:
scope changes only take effect on a *fresh* token, and Cognito's Google Hosted-UI flow used here is the
OAuth **implicit** grant, which issues no refresh token — so there's no silent background renewal path the
way there is for password/SRP sign-in. The only fix is an explicit sign-out/sign-in. This is now documented
UX ("sign in again" banner) rather than a bug to fix at the auth-flow level, since implicit flow was a
deliberate simplicity trade-off, not an oversight.
→ *What this demonstrates:* understanding OAuth grant-type mechanics well enough to explain *why* a fix
had to be a UX affordance rather than a backend change — the constraint is inherent to the chosen flow.

**DynamoDB Local port collision with an unrelated local project.**
Local integration tests run against DynamoDB Local on port 8000 — but another unrelated local project on
the same machine also binds 8000, so after that project runs, DynamoDB Local's container fails to bind and
tests fail with a connection error that looks like a test/infra bug, not a port conflict. Fix is
operational: recreate the DynamoDB Local container after noticing a failed bind, rather than debugging the
tests themselves.
→ *What this demonstrates:* a small but real example of correctly triaging "is this my code or my
environment" — the fastest debugging skill, and one that's easy to skip in a rush.

**A reseed is a two-step, order-dependent operation, not a single write.**
Publishing a new catalog dataset version is: write the new-version rows to DynamoDB, **then** atomically
flip the `CONFIG/ACTIVE` pointer — in that order, deliberately, so a reader can never observe a
half-written version as active. The Lambda's soft TTL (~5 minutes) means a flip doesn't require a redeploy
to take effect, but it does mean there's up to a 5-minute window where warm Lambdas serve the stale
version even after the flip — a real, accepted staleness bound, not a bug, and one worth being able to
quote exactly if asked "how fresh is fresh."

**"Deployed e2e" needed its own disposable-user pattern.**
Testing against the *live* deployed API (not a local mock) required real Cognito users, but you can't use
production-adjacent sign-up flows without leaving junk data behind. The pattern that emerged: script a
throwaway sign-up + `USER_PASSWORD_AUTH` login (with a specific gotcha — the AWS CLI's attribute shorthand
silently splits comma-containing values, so `--user-attributes` has to be passed as JSON, not the shorthand
form, to avoid corrupting an attribute value), run the e2e assertions, then purge every row/object the test
created. → *What this demonstrates:* the difference between "tests pass in CI against a mock" and "we
verified the actual deployed system," plus a specific, memorable AWS CLI footgun (shorthand attribute
parsing) that's genuinely easy to hit and rarely documented anywhere obvious.

---

## How to use this before the interview

1. Re-read Part 1 section headers only, and for each, say the one-sentence answer out loud from memory —
   if you blank, that's the one to actually re-read in full.
2. Pick 2–3 Part 3 incidents and be ready to tell them as a 60–90 second story: situation → root cause →
   fix → what you'd do differently. These are your strongest material because they're specific and true.
3. For Part 2, you don't need to lecture the mechanism unprompted — but if asked "how does X work" and you
   only know the project's *use* of X, that's the gap this section closes.
4. Don't overclaim. Several answers above explicitly flag "verify current state before claiming this is
   live" — `progress.md` is the source of truth for what's actually built vs. designed; an interviewer
   who catches daylight between your claim and your own repo will trust everything else you said less.
