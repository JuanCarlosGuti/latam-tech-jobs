# LATAM-eligible tech jobs — open daily dataset

A free, daily-refreshed dataset of open tech roles that a candidate **based in
Latin America** could actually take, normalized into one schema across six
applicant tracking systems.

No token. No signup. No login. Two URLs:

```bash
# the postings
curl https://api.apify.com/v2/key-value-stores/crearcode~latam-tech-jobs/records/latest

# run metadata + per-source health
curl https://api.apify.com/v2/key-value-stores/crearcode~latam-tech-jobs/records/meta
```

Refreshed every day at 07:00 UTC. Typically **800-900 postings from ~46 job
boards**. Around 70 companies are queried; on a given day roughly 30 of them
have LATAM-eligible roles open — see `/records/meta` for the live count.

### Daily history

`latest` is overwritten on every run, so each day is also archived under its own
key. Job-market history cannot be collected retroactively — if it is not captured
on the day, it is gone.

```bash
# what days exist (first, last, count, and the full list)
curl https://api.apify.com/v2/key-value-stores/crearcode~latam-tech-jobs/records/history-index

# one specific day, same schema as /latest
curl https://api.apify.com/v2/key-value-stores/crearcode~latam-tech-jobs/records/history-2026-08-19
```

Keys are `history-YYYY-MM-DD` (UTC). Read `history-index` first rather than
guessing dates — a day the run failed simply will not be there.

## Why this exists

Every major ATS publishes its job board as a public JSON endpoint. But each one
names things differently, buries compensation in a different place, and
describes location in a dozen incompatible ways. Aggregators either sit behind a
login or hand you the raw payload and leave the normalization to you.

This is the normalized output, published for free.

The specific angle: **`latamFriendly`** — whether a candidate based in Latin
America could realistically take the role. That means LATAM-based positions plus
remote roles that don't restrict hiring to another region. It's a filter almost
nobody publishes, and it's the reason this dataset exists rather than being one
more job scrape.

## Schema

One object per posting.

| Field | Type | Notes |
|---|---|---|
| `id` | string | `{source}:{slug}:{nativeId}` — stable across runs, safe to upsert on |
| `source` | string | `greenhouse` \| `lever` \| `ashby` \| `workable` \| `smartrecruiters` \| `recruitee` \| `workday` \| `eightfold` |
| `companySlug` | string | the board identifier on that ATS — **the field to join on** |
| `companyName` | string | display name. See the caveat below |
| `title` | string | |
| `url` | string | public posting URL |
| `locationRaw` | string | as written by the employer, lightly normalized |
| `countryCode` | string | ISO 3166-1 alpha-2, inferred. `null` on 1% |
| `latamFriendly` | boolean | see above |
| `workMode` | string | `remote` \| `hybrid` \| `onsite` \| `unknown` |
| `department` | string | |
| `employmentType` | string | **`null` on 71%** — most boards don't set it |
| `postedAt` | string | ISO 8601 |
| `updatedAt` | string | ISO 8601. `null` on 30% |
| `salary` | object | **`null` on 91%.** Read the caveat, it matters |
| `language` | string | detected language of the description |
| `description` | string | plain text, truncated to 600 chars in this dataset |
| `scrapedAt` | string | ISO 8601 capture timestamp |

### `salary`

```json
{
  "min": 120000,
  "max": 160000,
  "currency": "USD",
  "period": "year",
  "annualizedMin": 120000,
  "annualizedMax": 160000,
  "rawText": "$120,000 - $160,000",
  "confidence": "parsed"
}
```

`annualizedMin` / `annualizedMax` are the comparable numbers: an hourly US rate
and a monthly Colombian salary both end up as annual figures in the same column,
so they sort against each other. `rawText` is kept so you can audit the parse.

## Caveats — read these before you trust a column

**1. Only ~6% of postings have a salary, and that is a fact about the market,
not about the parser.**

Measured on the 2026-08-19 snapshot (855 postings):

| ATS | Share of rows | Rows with salary |
|---|---|---|
| Greenhouse | 45% | 4.4% |
| Eightfold | 35% | 0.0% |
| Ashby | 14% | 15.6% |
| Lever | 5% | 37.5% |
| Workable | 1% | 20.0% |

US pay-transparency laws force employers to publish ranges. Most Latin American
postings simply don't include one, and a parser cannot extract a number that was
never written. The two sources carrying 80% of the volume are the two that
almost never have a number in them.

**2. `companyName` is authoritative for some sources and derived for others.**

Greenhouse, Workable, SmartRecruiters and Recruitee return the employer's
display name in their API. **Ashby, Lever, Workday and Eightfold do not expose it anywhere**
— not at the top level, not on individual postings. For those four the name is
derived from the board slug (`nubank` → `Nubank`), so casing on multi-word
brands will sometimes be wrong.

If you need an exact identifier, use `companySlug`. It's always the raw ATS
value.

**3. `countryCode` and `workMode` are inferred, not declared.**

No ATS has a reliable structured field for either. They're derived from location
strings and description text. They're good, not perfect.

**4. This is a curated sample, not the whole market.**

It runs against ~70 companies known to hire in or from Latin America. It is not
an exhaustive index of every open role in the region, and it never claims to be.

## Per-source health

Every run writes a status block, so a quiet week and a broken API don't look the
same:

```json
{
  "generatedAt": "2026-08-18T23:02:22.035Z",
  "postings": 566,
  "boards": 44,
  "sources": [
    { "ats": "greenhouse", "status": "ok", "boards": 20, "boardsOk": 20,
      "jobsFound": 2817, "jobsEmitted": 373, "errors": [] }
  ]
}
```

`status` is `ok`, `partial` or `failed`. `jobsFound` vs `jobsEmitted` separates
"the source is broken" from "the filters are aggressive."

## Where the data comes from

All source endpoints are public and answer without a session, a cookie, or an
account:

| ATS | Endpoint |
|---|---|
| Greenhouse | `boards-api.greenhouse.io/v1/boards/{slug}/jobs` |
| Lever | `api.lever.co/v0/postings/{slug}` |
| Ashby | `api.ashbyhq.com/posting-api/job-board/{slug}` |
| Workable | `apply.workable.com/api/v1/widget/accounts/{slug}` |
| SmartRecruiters | `api.smartrecruiters.com/v1/companies/{slug}/postings` |
| Recruitee | `{slug}.recruitee.com/api/offers/` |
| Workday | `POST {host}/wday/cxs/{tenant}/{site}/jobs` |
| Eightfold | `{tenant}.eightfold.ai/api/pcsx/search?domain={domain}` |

Nothing here requires logging in, and nothing that requires logging in will ever
be added. No personal data is collected or published — job postings only.

## Notes for anyone building the same thing

A few things that cost me time and aren't documented anywhere:

- **Workday silently caps `limit` at 20.** Ask for 100 and you get `200 OK` with
  an empty array and no error — which reads exactly like "this company has no
  openings."
- **Workday stops paginating at 10,000** on large tenants, with no flag saying so.
- **There is no public directory of Workday tenants.** Host, tenant and site only
  come from the careers URL, which is why Workday is excluded from automatic
  discovery here.
- **Ashby hides compensation behind a query parameter.** Without
  `?includeCompensation=true` the field is absent entirely — not `null`, absent.
- **Eightfold’s obvious endpoint lies about being closed.** `/api/apply/v2/jobs`
  answers `403 Not authorized for PCSX`; the one the career site itself calls,
  `/api/pcsx/search`, is fully open. And its `num` silently caps at 10. Not every
  tenant allows it — some answer 403 to any non-browser request.
- **A company's domain is not its ATS slug.** `datadoghq.com` is `datadog` on
  Greenhouse; `joinhandshake.com` is `handshake` on Ashby.

## License

The data is aggregated from public job boards and published as-is, with no
warranty. Postings belong to their respective employers. Use it, build on it,
tell me what's broken.

---

Generated by an [Apify actor](https://apify.com/crearcode/greenhouse-lever-ashby-jobs-api)
I maintain. Issues and corrections welcome here.
