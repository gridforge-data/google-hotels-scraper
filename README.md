# Google Hotels Scraper: track hotel prices into a spreadsheet with Python

Comparing hotel prices gets tedious when you keep repeating the same search and copying rates into a spreadsheet. Tracking how those prices change is even harder unless you save the dates, guest count and time of each search.

This repo shows how to use [GridForge's Google Hotels Scraper](https://apify.com/gridforge/google-hotels-scraper) to collect a hotel-price snapshot and export it to CSV. You can start without code, then use Python when you want a repeatable workflow.

## What you get

Each result represents one hotel from the first page of the location search. Available fields include:

| Field | What it tells you |
|---|---|
| `name` | Hotel name |
| `pricePerNight` | Displayed nightly price before taxes |
| `pricePerNightWithTaxes` | Displayed nightly price including taxes |
| `rating` | Guest rating |
| `reviewCount` | Number of guest reviews |
| `hotelClass` | Hotel star classification |
| `latitude`, `longitude` | Property coordinates |
| `website` | Official website, when supplied |

Guest rating and hotel class measure different things. A four-star hotel does not necessarily have a guest rating of four. Some properties may have missing fields; keep those cells empty instead of treating a missing price as zero.

## Option 1: no code

1. Open the [Google Hotels Scraper](https://apify.com/gridforge/google-hotels-scraper) and click **Try for free**.
2. Enter a city, neighborhood or area, such as `Times Square, New York`.
3. Set check-in and check-out dates. Choose the number of adults and currency.
4. Click **Start**.
5. Open the resulting dataset, choose **Export**, and download CSV or Excel.

For your first search, use one location and one stay. A neighborhood query is useful when a whole-city search gives you properties spread too far apart.

## Option 2: Python

```bash
pip install apify-client
```

Set your Apify token in an environment variable named `APIFY_TOKEN`. This script requests one stay and writes only the columns needed for a price-comparison spreadsheet:

```python
import csv
import os
from datetime import datetime, timezone
from decimal import Decimal

from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

search = {
    "locations": ["Times Square, New York"],
    "checkIn": "2027-01-15",
    "checkOut": "2027-01-18",
    "adults": 2,
    "currency": "USD",
    "country": "us",
}

run = client.actor("gridforge/google-hotels-scraper").call(
    run_input=search,
    max_total_charge_usd=Decimal("0.05"),  # hard spending cap for this run
)

if not run or run.get("status") != "SUCCEEDED":
    raise RuntimeError("Hotel search did not succeed")

checked_at = datetime.now(timezone.utc).isoformat()
columns = ["checkedAt", "checkIn", "checkOut", "currency", "name",
           "pricePerNight", "pricePerNightWithTaxes", "rating", "reviewCount", "website"]

with open("hotel-prices.csv", "w", newline="", encoding="utf-8") as file:
    writer = csv.DictWriter(file, fieldnames=columns)
    writer.writeheader()
    for hotel in client.dataset(run["defaultDatasetId"]).iterate_items():
        if not hotel.get("name"):
            print("Check this record:", hotel)
            continue
        row = {key: hotel.get(key) for key in columns}
        row.update({"checkedAt": checked_at, "checkIn": search["checkIn"],
                    "checkOut": search["checkOut"], "currency": search["currency"]})
        writer.writerow(row)
```

Replace the example dates with your intended stay. The script records the requested dates and currency beside each result, so every snapshot is easy to interpret later.

## Use case: daily price tracking

Choosing a hotel for a three-night trip? Run the same search daily, keeping the stay dates, adult count, currency and location unchanged. Append each run to one spreadsheet and compare the tax-inclusive price for the same property across days. Once your first results look right, set up a daily **Schedule** in Apify so it runs on its own.

## Limits

- Returns the first page Google shows, typically around 18–20 hotels per location. Search by neighborhood for more coverage.
- Displayed rates change and may differ from the final price at checkout.
- Nearby searches can overlap; deduplicate before counting unique hotels.

**Cost:** $1 per 1,000 hotels, plus a tiny start fee. Set a spending cap and a run can never cost more than you allow.

*Not affiliated with or endorsed by Google.*
