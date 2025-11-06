# Implementation Plan

[Overview]
Create authoritative documentation that deeply analyzes the current Twitter/X scraping solution in this repository, focusing first on the methods actually used by twitter_suite/cli.py and the PM2 loop, with a follow-up path to document the full twscrape API surface.

This plan covers the documentation scope, structure, and concrete steps to produce a complete, accurate methods reference for the running pipeline. It maps CLI commands to their underlying twscrape API calls and pool/locking behavior, details data models and DB interactions, and captures operational concerns like rate limiting, account lock rotation, and error handling. No code changes are required; deliverables are documentation files.

[Types]
Define a consistent documentation schema for methods and commands to ensure completeness and uniformity.

Proposed documentation data shapes (conceptual):
- MethodDoc
  - name: string (e.g., API.search)
  - module: string (e.g., twitter/twscrape/twscrape/api.py)
  - signature: string (verbatim from code)
  - description: string (purpose and behavior)
  - inputs: list of {name, type, description}
  - returns: string (type/iterator semantics)
  - side_effects: string (locks, DB writes, rate-limit changes)
  - errors: list of {type, when, handling}
  - rate_limiting: string (x-rate-limit headers, 15-min lock behavior)
  - dependencies: list of other methods/classes used
  - examples: list of minimal usage examples (pseudo-code)

- CLICommandDoc
  - name: string (e.g., batch-targets)
  - entrypoint: twitter_suite/cli.py function name (e.g., cmd_batch_targets)
  - arguments: list of {flag, type, default, notes}
  - flow: step-by-step of what it does
  - underlying_calls: list of MethodDoc references
  - inputs/outputs: files, DBs (e.g., scripts/cache/twitter_targets.json, SqliteMinerStorage.sqlite)
  - failure_modes: common errors and messages

[Files]
Documentation-only change: create one or more markdown files under the repository root (or docs/).

- New files to be created:
  - implementation_plan.md (this plan)
  - docs/twitter_scraper_method_support.md: primary deliverable, deep-dive docs for the CLI-used surface
  - docs/twscrape_full_api_reference.md: placeholder with structure to be filled subsequently

- Existing files to be modified:
  - None (documentation effort; no code changes)

- Files to be deleted or moved:
  - None in this plan; any cleanup will be a separate, explicit task

- Configuration updates:
  - None

[Functions]
No code functions will be added or modified in this plan; instead, the plan enumerates functions to document.

- New functions: N/A
- Modified functions: N/A
- Removed functions: N/A

However, the documentation will cover these existing functions in depth (CLI-used path first):

twitter_suite/cli.py
- cmd_trends(args): updates targets/trending by calling scripts.target_provider.update_twitter_targets(pool, include_trending=True, ...)
- cmd_batch_targets(args): reads scripts/cache/twitter_targets.json, windowing since/until, runs API.search to collect tweets into SqliteMinerStorage.sqlite
- cmd_batch(args): like batch-targets but queries are from flags/files
- cmd_one_shot(args): API.search for a single query, XContent mapping to DataEntity and prints JSON
- cmd_refresh(args): account cookie validation, or IMAP-based relogin via AccountsPool; writes refreshed accounts file
- cmd_load(args): AccountsPool.add_account for each account, proxy assignment
- cmd_reset_locks(args): AccountsPool.reset_locks
- cmd_status(args): AccountsPool.stats, next_available_at, list locked/unlocked
- cmd_report(args): twitter_suite/report.gather_pool_data + CSV/JSON export
- cmd_accept(args): orchestration pipeline over the above (sample run)

twitter/twscrape/twscrape/api.py (document these methods as they relate to CLI usage)
- API.search(q, limit=-1, kv=None)
- API.search_raw(q, ...)
- API.user_by_login(login, ...)
- API.user_by_id(uid, ...)
- API.tweet_details(twid, ...)
- API.tweet_replies(twid, limit=-1, ...)
- API.user_tweets(uid, ...), API.user_tweets_and_replies(uid, ...), API.user_media(uid, ...)
- API.followers(uid, ...), API.verified_followers(uid, ...), API.following(uid, ...)
- API.list_timeline(list_id, ...)
- API.trends(trend_id, ...); API.search_trend(q, ...)
- API.bookmarks(...)

twitter/twscrape/twscrape/accounts_pool.py (pool mechanics used by CLI)
- AccountsPool.add_account, relogin, login_all, get_all, get_for_queue(_or_wait), reset_locks, next_available_at, mark_inactive, stats

twitter/twscrape/twscrape/queue_client.py (operational semantics)
- QueueClient: account checkout/return, request execution, x-client-transaction-id retry, rate-limit handling and 15-minute lock window, ban detection, error categories (HandledError, AbortReqError)

scraping/x/model.py and scraping/utils.py
- XContent schema, DataEntity mapping, timestamp obfuscation to minute

scripts/target_provider.py
- update_twitter_targets: gravity + trending sampling via API.search; writes scripts/cache/twitter_targets.json and twitter_trending.json

[Classes]
No new/modified classes in code; the docs will detail behavior of existing classes:
- twscrape.API, twscrape.AccountsPool, twscrape.QueueClient, twscrape.models.* (Tweet, User, Trend)
- XContent (scraping/x/model.py)
- DataEntity/DataSource (common/data.py)

[Dependencies]
No runtime dependency changes. The docs will note relevant packages for context:
- httpx, fake_useragent, pyotp, sqlite3, pydantic (v1 used in XContent), etc.

[Testing]
Validation is documentation-oriented:
- Cross-check each documented signature against the source files cited
- Trace CLI flows with their default arguments to ensure inputs/outputs are accurate
- Verify locking and rate-limit notes match QueueClient and AccountsPool semantics
- Optional: compare example outputs to recorded log snippets (without executing commands here)

[Implementation Order]
Produce docs in a sequence that matches operational importance and reduces rework:

1) Document CLI commands used by PM2 loop
   - cmd_trends → scripts/target_provider.update_twitter_targets
   - cmd_batch_targets → API.search data flow to DB

2) Document supporting CLI commands
   - cmd_batch, cmd_one_shot
   - cmd_refresh, cmd_load, cmd_reset_locks, cmd_status
   - cmd_report (reporting outputs)

3) Document underlying twscrape classes and key methods used by the above
   - API.search (+ raw), trends, search_trend
   - AccountsPool access patterns
   - QueueClient rate limit and locking behavior

4) Document models and persistence
   - XContent and DataEntity mapping, timestamp obfuscation rule
   - ensure_db, insert_data_entity schema/indices

5) Prepare structure for full API reference and list remaining methods to cover later

6) Final review pass for consistency and completeness; link file paths and function names
