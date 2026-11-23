# hallelx2/gws-cli: what this fork adds

This is a fork of [googleworkspace/cli](https://github.com/googleworkspace/cli) (`gws`). The branch you are reading, **`hallelx2-main`**, is upstream `main` plus the changes below. Every change is also offered upstream as a pull request. The aim is to carry nothing here for long: when upstream merges and releases a change, it drops out of this list.

Based on upstream `main` at `cd3697e` (30 Oct 2026, released as v0.22.5).

## The differences at a glance

| Change | What it lets you do | Branch | Upstream |
|---|---|---|---|
| Unlisted APIs by `<api>:<version>` | Call any Google API with a Discovery document, not only the built-in list: Search Console, Analytics, YouTube and others | `fix/670-unlisted-api-version` | [#990](https://github.com/googleworkspace/cli/pull/990) (fixes [#670](https://github.com/googleworkspace/cli/issues/670)) |
| Named auth profiles | One `gws` for several Google accounts: `gws --profile work …` | `feat/auth-profiles` | [#989](https://github.com/googleworkspace/cli/pull/989) by @gfortaine, plus our `gws auth profiles` command |
| Token cache tied to the credential | A new login (new scopes or another account) never reuses an old access token | `fix/780-token-cache-invalidation` | [#991](https://github.com/googleworkspace/cli/pull/991) (fixes [#780](https://github.com/googleworkspace/cli/issues/780), [#572](https://github.com/googleworkspace/cli/issues/572)) |
| Gmail `+read <query>` and `+code` | Read the newest matching mails as plain text in one command; pull a one-time code straight into a shell variable | `feat/gmail-read-code` | PR pending |
| Drive `+path`, `+sync`; Docs `+export`, Markdown `+write` | Resolve Drive paths to ids; mirror a local folder to Drive with stable ids (replaces hand-written sync scripts and drive.json maps); move documents between Markdown and Google Docs | `feat/drive-docs-helpers` | PR pending |
| Meet `+recent`, `+attendees`, `+transcript`, `+notes`, `+recording`, `+new` | After a call, pull its attendees, transcript, Gemini notes and recording in one command each; list recent meetings with their Calendar titles; make a meeting link with a Calendar event | `feat/meet-helpers` | [#993](https://github.com/googleworkspace/cli/pull/993) |
| Search Console and Analytics `+report`, `+properties` | One-command search performance and GA4 reports as a table, CSV or JSON, with a clear "no data in range" line | `feat/report-helpers` (on `feat/ordered-report-output`) | PR pending |
| YouTube `+upload`, `+videos` | Resumable upload with progress, private by default, thumbnail and playlist; list your recent uploads with stats | `feat/youtube-helpers` | PR pending |
| Forms `+responses` | One row per response, question titles as headers, `--since` | `feat/forms-responses` | PR pending |
| Calendar Meet links, `+free`, `--all-profiles` | `+agenda` shows the Meet link; `+free` finds open slots in working hours; both can merge every signed-in profile | `feat/calendar-agenda-free`, `feat/calendar-all-profiles` | PR pending (`--all-profiles` waits on #989) |

## 1. Unlisted APIs: `<api>:<version>`

Upstream's "Unknown service" error says to `Use '<api>:<version>' syntax for unlisted APIs`, but upstream never implemented it, so every unlisted API failed. Here it works:

```bash
gws searchconsole:v1 sites list
gws searchconsole:v1 searchanalytics query --params '{"siteUrl":"sc-domain:example.com"}' \
  --json '{"startDate":"2026-10-01","endDate":"2026-10-31","dimensions":["query"]}'
gws analyticsadmin:v1beta accountSummaries list
gws analyticsdata:v1beta properties runReport --params '{"property":"properties/123"}' \
  --json '{"dateRanges":[{"startDate":"28daysAgo","endDate":"today"}],"metrics":[{"name":"activeUsers"}]}'
gws youtube:v3 channels list --params '{"part":"snippet","mine":true}'
gws schema searchconsole:v1.searchanalytics.query
```

- `--api-version v1` works too: `gws searchconsole --api-version v1 sites list`.
- A name and version are checked before they go into a URL or a cache filename. Only letters, digits, `-`, `_` and `.` are allowed, so `../x:v1` is rejected.
- An unlisted name with no version still fails, so a typo in a built-in name like `drvie` still errors straight away.
- The API has to be enabled in the Google Cloud project behind your OAuth client, and your login needs its scope. For example, Search Console needs `webmasters` or `webmasters.readonly`.

Code: `crates/google-workspace/src/services.rs` (`resolve_service_with_version`, `split_service_spec`), `crates/google-workspace-cli/src/main.rs` (`parse_service_and_version`).

## 2. Named auth profiles

Upstream has one login per config directory, so several accounts meant several config dirs and a wrapper script for each. Here one binary holds several accounts:

```bash
gws --profile work auth login -s drive,gmail   # sign a profile in
gws --profile work drive files list            # use it
GOOGLE_WORKSPACE_CLI_PROFILE=work gws gmail users getProfile --params '{"userId":"me"}'
gws auth status --profile work                 # who is signed in there
gws auth profiles                              # list every profile
```

- No `--profile` means the **`default`** profile, which is exactly upstream's layout (`~/.config/gws`). Nothing changes for one-account users.
- Named profiles live in `~/.config/gws/profiles/<name>/`. Each has its own encrypted credentials, token caches and timezone cache. It can also have its own `client_secret.json`; otherwise the shared one is used.
- Shared between profiles: the encryption key and the Discovery document cache.
- Profiles are isolated from Application Default Credentials and from each other's login temp files.

This is upstream PR #989, adopted as is and tested here with three accounts, plus one commit of ours (`gws auth profiles`). The README section "Multiple accounts (profiles)" has the full reference.

**Moving from one-config-dir-per-account wrappers:** each old dir had its own encryption key, so its `credentials.enc` must be decrypted with the old key and re-encrypted with the shared one. Copying the file won't work. A wrapper can then shrink to `exec gws --profile <name> "$@"`.

## 3. Token cache tied to the credential

Upstream caches access tokens per scope, and `gws auth login` didn't clear that cache. After signing in again with new scopes, or as another account, gws kept serving the old token until it expired, for up to an hour. It typically showed up as `403 insufficient authentication scopes` straight after a login that granted those scopes.

Here, `gws auth login` clears the profile's token caches. Each cached token also records a fingerprint of the credential that minted it, so a token from a different login is never reused, even when switching with `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE`.

Code: `crates/google-workspace-cli/src/token_storage.rs`, `auth.rs`, `auth_commands.rs`.

## 4. Gmail: `+read <query>` and `+code`

Reading a mail used to take three steps: `messages list` to find it, `messages get` to fetch it, then decoding base64url MIME parts by hand. Pulling a sign-in code out of a mail took the same three plus a regex. Two helpers replace both:

```bash
gws gmail +read 'from:hello@example.com'              # newest match: From, To, Date, Subject, attachments, body
gws gmail +read 'subject:invoice newer_than:7d' --limit 3
gws gmail +read 'from:alerts@example.com' --format json | jq '.[0].body_text'
gws gmail +read --id 18f1a2b3c4d --headers            # upstream's --id form still works

code=$(gws gmail +code 'from:hello@example.com')      # only the code reaches stdout
gws gmail +code 'from:no-reply@example.com' --within 5m --wait 90s
gws gmail +code 'subject:"sign in"' --pattern '[A-Z0-9]{6}'
gws --profile bachs gmail +code 'from:hello@example.com'
```

- `+read` takes a search query or `--id`. With a query it reads the newest `--limit` matches (default 1, at most 50) and always prints the headers. Bodies prefer `text/plain`. An HTML-only body is converted to text: scripts and styles are dropped, and links keep their target in brackets. Attachments are listed by name, size and type. JSON output is an array with `id`, the headers, `body_text`, `body_html` and `attachments`.
- `+code` reads only the newest match received within `--within` (default `10m`). It checks the subject first, then the body. The default pattern is 4 to 8 digits, or two groups of 3 or 4 digits split by one space; the space is removed. A number shortly after a word like "code", "OTP" or "verification" wins. Without one, the first number that isn't a year wins, and digits inside links are ignored. `--pattern` takes your own regex, and uses its first capture group when it has one. `--wait` polls every 5 seconds until a code arrives.
- Failures from `+code` go to stderr only: no code exits 1, auth failures exit 2, bad arguments exit 3. stdout holds the code or nothing, so a failed lookup is never mistaken for a code. Check the exit status (`code=$(…) || exit 1`).
- Both are read-only (`gmail.readonly`) and use whichever profile `--profile` selects.
- A side effect of the HTML conversion: when `+forward` quotes an HTML-only message in plain text, it now quotes the whole message instead of Gmail's 200-character snippet.

Code: `crates/google-workspace-cli/src/helpers/gmail/read.rs`, `code.rs`, `html_text.rs`; shared search and fetch helpers in `gmail/mod.rs`.

## 5. Drive and Docs: `+path`, `+sync`, `+export`, Markdown `+write`

Syncing posters, decks and PDFs to Drive meant a Python script per repo that listed folders, compared checksums, picked `files.create` or `files.update`, and kept a drive.json of ids by hand. Moving a plan between Markdown and Google Docs meant copy and paste. Four helpers replace both:

```bash
gws drive +path 'MBPrep Company/design/social'           # → id; lists the ids if two items share a name
gws drive +path 'Shared drives/Engineering/specs' --create  # makes missing folders

gws drive +sync ./out 'MBPrep Company/design/social/launch' --ids drive.json
gws drive +sync ./out FOLDER_ID --dry-run --delete        # show what would change, change nothing
gws --profile bachs drive +sync ./deck 'Decks/2026'

gws docs +write plan.md --new 'Q1 plan' --folder 'MBPrep Company/strategy'
gws docs +write plan.md --doc 'MBPrep Company/strategy/Q1 plan'   # replace the body in place
gws docs +export 'MBPrep Company/strategy/Q1 plan' --output plan.md
gws docs +export DOC_URL --format txt                     # or html
```

- **Paths.** `a/b/c`, `/a/b/c` and `My Drive/a/b/c` start at My Drive; `Shared drives/<name>/…` starts at a shared drive. Write a slash inside a name as `\/`. Anywhere a folder or doc is expected, an id, a Drive or Docs URL, or `id:<ID>` also works. Resolution goes one folder name at a time and never guesses between items that share a name.
- **`+sync`** builds the whole plan before touching Drive. If two remote items share a name, a file sits where a folder should be, or a Google Doc has a local file's name, it lists every such path and changes nothing. New files are uploaded; files whose size or MD5 differ are updated in place with `files.update`, so ids and share links stay. `--delete` moves extra items to the trash, never permanently, and leaves Google Docs, Sheets and Slides alone. `--ids` writes `{"root", "folders": {path: id}, "files": {path: id}}`. Subdirectories sync recursively; `.DS_Store`, `Thumbs.db`, `desktop.ini` and `.git` are skipped, symlinks are not followed, `--exclude GLOB` skips more. Uploads run four at a time, with retries on rate limits, 5xx and dropped connections, and resumable uploads above 5 MB.
- **`+write`** goes through Drive's own Markdown import, so headings, nested lists, bold, italic, strikethrough, links, code (monospace), tables and images become real formatting. `--doc` replaces the body of an existing Doc and keeps its id, link, sharing and history. Local images are embedded; remote image URLs must be public. `--document ID --text '…'` still appends plain text as before.
- **`+export`** uses Drive's export. Its Markdown embeds images as base64; here they become links instead, to files saved in `<name>_images/` next to `--output`, or on stdout to the Docs image URLs (which expire). Drive's needless escapes (`a\_b`), list-item line breaks and `:----` table rules are tidied, and the text export loses its BOM and CRLFs. A document over Drive's 10 MB export limit is converted from the Docs API JSON instead.
- All four honour `--profile` and `--dry-run`. The local directory, `--output` and `--ids` must lie under the current directory.

Verified live on 2026-11-23: a re-sync after editing one file kept that file's id and served the new content; `--dry-run --delete` then `--delete` trashed exactly the file removed locally; a duplicate remote name stopped both `+sync` and `+path`. A Markdown file with headings, nested and numbered lists, a table, a code block, a quote and a local image went through `+write --new` and back through `+export`. The only differences were Drive's `python` → `py` fence label and the image path.

Code: `crates/google-workspace-cli/src/helpers/drive/{api,path,sync}.rs`, `helpers/docs/{export,write,json_to_md}.rs`.

## 6. Google Meet: `+recent`, `+attendees`, `+transcript`, `+notes`, `+recording`, `+new`

After a call, one command each pulls what happened in it. The helpers join the Meet REST API v2 with Calendar (event titles, notes attachments) and Drive (Doc exports, recording files). Every helper takes a conference record (`conferenceRecords/ID`), a meeting code (`abc-defg-hij`), a pasted Meet link, or `latest`.

```bash
gws meet +recent                              # last 7 days: start, duration, code, Calendar title, head count
gws meet +recent --days 30 --limit 5 --format json
gws meet +attendees latest                    # name, signed-in/anonymous/phone, first join, last leave, total time
gws meet +transcript latest                   # "[00:12:31] Ada: …" speaker-labelled text
gws meet +transcript abc-defg-hij --format json
gws meet +notes latest                        # Gemini "Take notes for me" notes as text
gws meet +recording latest --download recordings
gws meet +new --title 'Design review' --minutes 45   # link + Calendar event holding it
gws --profile bachs meet +recent              # works with any profile
```

- When a meeting has no transcript, notes or recording, the helper says so and exits 1 (`notFound`). It never prints an empty success.
- `+transcript` reads `transcripts.entries` and resolves speakers through the participant list. When the API has no entries, it exports the transcript Doc instead.
- `+notes` reads Meet smart notes first. If there are none, it falls back to the Gemini notes Doc attached to the Calendar event.
- `+recording --download` takes a relative directory inside the current one. It names files after the Drive file and adds the missing `.mp4`. It skips files already downloaded at full size, and an interrupted download resumes from its `.part` file.
- Listing helpers default to `--format table`. `+transcript` and `+notes` default to plain text (`--format text`), and `--format json` gives structured output.
- Event titles come from the primary calendar. A meeting nobody put on your calendar has no title.
- Needed scopes: `meetings.space.readonly` (plus `meetings.space.created` for `+new`), `calendar`, `drive`.

Code: `crates/google-workspace-cli/src/helpers/meet/`.

## 7. Search Console and Analytics reports

```bash
gws searchconsole +report sc-domain:example.com                 # top 25 queries, last 28 days
gws searchconsole +report sc-domain:example.com --by page,device --days 90 --limit 100
gws searchconsole +report https://example.com/ --by date --format csv
gws analytics +properties                                        # every GA4 account and property you can read
gws analytics +report 123456789 --by country --days 28
gws analytics +report properties/123456789 --by date,deviceCategory --metrics activeUsers,sessions
```

- `+report` (Search Console) takes `--by` from query, page, country, device, date and searchAppearance, `--days` (default 28) or `--start/--end`, `--limit` (default 25, at most 25,000), `--type` (web, image, video, news, discover, googleNews) and `--final-only`. A totals line is fetched alongside. CTR shows as a percent and position to one decimal.
- `+report` (Analytics) checks dimensions and metrics against an allowlist before the call. Default metrics are activeUsers, newUsers, sessions, screenPageViews and engagementRate; rows sort by the first metric, or by date when `--by date`. `G-` measurement ids are rejected with a hint to use the numeric property id.
- An empty range prints `No data in range: …` instead of an empty table; CSV keeps its header and puts the note on stderr; JSON carries a `note` field.
- **Scope:** the Analytics Data API needs `analytics.readonly`; `analytics.edit` is not enough. A 403 prints the exact `gws auth login --scopes …` line to add it. Search Console needs `webmasters` or `webmasters.readonly`.
- `--dry-run` prints the request(s) without calling the API.

Code: `helpers/searchconsole.rs`, `helpers/analytics.rs`; shared ordered output in `formatter.rs` (`render_report`) and `executor.rs` (`fetch_json`).

## 8. YouTube: `+upload` and `+videos`

```bash
gws youtube +videos --limit 10
gws youtube +upload clip.mp4 --title 'Launch' --description @desc.md --tags 'mbprep,launch' --dry-run
gws youtube +upload clip.mp4 --title 'Launch' --privacy unlisted --thumbnail thumb.png --playlist PLxxxx
```

- `+upload` is resumable in 8 MiB chunks, with a progress line (deciles when stdout isn't a terminal). On a 5xx, 429 or dropped connection it asks the server how much it holds and resumes from there, up to 8 retries with backoff. Privacy defaults to **private**. `--publish-at` schedules a private video. Title (≤100 chars), description (≤5000 bytes, `@file` allowed), tags (≤500 chars) and file types are checked before any request. A thumbnail or playlist failure after a successful upload becomes a warning, not an error, so the video id is never lost.
- `+videos` lists your channel's uploads newest first: published, title, privacy, views, likes, comments, duration, url.
- `--dry-run` prints every request `+upload` would make, without auth.

Code: `helpers/youtube.rs`.

## 9. Forms: `+responses`

```bash
gws forms +responses 1AbC…xyz                      # or the form's edit URL
gws forms +responses FORM_URL --since 2026-11-01 --format csv > responses.csv
```

One row per response, sorted by submit time, with the question titles as headers. Grid questions become one column per row (`Title [row]`); several answers join with `; `; uploaded files show as name plus Drive link. Email and score columns appear only when the form collects them. The public `/forms/d/e/…` link is rejected with a hint to use the edit URL.

Code: `helpers/forms.rs`.

## 10. Calendar: Meet links, `+free`, `--all-profiles`

```bash
gws calendar +agenda --days 1                       # now with a meet column
gws calendar +free --days 3 --minutes 30 --between 09:00-18:00
gws calendar +free --all-profiles --weekdays        # busy time merged across every signed-in profile
gws calendar +agenda --all-profiles
```

- `+agenda` adds the Meet link (hangoutLink, or the conference video entry point) and prints `No events between …` when empty.
- `+free` reads events, not freeBusy, so it can skip what shouldn't block you: cancelled, "free"/transparent and declined events, and all-day events unless they are out-of-office (or `--count-all-day`). It falls back to freeBusy for calendars it can't list. Slots start on the quarter hour, in the calendar's time zone (Africa/Lagos here) or `--timezone`.
- `--all-profiles` runs the same command once per signed-in profile and merges the results; profiles without credentials are skipped with a note. It depends on named profiles (§2), so it is fork-only until #989 lands.

Code: `helpers/calendar.rs`.

## Building and installing

```bash
git clone https://github.com/hallelx2/gws-cli && cd gws-cli
git checkout hallelx2-main
cargo build --release
install -m755 target/release/gws ~/.local/bin/gws   # put ~/.local/bin before any npm/mise gws on PATH
gws --version
```

It uses the same config dir and keyring as the upstream binary, so an existing login keeps working. To go back to upstream, delete `~/.local/bin/gws`.

## Keeping the fork current

```bash
git fetch origin                     # origin = googleworkspace/cli
git checkout hallelx2-main
git merge origin/main                # then re-merge any branch that changed:
git merge fix/670-unlisted-api-version feat/auth-profiles fix/780-token-cache-invalidation feat/gmail-read-code feat/drive-docs-helpers feat/meet-helpers \
  feat/report-helpers feat/youtube-helpers feat/forms-responses feat/calendar-all-profiles
cargo test && cargo build --release
```

When upstream merges one of the PRs above, remove its row from the table. When all of them are released, retire the fork and go back to the official package.

## Tests

`cargo fmt --check` is clean, and each change brings its own unit tests. On rustc 1.99, upstream `main` itself already fails some `cargo clippy -- -D warnings` lints in files these changes don't touch. It also fails two ADC tests (`test_get_quota_project_reads_adc`, `test_get_quota_project_priority_adc_fallback`) on machines with gcloud ADC configured. Those failures aren't introduced here.

## Known limitation, not a code change

An OAuth client whose consent screen is in **Testing** mode issues refresh tokens that expire after about 7 days. With it, each profile needs `gws --profile <name> auth login` again weekly. Publishing the consent screen, or using an internal Workspace client, removes the limit.
