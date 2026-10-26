# Company Career Page Scraper: every open job from any company, by name or domain

Give it company names or websites (`stripe.com`, `Marriott`, `Casey's`) and get every
open job on each company's careers page as one clean row, whichever hiring system the page
runs on: Workday, Oracle, ADP, iCIMS, SAP SuccessFactors, Eightfold, Phenom, Greenhouse,
Lever, Ashby and 14 more. No board tokens to look up, no API key for any of those systems,
no proxy.

This repo is the quick start for the hosted
[**Company Career Page Jobs Scraper**](https://apify.com/glitchbound/ats-jobs-scraper) on
Apify: one HTTP call from the shell, a standard-library Python script, and the setup for
using it as a tool in Claude, ChatGPT or Cursor.

- [Quick start: one curl](#quick-start-one-curl)
- [Quick start: Python](#quick-start-python)
- [Use it from an AI agent (MCP)](#use-it-from-an-ai-agent-mcp)
- [Input](#input) · [Output](#output)
- [How a name becomes a board](#how-a-name-becomes-a-board)
- [The systems it reads](#the-systems-it-reads)
- [FAQ](#faq)

## Quick start: one curl

Get a free Apify API token at
[console.apify.com/settings/integrations](https://console.apify.com/settings/integrations),
then:

```bash
export APIFY_TOKEN=apify_api_...
curl -s -X POST \
  -H "Authorization: Bearer $APIFY_TOKEN" -H "Content-Type: application/json" \
  -d '{"companies": ["stripe.com", "Marriott", "Casey'"'"'s"], "titleKeywords": ["manager"], "maxItems": 20}' \
  "https://api.apify.com/v2/acts/glitchbound~ats-jobs-scraper/run-sync-get-dataset-items"
```

The response is a JSON array, one object per job. Stripe is on Greenhouse, Marriott on
Oracle, Casey's on ADP; you did not need to know any of that. For runs longer than five
minutes, start the run and fetch the dataset when it finishes.

## Quick start: Python

[`example.py`](example.py) uses only the standard library and writes a CSV:

```bash
python3 example.py --company stripe.com --company Marriott --company "Casey's" --title manager --max 6
python3 example.py inputs/security-roles.json
```

A real run of the first command:

```
6 jobs written to jobs.csv
  Stripe: AEO and GEO Marketing Manager (Remote in the US)
  Stripe: AI Solutions Program Manager, Finance (SEA, SF, NYC, CHI)
  Marriott: Revenue Manager (LONDON, United Kingdom)
  Marriott: Analytics Manager (Goa, India)
  Casey's: Center Store Manager - Gardner, KS (Gardner, Kansas, United States)
```

## Use it from an AI agent (MCP)

Add this as a connector in Claude or ChatGPT:

```
https://mcp.apify.com?tools=glitchbound/ats-jobs-scraper
```

In Cursor or VS Code, the same address goes in `mcp.json`:

```json
{ "mcpServers": { "apify": { "url": "https://mcp.apify.com?tools=glitchbound/ats-jobs-scraper" } } }
```

The first call asks you to sign in to Apify or takes an API token. Then "what is Stripe
hiring for in Dublin?" is one tool call.

It is also listed in the official MCP Registry as
[`io.github.danielhagever/career-site-jobs`](https://registry.modelcontextprotocol.io/v0.1/servers/io.github.danielhagever%2Fcareer-site-jobs/versions/latest),
together with the Workday and Job Listings tools ([`server.json`](server.json)).

## Input

| Field | What it does |
|---|---|
| `companies` | Company names or domains: `NVIDIA`, `Home Depot`, `stripe.com`. Each is looked up in a directory of career sites already read, and guessed (then verified against the live board) only when the directory has nothing. |
| `boards` | Skip discovery and read a board directly as `platform:token`, e.g. `greenhouse:stripe` or `lever:palantir`. Workday takes its careers URL: `workday:https://nvidia.wd5.myworkdayjobs.com/NVIDIAExternalCareerSite`. |
| `titleKeywords` | Keep roles whose title contains any of these. Plain text, so `c++` means c++. |
| `locationKeywords` | Keep roles whose location contains any of these, e.g. `London`. |
| `remoteOnly` | Keep remote roles only. |
| `includeDescription` | Also fetch the description text. Off by default, because some sources are many times heavier with it. |
| `maxResultsPerCompany` | Ceiling per board (default 1,000). |
| `maxItems` | Ceiling for the whole run. |

Ready-made inputs are in [`inputs/`](inputs). Rows your filters drop are not charged, and
on large Workday, Oracle, Eightfold, Phenom and Amazon boards a title or city filter is sent
to the employer's own search, so it finds every match rather than skimming the newest
postings.

## Output

Every system names things differently; every row comes back in one shape. One real row
from each of the three systems in the run above is in
[`sample-output.json`](sample-output.json):

```json
{
  "jobId": "5001234068100",
  "title": "Center Store Manager - Gardner, KS",
  "location": "Gardner, Kansas, United States",
  "departments": ["Store / Field Leadership"],
  "url": "https://myjobs.adp.com/caseysstore/cx/job-details?reqId=5001234068100",
  "postedAt": "2026-10-25T13:02:33Z",
  "employmentType": "full-time",
  "company": "Casey's",
  "atsPlatform": "adp",
  "boardToken": "caseysstore",
  "boardUrl": "https://myjobs.adp.com/caseysstore",
  "workArrangement": "onsite",
  "isRemote": false,
  "country": "United States",
  "region": "Kansas",
  "city": "Gardner"
}
```

`atsPlatform` says which system each row came from. Error rows (`"recordType": "error"`)
say why a company returned nothing and are not charged.

## How a name becomes a board

Most scrapers that take a company name can only guess at a board token, which finds
startups on Greenhouse or Lever and misses the employers people ask about first: Workday,
Oracle, ADP and iCIMS addresses carry ids that no name or domain can be turned into.

So the Actor ships a directory of over 75,000 company names, each mapped to a career site
it has already read real postings from, refreshed nightly. A name is looked up there first.
Only when the directory has nothing does it derive candidate tokens from the domain, and
then it checks each one against the live board: on the systems where a board that does not
exist answers 404, a 200 is proof, not a guess.

Some companies run careers software of their own (Google, Apple, Meta, Tesla). For those
the run says so and charges nothing, instead of returning some other company's board that
happens to share the name.

## The systems it reads

| System | Examples of employers it answers by name |
|---|---|
| Workday | NVIDIA, Walmart, Salesforce, Disney, Wells Fargo |
| Oracle Recruiting Cloud | Marriott, Kroger, JPMorgan Chase, Hilton, Uber |
| ADP | Casey's, PepsiCo Beverages North America |
| iCIMS | Charles Schwab, Liberty Mutual, EMCOR |
| SAP SuccessFactors (on the employer's own site) | EY, Compass Group, Aramark, Union Pacific |
| Eightfold | Microsoft, Starbucks, Lockheed Martin, Qualcomm |
| Phenom | UPS, RTX, Eli Lilly, United Airlines |
| Radancy | UnitedHealth Group, Comcast, Kaiser Permanente |
| Jibe | Costco, H&R Block, PetSmart, AMD |
| BrassRing | Home Depot, Best Buy, Publix, Edward Jones |
| Taleo | Valero, Textron |
| Amazon (its own search) | Amazon, AWS |
| Greenhouse, Lever, Ashby | Stripe, Airbnb, Palantir, OpenAI, Notion |
| Workable, Recruitee, Rippling, Teamtailor, Personio, BambooHR, Breezy, JazzHR, Paylocity | the long tail of small and mid-size employers, many in Europe |

The raw public endpoint of each of these, with a tested `curl`, is in
[**job-board-apis**](https://github.com/danielhagever/job-board-apis).

## FAQ

**How do I get every open job from a company's career page?**
Put its name or website in `companies`. The Actor works out which system the careers page
runs on and reads that system's own job feed, so you never need the board's URL or token.

**Does it work for the biggest employers?**
Yes, that is what the directory is for. Each board gives up to 1,000 roles by default; raise
`maxResultsPerCompany` and a 20,000-job board such as Costco's is read in full. A title or
city filter goes to the employer's own search, so it finds every match either way.

**Why not just scrape Indeed or Glassdoor?**
They have no public jobs API and block automated requests at the edge. They also aggregate
from these same career pages, so the source is fresher.

**What does it cost?**
It is paid per result; rows dropped by your filters and error rows are not charged. The
current rate for your plan is on the [Actor page](https://apify.com/glitchbound/ats-jobs-scraper).

**I want what is new, not everything.**
[Job Listings API](https://apify.com/glitchbound/ats-jobs-search) searches the whole index
by title, place or salary, with `newWithin` for only the roles that appeared since the last
daily refresh.

## Related

- [Workday Jobs Scraper](https://apify.com/glitchbound/workday-jobs-scraper) and its quick start, [workday-jobs-scraper](https://github.com/danielhagever/workday-jobs-scraper).
- [Oracle Recruiting Jobs Scraper](https://apify.com/glitchbound/oracle-jobs-scraper): Oracle Cloud careers sites by name.
- [iCIMS Jobs Scraper](https://apify.com/glitchbound/icims-jobs-scraper): iCIMS career portals by name.
- [Job Listings API](https://apify.com/glitchbound/ats-jobs-search): one search across every company in the index; quick start in [job-listings-api](https://github.com/danielhagever/job-listings-api).
- Write-up: [Company Career Page Scraper: every open job from any company, by name or domain](https://dev.to/glitchbound/company-career-page-scraper-every-open-job-from-any-company-by-name-or-domain-ede).

Questions, or a company that does not resolve? Open an issue here or on the Actor page.

## License

MIT
