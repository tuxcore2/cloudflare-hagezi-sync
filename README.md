# Cloudflare Hagezi Sync
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)

Keeps a Cloudflare Zero Trust Gateway DNS policy in sync with the [Hagezi Pro](https://github.com/hagezi/dns-blocklists) blocklist. It runs on GitHub Actions every six hours, so once it's set up you don't have to touch it.

## How it works

Cloudflare's free plan allows 300 lists with 1,000 domains each, so the blocklist gets split into chunks of 1,000 and every chunk becomes one Gateway list (`Hagezi_Pro_List_1`, `Hagezi_Pro_List_2`, ...). A single policy named `Hagezi Pro` blocks anything that matches one of those lists.

On each run the script:

1. Reads the `# Version:` header of the Hagezi file and compares it with the version stored in the policy description. If nothing changed, and the lists still hold the number of domains the policy expects, it stops there. That's the usual outcome.
2. Otherwise it downloads the list, reads what's currently in your Cloudflare lists, and works out the difference. Only that difference is sent: domains that are gone get removed, new ones fill the free space in existing lists, and extra lists are created or deleted only when the size changes.
3. Switches the policy to the final set of lists, then deletes lists that are no longer used.

The order matters. Removals go first (so you stay under the free-plan limits), unused lists are deleted last, so a domain that is being moved between lists is never unblocked in between.

If anything fails halfway, the version is not written to the policy and the job exits with an error. The next run notices that the state doesn't match and finishes the job. Because it only sends differences, that repair is cheap.

It also refuses to do anything when the download looks wrong: no valid domains, more domains than the free plan allows, or a list that suddenly shrank to under half of its previous size (usually a truncated download). Nothing is changed in that case. If the shrink is real, run it once with `FORCE_SYNC=true`.

## Setup

You need a Cloudflare account with Zero Trust enabled (the free plan is fine) and a GitHub account.

**1. Cloudflare account ID.** Log in to the [dashboard](https://dash.cloudflare.com). The 32-character string in the URL is your account ID.

**2. API token.** Go to [API Tokens](https://dash.cloudflare.com/profile/api-tokens), create a custom token and give it these permissions:

- Account / Zero Trust / Read
- Account / Zero Trust / Edit

Copy the token right away, Cloudflare only shows it once.

**3. Put the code in your own repo** (fork it, clone it, or copy the files), then add two secrets under *Settings > Secrets and variables > Actions*:

- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`

Use exactly these names, the workflows read them.

**4. Run it once by hand.** Open the *Actions* tab, pick *Cloudflare Hagezi Sync* and press *Run workflow*. Check the log: you should see a line starting with `Plan:` on the first run and `up to date` on the second.

After that the schedule takes over: 23 minutes past 00:00, 06:00, 12:00 and 18:00 UTC. The odd minute is deliberate, top-of-the-hour slots are busy and GitHub tends to delay them. Note that GitHub can pause scheduled workflows in a public repo after about 60 days without any activity. If the runs stop, re-enable the workflow from the Actions tab.

## Running it locally

```bash
pip install -r requirements.txt
export CLOUDFLARE_API_TOKEN=...
export CLOUDFLARE_ACCOUNT_ID=...
python update_gateway.py
```

I developed it on Python 3.12, the workflow uses 3.14.

## Settings

All optional, set them as environment variables (or in the workflow's `env:` block).

| Variable | Default | What it does |
|---|---|---|
| `FORCE_SYNC` | `false` | Skip the version/drift check and the shrink guard. Also available as a checkbox when you run the workflow by hand. |
| `FRESH_START` | `false` | Delete the policy and all of its lists first, then rebuild. Blocking is off while this runs, so use it only when something is badly broken. Also a checkbox in the workflow. |
| `CHECK_VERSIONS` | `true` | Set to `false` to always run the full comparison. |
| `MIN_KEEP_RATIO` | `0.5` | Abort if the new list is smaller than this fraction of the old one. |
| `MAX_LISTS` | `300` | Your account's list limit. |
| `MAX_TOTAL_DOMAINS` | `300000` | Your account's total domain limit. |
| `REQUESTS_PER_SECOND` | `4` | Cloudflare allows roughly 1,200 API calls per 5 minutes. Raising this is safe in the sense that 429 responses are retried, but it won't make things much faster. |
| `MAX_CONCURRENT_REQUESTS` | `5` | Parallel requests. |
| `REQUEST_TIMEOUT` | `30` | Seconds per API request. |

If you're on a paid plan with higher limits, raise `MAX_LISTS` and `MAX_TOTAL_DOMAINS` accordingly.

## Using a different blocklist

Edit `BLOCKLISTS` in `gateway_logic.py`. Each entry has a name, a list of URLs (the first one is tried first, the rest are fallbacks) and a policy precedence. The name decides what the policy and lists are called, so changing it means a new policy and new lists get created. The old ones stay until you remove them, so run the cleanup workflow before you rename.

## Cleanup

The *Cloudflare Hagezi Cleanup* workflow removes the policy and lists this script created, nothing else. It only touches a policy with the exact blocklist name and lists named `<Name>_List_<number>`, so anything you made by hand is left alone.

## Tests

```bash
python -m unittest discover -s tests
```

The tests cover the parsing, the skip logic, the diff planner (with randomized inputs) and the whole sync flow against an in-memory fake of the Gateway API. That fake enforces list and item limits and checks after every call that nothing that should be blocked loses coverage. It is still a fake, though: the real Cloudflare API isn't part of the test suite, so if Cloudflare changes how it behaves, the tests won't tell you.

## Troubleshooting

- **Fails right away with an auth error:** check the secret names and that the token has the permissions above.
- **`SafetyError` in the log:** one of the checks stopped the run before anything was changed. The message says which one.
- **Run fails with "failed, version NOT recorded":** some API calls failed. Just wait for the next run, or start it manually.
- **Scheduled runs stopped:** see the note about inactive repos in the setup section.
