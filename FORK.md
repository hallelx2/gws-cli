# hallelx2/gws-cli: what this fork adds

This is a fork of [googleworkspace/cli](https://github.com/googleworkspace/cli) (`gws`). The branch you are reading, **`hallelx2-main`**, is upstream `main` plus the changes below. Every change is also offered upstream as a pull request. The aim is to carry nothing here for long: when upstream merges and releases a change, it drops out of this list.

Based on upstream `main` at `cd3697e` (30 Oct 2026, released as v0.22.5).

## The differences at a glance

| Change | What it lets you do | Branch | Upstream |
|---|---|---|---|
| Unlisted APIs by `<api>:<version>` | Call any Google API with a Discovery document, not only the built-in list: Search Console, Analytics, YouTube and others | `fix/670-unlisted-api-version` | [#990](https://github.com/googleworkspace/cli/pull/990) (fixes [#670](https://github.com/googleworkspace/cli/issues/670)) |
| Named auth profiles | One `gws` for several Google accounts: `gws --profile work …` | `feat/auth-profiles` | [#989](https://github.com/googleworkspace/cli/pull/989) by @gfortaine, plus our `gws auth profiles` command |
| Token cache tied to the credential | A new login (new scopes or another account) never reuses an old access token | `fix/780-token-cache-invalidation` | [#991](https://github.com/googleworkspace/cli/pull/991) (fixes [#780](https://github.com/googleworkspace/cli/issues/780), [#572](https://github.com/googleworkspace/cli/issues/572)) |

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
git merge fix/670-unlisted-api-version feat/auth-profiles fix/780-token-cache-invalidation
cargo test && cargo build --release
```

When upstream merges one of the PRs above, remove its row from the table. When all of them are released, retire the fork and go back to the official package.

## Tests

`cargo fmt --check` is clean, and each change brings its own unit tests. On rustc 1.99, upstream `main` itself already fails some `cargo clippy -- -D warnings` lints in files these changes don't touch. It also fails two ADC tests (`test_get_quota_project_reads_adc`, `test_get_quota_project_priority_adc_fallback`) on machines with gcloud ADC configured. Those failures aren't introduced here.

## Known limitation, not a code change

An OAuth client whose consent screen is in **Testing** mode issues refresh tokens that expire after about 7 days. With it, each profile needs `gws --profile <name> auth login` again weekly. Publishing the consent screen, or using an internal Workspace client, removes the limit.
