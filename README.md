<p align="center"><a href="https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals"><img src="assets/banner.svg" alt="YouTube Shorts Sponsorship Signals Scraper — Python example" width="100%"></a></p>

# YouTube Shorts Scraper in Python: sponsorship scores per Short, straight into pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/youtube-shorts-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/youtube-shorts-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Here is a compact Python client for the **[YouTube Shorts Sponsorship Signals Scraper](https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals)** on Apify. You hand it channel handles, for instance `@GoPro`, and for each recent Short on those channels it brings back views, length, hashtags, a transcript flag and a sponsorship score between 0 and 1 together with the words that produced it. Paid placements in short vertical video are mostly spoken aloud, so the text a person would have to sit and listen to becomes a column you can filter. The collection work runs on Apify; your notebook or cron job only reads `results.json` and `results.csv`.

> **Actor:** [YouTube Shorts Sponsorship Signals Scraper](https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [youtube-shorts-scraper-javascript](https://github.com/devilscrapes/youtube-shorts-scraper-javascript)

## 🔥 What you get

- **Speech is read, not just captions**: for each Short the Actor pulls the spoken transcript (your `language` first, English second) and searches it together with the title and description, because a Short's description box is frequently blank.
- **A score you can recompute by hand**: strong phrases such as `use code` add 0.5, softer ones such as `link in bio` add 0.15, a disclosure hashtag like `#ad` adds 0.4 and each `@handle` or shop-style domain in the description adds 0.1. The sum is capped at 1.0 and 0.5 or higher sets `is_likely_sponsored`.
- **Evidence on every row**: `sponsorship_signals` holds the exact strings that matched, so a reviewer checks a verdict in seconds instead of rewatching the clip.
- **Brand tokens pulled out**: `detected_brands` collects description handles, domains and the word right after `sponsored by` or `code`; that last kind is taken from lowercased text, so expect `nordvpn` rather than `NordVPN`.
- **Silent Shorts still count**: a clip without captions yields a row with `has_transcript` false, scored only on its written text.
- **Filter before you pay**: Shorts under `minSponsorshipScore` are dropped inside the run and never charged.
- **Many handles in one run**: a channel that cannot be listed is logged and skipped while the others continue.

## 🎯 Use cases

### Measure how commercial a creator roster is

Feed in the handles an agency is considering, then `groupby('channel')` on `is_likely_sponsored`. The share of flagged Shorts per channel is a quick proxy for how often each creator already sells airtime, which matters when you negotiate exclusivity.

### See which brands keep returning

Explode `detected_brands` and count occurrences across channels. A token that shows up under five different creators in one month is a sponsor actively buying Shorts inventory in your niche.

### Disclosure audit for a compliance review

Select rows whose `sponsorship_signals` contain a spoken phrase but no hashtag such as `#ad`. That subset is the list a legal or brand-safety reviewer should watch first, with the URL ready to open.

### A weekly snapshot on a schedule

Run the script from cron every Monday, write each CSV under a dated name and concatenate them in a notebook. `scraped_at` plus `video_id` lets you follow view counts of the same Short across weeks.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/youtube-shorts-scraper-python.git
cd youtube-shorts-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [YouTube Shorts Sponsorship Signals Scraper](https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Strip away argument parsing and file output and this is all the code there is. The `apify-client` package launches the Actor, blocks until the run ends, then walks the dataset row by row.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/youtube-shorts-sponsorship-signals").call(run_input={'channels': ['@GoPro'], 'maxShortsPerChannel': 10, 'language': 'en', 'minSponsorshipScore': 0.0, 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`input.json` goes to the Actor unchanged. The bundled example checks the ten latest Shorts from GoPro's official brand channel, keeps every row whatever its score and runs without a proxy, which is the Actor's recommended default. Put several handles in `channels` to compare brands or creators side by side.

```json
{
  "channels": [
    "@GoPro"
  ],
  "maxShortsPerChannel": 10,
  "language": "en",
  "minSponsorshipScore": 0.0,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `channels` | array | yes | `["@mkbhd"]` | — |
| `maxShortsPerChannel` | integer | no | `20` | — |
| `language` | string | no | `"en"` | — |
| `minSponsorshipScore` | number | no | `0.0` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

One row per Short. Rows come back in the order they finish processing, so sort on `channel` or `view_count` yourself. When YouTube's metadata response arrives empty for a clip, its row still appears with a blank `title` and `null` counters; drop those with `df[df.title != '']` if they get in the way.

| Field | Type | Always present |
|---|---|---|
| `channel` | string | yes |
| `video_id` | string | yes |
| `url` | string | yes |
| `title` | string | yes |
| `published_text` | string or null | no |
| `view_count` | integer or null | no |
| `length_seconds` | integer or null | no |
| `description` | string | no |
| `tags` | array | no |
| `has_transcript` | boolean | yes |
| `transcript_chars` | integer | no |
| `hashtags` | array | no |
| `mentions` | array | no |
| `detected_brands` | array | no |
| `sponsorship_signals` | array | no |
| `sponsorship_score` | number | yes |
| `is_likely_sponsored` | boolean | yes |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.004 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

A per-channel sponsorship summary with pandas (`pip install pandas`). It computes the flagged share, the median views of flagged versus unflagged Shorts and the three most frequent brand tokens for every channel, then saves `channel_summary.csv`.

```python
import json

import pandas as pd

shorts = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
shorts = shorts[shorts["title"] != ""]

summary = shorts.groupby("channel").agg(
    shorts=("video_id", "count"),
    flagged_share=("is_likely_sponsored", "mean"),
    with_transcript=("has_transcript", "mean"),
)
views = shorts.pivot_table(index="channel", columns="is_likely_sponsored", values="view_count", aggfunc="median")
summary["median_views_flagged"] = views.get(True)
summary["median_views_other"] = views.get(False)

brands = shorts.explode("detected_brands").dropna(subset=["detected_brands"])
top = brands.groupby("channel")["detected_brands"].agg(lambda s: ", ".join(s.value_counts().index[:3]))
summary["top_brands"] = top

summary.round(2).to_csv("channel_summary.csv")
print(summary.round(2).to_string())
```

## ❓ FAQ

### Does a score of 1.0 prove the Short is an ad?

No. It means several commercial phrases or disclosure tags matched. A creator saying "check out my other video" earns weak points too. Treat the flag as a lead for a human, and read `sponsorship_signals` before you act on it.

### Why are there fewer rows than `maxShortsPerChannel`?

Three reasons are possible. The Actor works from the Short IDs present on the channel's Shorts tab when it first loads and does not scroll for more, `minSponsorshipScore` may have filtered some out, or the channel simply has fewer Shorts.

### What if the transcript is in another language?

Set `language` to the ISO code you expect. The Actor asks for that track, then for English. The phrase list is English, so non-English speech mainly contributes through hashtags, handles and domains.

### Should I turn the proxy on?

Start without it. YouTube tends to return empty metadata to shared datacenter addresses. If a large run starts producing blank rows, switch to an Apify Proxy residential group in `proxyConfiguration`.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [YouTube Shorts Sponsorship Signals Scraper on Apify Store](https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals)
- [JavaScript version of this example](https://github.com/devilscrapes/youtube-shorts-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [YouTube Channel Videos Scraper](https://apify.com/DevilScrapes/youtube-channel-scraper) — channel-level stats and long-form uploads for the same handles.
- [YouTube Comments Scraper](https://apify.com/DevilScrapes/youtube-comments-scraper) — how viewers react under a sponsored video.
- [Patreon Creator Scraper](https://apify.com/DevilScrapes/patreon-creator-scraper) — the creator income stream that sits beside brand deals.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/youtube-shorts-sponsorship-signals) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with YouTube or Google LLC.
