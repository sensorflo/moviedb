# MovieDB – plan

A personal movie database replacing EMDB. The design is in discussion; nothing of it is built yet. For what
exists, see albumflow's `docs/plan.md` (section "Movies: store, view, movie annex, EMDB") and
`src/albumflow/movies.py`, `emdb.py`, `mediainfo.py`, `movienames.py`.

## Working with the user

- Discuss designs before building; ask where unsure, with a recommendation, not a survey. The user answers
  point by point and decides; they question terms and want the established one where there is one.
- Reliability first, then simplicity (KISS); efficiency last. One self-contained commit per feature.
- Consistent terminology everywhere; help texts without needless redundancy; concise CLI output.
- Deletes hard to do by accident (choose by number, then type `delete`).
- Bash heredocs on this machine mangle backslash escapes (`\\`, `\n`); write scripts with the Write tool and
  run them, or edit with the Edit tool. Run tests with `-W error::SyntaxWarning`.

## What exists (albumflow, committed)

albumflow still says "release" (name) for a store folder('s name); see Open.

- **Movies on disk**, `C:\Users\senso\Large\Videos`:
  - `Store\<folder>`: movies as downloaded, never renamed (qBittorrent seeds them).
  - `Downloading`: qBittorrent's incomplete downloads; albumflow ignores it.
  - `Videos\<canonical name>`: the view, junctions only, named
    `<English title>[ - <original title>] (<year>)[ {<edition>}] [<tech>]`.
  - The movie annex: an external HDD (marker `.movie-annex`, often disconnected), with the store at
    `<HDD>\Large\Videos\Store`; junctions to it point to its volume.
  - qBittorrent: save path `Store`, incomplete torrents in `Downloading`, content layout "Create subfolder".
- **Commands:** `movie-to-annex`, `movie-from-annex`, `movie-status [--fix] [-v]`.
  - `--fix` fixes the junctions (also after qBittorrent's Set location), names them, and keeps EMDB's
    Locations pointing through the view.
  - The files are the master: movies added, renamed and removed are followed.
- **MediaInfo** (CLI, `C:\Program Files\MediaInfo CLI\MediaInfo.exe`): tracks per file, cached in
  `%LOCALAPPDATA%\albumflow\mediainfo.json` (by store folder and file, size and time). For a Blu-ray folder,
  its largest playlist (not yet verified on real data).
- **The tech part of names:** resolution, HDR, video codec and Mbit/s. Audio per language: the original (the
  first track's), then en and de, each its best track. kbit/s for lossy formats only. Subtitle languages;
  others counted as `+n`.
- **EMDB** (5.41, 32-bit, `%APPDATA%\EMDB\emdb.dat`):
  - UTF-16 with BOM, CRLF, JSON-like.
  - Per movie a `"title"` line (title, `#O<original title>`, studio).
  - A `"year"` line of 17 fields separated by `\x1e`: 0 year, 2 runtime, 3 country (`#US`), 6 resolution,
    7 Location.
  - A `"genres"` line: field 1 the IMDb id (`6751668|0`), the last field the TMDB id.
  - Escapes in values: `[ ] { } "` as `<SC> <SCB> <CB> <CCB> <DQ>`. A raw `{` broke EMDB once.
  - albumflow changes only Locations, byte for byte otherwise, only while EMDB is closed and its newest
    backup made in EMDB (`Documents\EMDB\Backups\EMDB_BACKUP_*`) equals the database.
  - 754 entries, 357 with a Location; about 350 movies are on the HDD.
  - EMDB doesn't play Locations over 259 characters.
  - EMDB's MediaInfo is off (`UseMediaInfo = 0`); it would need the 32-bit MediaInfo.dll.
- **Backup tool** (`C:\Users\senso\src\backup`):
  - excludes `Large\Videos\*\*` (restic stores junctions as links, doesn't follow them)
  - backs up qBittorrent's `BT_backup`.
  - backup-vcs pushes git repos below `Documents` to their remote and fails for a repo without a remote of
    the user's.
- **Pending (user):** the HDD migration (`movie-status --fix` with the HDD connected, see albumflow's
  plan.md).

## Decided

- **Why:** EMDB's model is too narrow: one entry per file, no cuts, no tags, no table view, no filter
  expressions, no subtitle languages or bit rates. The user mainly used it for fetching metadata and as a
  simple database; they typed in only "seen" and sometimes the aspect ratio (MediaInfo has it).
- **Steps:**
  1. A catalog generated from the store, MediaInfo and TMDB; EMDB still in use.
  2. The personal data moved from EMDB into `movies.yml`.
  3. EMDB retired; albumflow no longer writes its Locations.
- **Place:** `Documents\MovieDB` (for now; the user isn't entirely happy with it), a git repo with a private
  GitHub remote (the user creates it), so backup-vcs pushes it. restic covers the working tree.
- **`movies.yml`:** the only source of personal data, one file, edited by hand.
  - albumflow only adds entries, keeping everything else exactly (ruamel.yaml), atomically.
  - Under `movies:`, keyed by **TMDB id** (TV series, if ever, under their own key).
  - Comments `# <title> (<year>)` written by albumflow; they stay, though from TMDB (public facts).
  - Holds: which movies (also seen without a file), seen, rating, notes, tags, cuts, which media are whose.
- **`movies.sqlite`:** the catalog, generated from scratch from `movies.yml`, the local copy of TMDB's data,
  the store and MediaInfo. Never edited, not in git.
  - Viewed with **Datasette** (localhost only): SQL filter expressions (the user wants full flexibility, not
    canned use cases), facets, saved queries.
  - Later a shelf page, a cover grid with the same filter expressions.
- **The hierarchy: movie → cut → edition → media.** "Release" isn't used.
  - Cut: which content (theatrical, director's, extended, …).
  - Edition: which studio release of a cut: Remastered, Criterion, `<n>th` Anniversary, IMAX.
  - Media: the files; each is exactly one store folder, e.g. a 1080p and a 2160p encode. A store folder holds
    one movie (no packs).
  - Today's `movienames.EDITIONS` splits accordingly: Director's Cut, Final Cut, Extended, Uncut, Unrated and
    Theatrical are cuts; Remastered, Criterion, Anniversary and IMAX are editions.
  - View names: `{<cut>}` only when it isn't theatrical; the edition leads the tech part, e.g.
    `Donnie Darko (2001) {Director's Cut} [Remastered, 2160p HEVC …]`.
- **A store folder deleted:** its line in `movies.yml` stays (albumflow only adds); the catalog shows it as
  not on disk. Later perhaps a cleanup/sync command.
- **Seen:** a list of dates per movie and per cut, in **EDTF level 1**: `2024-03-15`, `2024-03`, `2010~`,
  `2010?`, `2010%`, `199X`, `2015/2018`, `../1999`, `2015/..`.
  - "Seen, no idea when" is an open start, e.g. `../2026-10`; EMDB's seen movies migrate as that.
  - For searching, `~ ? %` widen by the precision: ±2 years, ±2 months, ±7 days.
  - Only a date per viewing; no ages.
- **Rating:** 1–5 whole stars, optional, per movie and per cut. The user won't use it at first.
- **Note:** free text, per movie and per cut.
- **Seen, rating and note on the movie** mean "some cut, unknown which" (or its only one); on a cut, that cut.
- **Tags:** per movie only (not per cut), hierarchical with `/`.
  - `movie-series/…`: set automatically from TMDB's collections (only in the catalog). "Movie", not "film",
    for consistency.
  - `franchise/…` and all other tags: typed by the user.
- **Cut** is the term for a version of the content: theatrical, director's, extended, final, redux,
  unrated, uncut.
  - A movie without a known cut has the theatrical one.
  - Known cut names form a list; others are allowed but reported as a hint (typos).
- **TMDB** is the only external source: no IMDb data or scraping, no ratings from IMDb or TMDB.
  - API terms: attribution with TMDB's logo (less prominent than ours) and "This program uses TMDB and the
    TMDB APIs but is not endorsed, certified, or otherwise approved by TMDB." (catalog footer, README, help).
  - Personal, non-commercial use, not shared; no AI or machine-learning use of TMDB data.
  - Nothing cached longer than 6 months: the local copy (`MovieDB\tmdb`, `MovieDB\posters`) is refreshed
    before then, and kept out of restic and git.
  - The API key (the user registers for it) lives in albumflow's settings under `%APPDATA%`, never in the
    repo.

## Draft: the `movies.yml` schema

`movies.yml` is the master of the hierarchy, nested; store folder names follow a convention but carry no
meaning.

```yaml
movies:
  1018:  # Mulholland Drive (2001)
    seen: [../2026-10]
    tags: [mood/dark, director/lynch]
    media: [Mulholland.Drive.2001.1080p.BluRay.x264-AMIABLE]

  141:  # Donnie Darko (2001)
    seen: [2004~]
    media: [Donnie.Darko.2001.1080p.BluRay.x264-ABC]
    cuts:
      Director's Cut:
        seen: [2024-03-15]
        rating: 4
        note: Clearer than the theatrical, less mysterious.
        editions:
          Remastered:
            - Donnie.Darko.2001.REMASTERED.Directors.Cut.2160p.UHD.BluRay.x265-XYZ
            - Donnie.Darko.2001.REMASTERED.Directors.Cut.1080p.BluRay.x264-DEF

  550:  # Fight Club (1999)
    seen: [2000?, 2015/2018]
```

- A movie: `seen`, `rating`, `note`, `tags`, `cuts`, `editions`, `media`, all optional. Without any media:
  seen without a file.
- A cut: `seen`, `rating`, `note`, `editions`, `media`, all optional; keyed by cut name.
- An edition: keyed by edition name (several in one, e.g. `Remastered, IMAX`), its value the list of its
  media, each a store folder name. Editions hold no personal data.
- `cuts:` and `editions:` are optional: `editions:` or `media:` directly on the movie are the theatrical
  cut's; `media:` directly on a movie or cut are of no particular edition.
- A store folder name occurs once in the whole file.
- Matching a new store folder proposes its cut and edition from the folder name (`movienames` rules); the
  user confirms or corrects, albumflow writes it there.
- Store folders not in the file: unmatched; the catalog lists them.
- The view's names take cut and edition from here; an unmatched folder keeps its folder name.
- New movies are appended at the end of `movies:`.
- Errors: unknown keys, rating outside 1–5, invalid EDTF dates, a folder name twice. Hint: cut names not on
  the list.
- YAML reads an unquoted `2024-03-15` as a date; albumflow accepts it, so dates are never quoted.
- Later top-level keys: `watchlist:` (ordered TMDB ids with the same comments), `series:`.

## Open

- **The nested schema above** (the user's proposal): in discussion. Not handled for now: a disc with
  seamless branching (several cuts in one store folder).
- **The view's names** (store folder names are never changed): `<English title>[ - <original title>]
  (<year>)[ {<cut>}] [[<edition>, ]<tech>]`; the tech part as albumflow's `mediainfo.describe` makes it today.
- **albumflow's terms** (decided, later): it says "release" (`movie-to-annex <movie>` takes "its release
  name", the view's `{<edition>}`, `mediainfo.json` "by release", its plan.md); to change to the terms above,
  consistently everywhere.
- **An ordered "to watch" list:** wanted later. It's an ordered list, so not just a tag.
- **To design:**
  - the catalog's tables
  - matching new store folders to TMDB movies (search by title and year, choice by number, stored in
    `movies.yml`)
  - the migration from EMDB (its TMDB ids and seen flags)
  - refreshing TMDB data within 6 months, spread over time
  - the shelf page
  - whether MovieDB is part of albumflow or a tool of its own.
