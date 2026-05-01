# Bollywood Song Scraper

Scrapes every Hindi-language film listed on Wikipedia (1931–present) and extracts structured soundtrack data — song titles, playback singers, and music directors — into a single CSV.

The output is `bollywood_full_history.csv` with columns: **Year, Movie, Song, Singer, Composer**.

## How it works

The scraper walks Wikipedia's yearly index pages (`List of Hindi films of 20XX`), follows each film's link, locates the soundtrack table, and pulls out per-song metadata. Nine decades of Wikipedia editors means nine decades of formatting variation, so the extraction pipeline has to be aggressive about normalization:

- **Rowspan/colspan resolution** — builds a full 2-D grid from each HTML table so column indices stay correct even when a music director cell spans an entire album.
- **Two-pass column detection** — claims lyricist, singer, composer, and duration columns first, then identifies the song title from whatever remains. This prevents lyrics columns from being misidentified as titles, which was a persistent early bug.
- **`<th scope="row">` awareness** — picks up song titles stored in header cells rather than data cells. Common pattern in modern soundtrack tables.
- **Three fallback strategies** — tries structured table extraction first, then bulleted lists under "Soundtrack" or "Songs" headings.

## Filtering

Not every link on a year page points to a film. The scraper uses infobox field detection and category heuristics to skip bios, plays, and disambiguation pages. Within film pages, it excludes navbox/sidebar/infobox tables, non-Hindi language sections, and discography/awards/controversy tables. Song tables are capped at 40 rows to avoid scraping filmography lists disguised as tracklists.

Cell-level cleanup strips citation brackets, a.k.a. suffixes, smart quotes, and concatenated composer–lyricist names (e.g. `Bappi LahiriAnjaan` → `Bappi Lahiri`).

## Performance

- Concurrent film-page fetching within each year — configurable thread pool, default 5 workers.
- Connection reuse via `requests.Session()`.
- Resume support — detects years already in the output CSV and skips them on restart.
- Tunable `SLEEP_PER_FILM` and `SLEEP_PER_YEAR` for polite crawling.

## Usage

```bash
pip install requests beautifulsoup4
python scraper.py
```

The scraper writes incrementally — you can kill it and restart without losing progress. Adjust `MAX_WORKERS`, `SLEEP_PER_FILM`, and `SLEEP_PER_YEAR` at the top of the file if you want to tune throughput vs. politeness.

## Known limitations

- Wikipedia coverage is sparse for pre-1940s films. Many early talkies don't have soundtrack tables, so they'll show up in the no-songs audit log but not in the main CSV.
- Song data quality depends entirely on what Wikipedia editors have entered. Some pages have full singer/composer metadata, others just have titles.
- The year-page table detection uses a scoring heuristic that can miss tables with unusual layouts. If a year returns zero films, check the page structure manually.

## Output format

| Column   | Description                                      |
|----------|--------------------------------------------------|
| Year     | Release year                                     |
| Movie    | Film title (cleaned, no citation brackets)       |
| Song     | Song title                                       |
| Singer   | Playback singer(s), comma-separated if multiple  |
| Composer | Music director / composer                        |

Films that pass the `page_is_film` check but yield no songs are logged to `no_songs_log.csv` for manual review.