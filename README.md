# TM GROW

A social task marketplace with four connected systems:

- **Worker app** — Google sign-in with automatic name/photo sync, task marketplace, claim window, an in-app task runner (30-second timer, target link, optional proof link, up to 6 screenshots), review status, USDT wallet, on-chain deposit verification, BEP-20 withdrawals, transaction history, weekly leaderboard, referrals, notifications, profile, linked accounts and support tickets.
- **Buyer / order platform** — service catalogue with real platform logos, campaign builder (quantity, reward, target link, instructions, countries, minimum trust), escrow funding, delivery reporting, pause / resume / cancel, deposits, buyer wallet and ledger.
- **Master admin panel** — overview analytics, maintenance mode switch, users and balances, tasks, campaigns, proof verification, deposits, withdrawals, ledger, services and pricing, countries, support inbox, fraud flags, platform settings, admin accounts, audit log and announcements.
- **AI admin assistant** — answers from one live snapshot of the database (`admin_ai_snapshot`). Every number it prints comes from that snapshot; it never approves, pays or edits anything itself.

All money is **USDT on BNB Smart Chain (BEP-20)**. Approved work also earns **TG Coin**, a platform bonus token tracked in the wallet with no guaranteed cash value.

**Important honesty note:** verification, deposit crediting and payouts are **manual**. The code does not pretend otherwise, does not auto-approve proofs and never fabricates a transaction hash. Nothing in this project has been executed in a browser or against your database from here — see *Verification status* at the end.

---

## 1. Project layout

```
index.html               app shell, boot screen, mounts #app / #modal-root / #toast-root
styles/app.css           design system (dark fintech, orange/gold accent, mobile-first)
vendor/supabase.js       vendored supabase-js (no CDN, no build step)
vendor/telegram-web-app.js  vendored Telegram Mini App SDK
src/config.js            Supabase URL + publishable key, brand, network, bucket
src/lib/telegram.js      Mini App detection, init and sign-in through the Edge
                         Function (never trusts initDataUnsafe for auth)
src/lib/sb.js            Supabase client, RPC wrapper, friendly error messages
src/lib/store.js         session, profile/wallet context, settings, referral capture
src/lib/ui.js            icons (incl. platform logos), formatting, modals, toasts, uploads
src/lib/router.js        hash router with :param routes
src/views/landing.js     public landing page
src/views/auth.js        dark sign-in screen: Google OAuth, Telegram (custom OIDC
                         provider), email sign-in / sign-up
src/views/worker.js      dashboard, tasks, task detail, wallet, leaderboard, referrals,
                         profile, notifications, support
src/views/buyer.js       buyer hub, services, campaign builder, campaigns, campaign
                         detail, buyer wallet
src/views/admin.js       overview, users, tasks, campaigns, proofs, task accounts,
                         deposits, withdrawals, ledger, referrals, services,
                         countries, support, flags, settings, admins, audit,
                         announcements, Telegram, AI assistant
supabase/schema.sql      everything the database needs — run once in the SQL editor
supabase/telegram-task-completion.sql
                         rank-progress "Task Completed" Telegram alert (one per
                         approved submission, configurable ranks)
supabase/telegram-db-sender.sql
                         pg_net + Vault sender: dispatches telegram_outbox from
                         PostgreSQL, reconciles Telegram's HTTP response, retries,
                         and replaces the telegram-notify Edge Function
supabase/admin-control-room.sql
                         Phase 1 control room: real per-module status RPC,
                         database-native automation runs and the pg_cron schedule
supabase/admin-activity-risk.sql
                         Phase 2: user activity timeline, login history,
                         risk board and rule-based risk profile
supabase/verification-automation.sql
                         Database-native proof verification: deterministic sweep,
                         pg_cron schedule and admin queue RPCs (no Edge Function)
supabase/socify-integration.sql
                         Vault + pg_net Socify client, endpoint allowlist, cache,
                         limits and admin overview (no Edge Function)
supabase/social-verification-frontend.sql
                         Public profile check for workers, admin social queue,
                         task verification rules and submission status fields
supabase/socify-fix.sql  Socify diagnostics, real natgeo end-to-end test and
                         clearer lookup errors (additive, re-runnable)
supabase/promote-owner.sql   grants OWNER (admin panel access) to an account,
                             with role and Telegram variants
supabase/owner-access.sql    seeds settings.owner_email and adds claim_owner(),
                             admin_wallet_overview() and admin_system_status()
supabase/advanced-management.sql   wallet freeze, suspension evidence, admin
                             notes, login history, fraud scan and the richer
                             admin_user_detail (safe, re-runnable)
supabase/fix-suspension-columns.sql   repairs a database missing the suspension
                             columns (error: record "p" has no field
                             "suspension_kind"), plus its triggers
supabase/fix-create-campaign.sql   replaces create_campaign(); fixes "column
                             reference c is ambiguous" (unnest alias vs the
                             c campaigns record variable)
supabase/owner-identity.sql   adds my_identity() (role, status, owner match) and
                             claim_owner() matched by owner_user_id as well as
                             owner_email (for Telegram-mini-app accounts)
supabase/access-repair.sql    re-creates is_admin(), admin_role(),
                             has_admin_role() and my_context() and adds
                             my_access_debug() (the row says OWNER but the app
                             still treats the account as a worker)
supabase/user-mode.sql       set_my_mode(): stores the worker/buyer mode used by
                             the dashboard and navigation (prefs.mode)
supabase/dual-role.sql       keeps is_worker + is_buyer together and re-creates
                             set_my_mode()/enable_buyer_mode() so a dual account
                             can always switch back to worker mode
supabase/task-timer.sql      tasks.started_at + start_task() so the task window
                             start is recorded server-side
supabase/task-rls.sql        RLS + RPC grants for tasks/submissions (worker can
                             read a task and claim it), with verification output
supabase/complete-task-filter.sql   list_available_tasks() excludes a worker's
                             submitted task and unassigned-only slots
supabase/telegram-identity.sql    link_telegram_identity(): stores the OIDC
                             "sub" as profiles.telegram_id, no duplicates
supabase/telegram-events.sql   telegram_user_id_by_email() recovery RPC plus
                             "task submitted" and "withdrawal requested"
                             notification triggers/templates
supabase/announcements-feed.sql    my_announcements() feed for the in-app
                             announcement page, plus eligible delete
supabase/withdrawal-notify-bscscan.sql   paid-withdrawal template with the real
                             tx and https://bscscan.com/tx/{{tx}}
supabase/health-check.sql    prints the real state of the database: my_context()
                             result, table presence, settings rows, required
                             functions, suspension columns, then reloads PostgREST
supabase/functions/telegram-auth/
  index.ts               Edge Function: verifies Telegram initData and returns a
                         one-time token hash for a real Supabase session
supabase/functions/telegram-notify/
  index.ts               Legacy option, no longer used: the database sender
                         above delivers the same queue. Kept only as a fallback.
```

There is **no build step**: the browser loads ES modules directly. Run publishes the files as they are.

---

## 2. Setup (required, in this order)

### Step 1 — Create the database objects

1. Open your Supabase project → **SQL Editor** → **New query**.
2. Paste the whole of [`supabase/schema.sql`](supabase/schema.sql).
3. Run it. It is safe to re-run after updates: tables use `if not exists`, columns `if not exists`, functions `create or replace`, and the policies are dropped and recreated.

What it does, safely:

- **Extends** the existing `public.profiles` table in place (adds country, buyer/worker flags, admin role, status, trust score, referral fields, preferences). Existing users are kept.
- Renames `wallets` / `tasks` / `campaigns` / `services` to `*_legacy` **only** when their shape is incompatible. Nothing is deleted.
- Creates `wallets` (with TG Coin balances), `services`, `campaigns`, `tasks`, `submissions`, `ledger_entries`, `platform_ledger`, `deposits` (with on-chain verification fields), `withdrawals` (with fee/net columns), `notifications`, `support_tickets`, `ticket_messages`, `referrals`, `audit_logs`, `fraud_flags`, `settings`, `countries`.
- Sets the real deposit address `0xaa23b4c36b00a1edefb4b12449a472de56cb843f` and the tiered withdrawal fee (5% below 5 USDT, free from 5 USDT).
- Enables RLS on every table and drops any previous `profiles` / `wallets` policies (an earlier build exposed profiles to anonymous visitors).
- Creates the **private** `proofs` storage bucket with policies that require the path to start with the uploader's user id.
- Creates every server function (`SECURITY DEFINER`) and grants `execute` only to `authenticated` (plus public settings/stats to `anon`).
- Seeds 14 services, 20 countries and the platform settings.

If the script stops, read the message: it either says both `public.x` and `public.x_legacy` exist (rename one manually) or points at the failing statement. The whole editor request is one transaction, so a failure leaves nothing half-applied.

**Re-run it after this update:** the same file adds `submissions.proof_paths`, `submissions.proof_link` and `submissions.time_spent_seconds`, replaces `submit_proof` with the multi-screenshot version, creates `release_task` (give up a claim), adds the `telegram_users` table plus a `profiles.telegram_id` index for Mini App sign-in, and adds the `telegram_outbox` delivery queue, its templates and the admin Telegram functions. Existing rows keep their single `proof_path`; the Telegram module is inert until you turn delivery on.

### Step 2 — Enable sign-in

1. **Authentication → Providers → Google**: enable it and paste your Google OAuth client id/secret.
   In Google Cloud Console add the callback URL Supabase shows (`https://<project-ref>.supabase.co/auth/v1/callback`).
2. **Authentication → URL Configuration**:
   - **Site URL**: your published app URL (`https://<slug>.freemilk.app`).
   - **Redirect URLs**: add both
     - `https://<slug>--preview.freemilk.app`
     - `https://<slug>.freemilk.app`

   The app signs in with `redirectTo: window.location.origin`, so every origin you actually use must be on that list.
3. Email + password sign-in works out of the box; confirm-email is controlled by Supabase Auth settings.
4. **Telegram (optional):** the sign-in screen has a Telegram button that calls `signInWithOAuth({ provider: 'custom:telegram' })` (`telegramProvider` in `src/config.js`). Supabase accepts a custom OIDC provider identifier, so this works once you add one under **Authentication → Providers** with the identifier `telegram`. Telegram itself is not an OIDC issuer, so that provider needs a bridge (a bot plus an Edge Function that verifies the Telegram login hash and finishes the OIDC flow). Until it is configured, the button reports that Telegram is not configured and Google/email keep working.
5. **Telegram Mini App (the bot path, fully implemented).** The app also runs inside a Telegram bot. Create a bot with **@BotFather**, then deploy the included function:

   ```bash
   supabase functions deploy telegram-auth --no-verify-jwt
   supabase secrets set TELEGRAM_BOT_TOKEN=123456:AA...
   ```

   Point the bot's menu button / Mini App URL at the published app (`https://<slug>.freemilk.app`). Opening it from Telegram signs the user in automatically: `Telegram.WebApp.initData` goes to `telegram-auth`, the function recomputes Telegram's HMAC-SHA256 signature with the bot token, rejects a bad signature or anything older than 24 hours, creates the profile and wallet from the Telegram name/photo (`telegram_users` + `profiles.telegram_id`), and returns a one-time token hash the browser exchanges for a real session. Nothing is trusted from `initDataUnsafe`, and the bot token never reaches the browser. Outside Telegram the normal Google / email screen is unchanged.
6. **Telegram notifications (optional, recommended) — sent by the database, no Edge Function.** The same bot delivers task-completion, rejection, deposit, withdrawal and referral messages, but PostgreSQL sends them itself with **pg_net** and keeps the bot token in **Supabase Vault**. Open [`supabase/telegram-db-sender.sql`](supabase/telegram-db-sender.sql) and:

   1. Run **section 0** (read-only) and check that `pg_net` is listed.
   2. Enable **pg_net** (and optionally **pg_cron**) in **Database → Extensions**.
   3. Replace `PASTE_TELEGRAM_BOT_TOKEN_HERE` in **section 2** with your token and run the whole file.

   Then open **Admin → Telegram** → **Sender status**: it reads the real state (pg_net, the Vault secret, both triggers, the cron job, queue counts) and links nothing that is not installed. Turn **Bot delivery** on.

   How it works: the approval RPC (proof, deposit, withdrawal, referral) writes a message row into `telegram_outbox` in the same transaction as the wallet movement, with a unique dedupe key. An `AFTER INSERT` trigger hands that row to `net.http_post`, pg_net's worker performs the request after commit, and `telegram_reconcile()` writes the real HTTP result back into the row (`sent`, `failed`, HTTP status, `message_id`). The task-completion trigger is a **deferred** constraint trigger, so it fires at commit — after the reward and TM Coin are credited, which also makes its rank-progress line accurate. A reward can never be paid twice by a retry, and a Telegram outage can never roll a payment back.

   Retries: failed rows are retried automatically up to 5 attempts; with pg_cron installed, `telegram_tick()` reconciles and retries once a minute. Without pg_cron, **Send queued now** and the per-row **Retry** button do the same work on demand. A response that never arrives leaves the row failed with a clear reason instead of pretending it was delivered.

   **The bot token must never be put in `src/config.js`, in a database setting, or in a commit.** It lives only in Vault; if a token has ever been pasted into a chat, a message or a repository, revoke it in **@BotFather → /revoke** and update the Vault secret.

### Step 3 — Make yourself the owner

Sign in once through TM GROW (this creates your profile row), then open [`supabase/promote-owner.sql`](supabase/promote-owner.sql), change the email inside it and run the whole file in the SQL editor. The app also shows this statement on `#/admin` when a signed-in account has no role yet. The equivalent is:

```sql
update public.profiles
   set admin_role = 'OWNER', status = 'active'
 where lower(email) = lower('you@example.com');
```

Check it:

```sql
select id, email, admin_role, status from public.profiles;
```

Reload the app: an **Admin** group appears in the sidebar and the mobile menu.

**Admin access:** there is no separate `admin.html` and no hard-coded admin password. The Master Admin Panel is part of the same app (`#/admin`), behind Supabase Auth plus `profiles.admin_role`. That matters because any password written into frontend JavaScript is readable by every visitor and cannot be rotated without a redeploy.

**Worker / buyer mode.** An account can hold both capabilities. `profiles.prefs.mode` (set by `set_my_mode()` in [`supabase/user-mode.sql`](freemilk://file?path=supabase/user-mode.sql)) decides which workspace renders: buyer mode shows only the buyer hub, campaigns, campaign wallet and account pages, and route guards keep it there; worker mode shows the earning pages. A **Switch to buyer / Switch to worker** action appears in the sidebar and the mobile menu. This is presentation only — every worker and buyer RPC still checks `is_worker` / `is_buyer` on the server.

**Telegram sign-in (browser OIDC).** Telegram now publishes a real OIDC issuer (`https://oauth.telegram.org`, scopes `openid profile`, claims `sub/name/picture`, no email). To make the browser Telegram button work:

1. In **@BotFather** enable the login/OIDC settings for your bot and register the callback URI **exactly** as `https://uqynletitbcksggunubs.supabase.co/auth/v1/callback`.
2. In Supabase → **Authentication → Providers**, confirm a **Custom OIDC** provider exists with identifier `telegram`, issuer `https://oauth.telegram.org`, scopes `openid profile`, the BotFather client id/secret, and **email optional** enabled.
3. In Supabase → **Authentication → URL Configuration**, add both the preview and published app origins as redirect URLs.

The button calls `supabase.auth.signInWithOAuth({ provider: 'custom:telegram', options: { redirectTo: origin, scopes: 'openid profile' } })`; the client secret and bot token stay in Supabase/Dashboard secrets. After the session is created the app calls `link_telegram_identity()` (in [`supabase/telegram-identity.sql`](freemilk://file?path=supabase/telegram-identity.sql)), which stores the stable `sub` as `profiles.telegram_id` once and reuses it on every later login — no duplicate account and no email needed. If the callback never returns a session, the sign-in screen now shows the provider error or a "did not complete" message after 20 seconds instead of an endless loading state. Inside Telegram, the Mini App `initData` path is still used.

**The OWNER lands in the panel, not the worker dashboard.** Run [`supabase/owner-access.sql`](supabase/owner-access.sql) once: it seeds the private `owner_email` setting (`mdn29930@gmail.com`) and creates `claim_owner()`, which grants OWNER only when the verified JWT email matches that setting. After that, signing in with the owner account calls it automatically, and the app routes OWNER accounts to `#/admin` with a dedicated admin shell (admin navigation, OWNER badge, sign-out in the panel). Worker screens stay available through the sidebar's **Worker tools** switch, never as the default. An ordinary account can never promote itself: the check runs on the server against `auth.users`, not on anything the browser sends.

**If the owner still lands on the worker dashboard,** the signed-in auth account is not the configured owner — a different Google account, or a Telegram-mini-app account whose email is `tg<id>@telegram.tmgrow.app`. Run [`supabase/owner-identity.sql`](supabase/owner-identity.sql) (adds `my_identity()` and lets `claim_owner()` match `owner_user_id` as well as `owner_email`), then open `#/admin`: the page prints your auth user id, profile role, status, the configured owner and whether you match, and offers a server-checked **Grant my OWNER access** button when you do. To pin the owner by id, run `update public.settings set value = to_jsonb('<auth-user-id>'::text) where key = 'owner_user_id';` (or insert the row if it is absent), then sign in again. A matching owner never sees the worker dashboard while the role is missing: they get the repair screen instead.

> Roles live in `profiles.admin_role`. This schema has no `admin_members` table, so a missing row there cannot block admin access. `Admin → System status` shows exactly which tables exist, your current role, your auth id and the configured owner email.

### Step 3b — Apply the advanced admin migration

Run [`supabase/advanced-management.sql`](supabase/advanced-management.sql) once, after `schema.sql` (and again after any re-run of `schema.sql`). It is idempotent and adds, without deleting anything:

- **Wallet freeze** (`wallets.frozen`, reason, date) — freezes outgoing transfers only. Balances, pending rewards and the whole ledger stay untouched. `wallet_move` refuses a debit for a frozen wallet or a suspended account; admins and the service role bypass the freeze so support and finance can still work.
- **Suspension detail** — `suspension_reason_key` (one of the reported categories) and `suspension_evidence` (up to 3 screenshots/PDFs stored in the private `proofs` bucket), plus the richer `admin_suspend_user(...)` signature.
- **Internal admin notes** (`admin_notes`) — admin-only, one row per note, with the author.
- **Login history** (`login_events`) — the app reports its user agent and platform (`web` / `telegram-mini-app`) once per session; only the service role and admin RPCs read it.
- **Fraud scan** (`admin_scan_fraud`, `admin_fraud_overview`) — flags duplicate proof hashes, repeated rejections, implausibly fast completions, referral bursts and withdrawal bursts. Flags only: nothing is banned, frozen or paid automatically.
- **Richer `admin_user_detail`** — tasks, proofs with screenshots, notes, logins and the current suspension object in one call.

**Suspended accounts:** the app renders one full-screen status page (reason, dates, duration, appeal, logout) and every write RPC refuses the account — claiming, submitting, campaigns, deposits and withdrawals. Open claims are released back to the marketplace, and the wallet is never touched. Temporary suspensions with an expiry are restored by the `restore_suspensions` automation job.

If PostgreSQL reports `record "p" has no field "suspension_kind"`, that database is missing the suspension columns (an older install). Run [`supabase/fix-suspension-columns.sql`](supabase/fix-suspension-columns.sql): it adds every suspension column, the check constraint and the blocking triggers. Do **not** substitute `suspension_reason_key` — that column is the reported category, while `suspension_kind` decides temporary / permanent / under-review / restricted behaviour.

**"Database is not initialised yet" in the app** is not a table lookup: `loadContext()` calls the `my_context` RPC, and a missing function (or a stale PostgREST schema cache) surfaces as that message. After running SQL, always end with `notify pgrst, 'reload schema';` (or Supabase → Project Settings → API → Reload). [`supabase/health-check.sql`](supabase/health-check.sql) runs `my_context()` directly and prints which tables/functions/columns are missing, so the real cause is visible instead of masked. The frontend no longer hides `column … does not exist` as a setup error for this reason.

**Password not known ("Invalid login credentials")?** Run [`supabase/reset-admin-password.sql`](supabase/reset-admin-password.sql): it sets a new bcrypt password for the existing auth user, confirms the email, upserts the profile as OWNER and creates the wallet row. The password is set on the `v_pass` line at the top of the file — change it there any time and run the file again. If it reports no auth user, create the account first in **Authentication → Users → Add user** (Auto Confirm), then run it again. That file contains a password, so do not publish the repository with a real one in it: a static host serves every file, including `.sql`.

### Step 3c — Install the Master Admin Control Room (Phase 1)

Run [`supabase/admin-control-room.sql`](supabase/admin-control-room.sql) after `schema.sql`, `advanced-management.sql` and (for per-module automation switches) `automation-engine.sql`. It is additive and re-runnable:

- **Safety check first**: the file raises a clear error naming any missing core table or function instead of half-installing. It also reports whether `pg_net`, `pg_cron` and Vault are present.
- **`admin_feature_status()`** — the real status of every admin module (`active`, `needs_config`, `disabled`, `error`, `coming_soon`) computed from live database objects. The panel navigation and the overview read it, so nothing is marked Active unless it exists.
- **`admin_automation_status()`** and **`admin_automation_tick(job)`** — the automation page now runs jobs through these RPCs. The `automation-tick` Edge Function is no longer used.
- **`automation_scheduled_tick()`** — the pg_cron entry point. `automation_execute()` refuses a caller without an admin JWT, so this wrapper runs the existing function with the oldest active OWNER identity, exactly as `automation_review_submission` already does. Cron cannot run it otherwise.
- **`admin_automation_module_set(key, enabled)`** — per-module ON/OFF with an audit row (OWNER / SUPER_ADMIN only).
- **pg_cron schedules** — `tmgrow-automation-sweep` every 5 minutes and, when the Telegram sender exists, `tmgrow-telegram-tick` every minute. If pg_cron is missing the file only raises a notice; manual runs still work.

Requires **pg_cron** for scheduled runs and **pg_net** for Telegram. Enable both in **Database → Extensions**. The file never assumes they exist: it reports their state and schedules only what is available.

AI stays **rule-based** here. No provider is configured, and the panel says so instead of presenting SQL as an AI model.

### Step 3d — User Activity and Risk History (Phase 2)

Run [`supabase/admin-activity-risk.sql`](supabase/admin-activity-risk.sql) after the control-room file. Additive and re-runnable; it creates no tables.

- **`admin_user_activity(user, kind, from, to, limit, offset)`** — one timeline from rows that already exist: registration, recorded logins, task claims/submissions, proof reviews, deposits, withdrawals, campaigns, referrals, support tickets, admin audit entries and risk flags. It never reconstructs missing history; if `login_events` is absent it says so.
- **`admin_login_history(query, limit, offset)`** — sessions with the platform and user agent the app actually records. No IP address is collected, so none is shown.
- **`admin_risk_board(limit, offset)`** — accounts with open flags, reused proof hashes, repeated rejections or withdrawal bursts.
- **`admin_risk_profile(user)`** — recorded signals, an explicit rule-based score with the points behind it, flags, notes and suspension state. The score never bans, freezes or pays; restrictions stay in `admin_suspend_user` / `admin_freeze_wallet`, with audit rows.
- **Routes**: `#/admin/activity`, `#/admin/activity/<user id>`, `#/admin/risk`, `#/admin/logins`. Investigation notes use the existing `admin_add_note`, which writes an audit row.

### Step 3e — Task verification automation (Phase 6, part 1)

Run [`supabase/verification-automation.sql`](supabase/verification-automation.sql) after `automation-engine.sql` and `admin-control-room.sql`. Additive and re-runnable.

- **`automation_verification_tick(limit)`** — claims queued proofs and applies the checks the database can actually prove: a duplicate screenshot hash is rejected, an inactive worker / missing evidence / Telegram task / unknown platform goes to **manual**. Nothing is auto-approved here: approving money needs a reliable check (Telegram membership through the bot, or a configured AI provider).
- **`automation_verification_scheduled()`** — cron entry point that runs the sweep with the active OWNER identity, because `verification_claim_batch()` refuses a caller without one.
- **`admin_verification_queue()`**, **`admin_verification_decide()`**, **`admin_verification_tick()`** — the panel's queue, manual decision and manual sweep RPCs (OWNER / SUPER_ADMIN for the sweep; OWNER / SUPER_ADMIN / ADMIN for decisions).
- **pg_cron job** `tmgrow-verification-sweep` every minute. If pg_cron is missing the file raises a notice; **Admin → Task verification → Run sweep now** still works.

Every decision goes through the existing `verification_decide()` → `automation_review_submission()` → `review_submission()` path, so wallet movements and audit rows stay single-sourced. The `verification-worker` Edge Function is no longer required.

Turn on **Automation** and **Task verification** in Admin → Automation before expecting a queue: the trigger only enqueues submissions while automation is enabled.

### Step 3f — Social verification (worker + admin)

Run [`supabase/social-verification-frontend.sql`](supabase/social-verification-frontend.sql) after `socify-integration.sql`. Additive and re-runnable.

- **Worker → Linked accounts → Public profile check**: for Instagram, TikTok and YouTube accounts, a **Check public profile** button starts `social_profile_check()` (which calls the existing `socify_lookup`), then polls `social_profile_finish()` until the result is stored. It shows name, username, avatar, followers, posts, verified/private and an explicit status: Not checked / Checking / Public profile / Needs review / Rejected / API error. It is labelled as **public data, not ownership**; the verification-code flow is unchanged.
- **Worker → My tasks**: each task shows its verification method and status (pending, manual review, approved, rejected, error) plus the reason when one exists.
- **Admin → Social verification** (`#/admin/social-verification`): public-profile queue with counts, detail drawer containing the stored API response and lookup history, and manual Verify / Reject with a mandatory reason (audited, worker notified).
- **Admin → Socify API** (`#/admin/socify`): key configured, pg_net state, service ON/OFF, worker access, limits, credits today/month, recent requests and a Retry button for failed calls.
- **Admin → Tasks**: a **Verification** button per task sets that campaign's verification method (manual / public profile / minimum followers / public media URL) and rules. Auto-approval is always stored as `false`; Socify never approves Follow, Like, Subscribe, View or Comment actions.

### Step 4 — Fill in the money settings before taking orders

Open **Admin → Platform settings** and set at least:

- `deposit_address` — the BEP-20 address buyers send USDT to. Seeded with `0xaa23b4c36b00a1edefb4b12449a472de56cb843f`; change it here if you move funds.
- `support_email` — used in support copy.
- `withdrawal_min`, `min_deposit`, `campaign_fee_pct` — defaults are 2, 5 and 10.
- `announcement` — optional banner on the landing page.

These are stored in the database and read live by the worker and buyer apps.

### Step 5 — Run

Tap **Run** to publish the preview and open it in the chat browser. Nothing else is needed.

---

## 3. How the money works

```
Buyer deposit (on-chain verified)  →  buyer available balance
Buyer creates campaign  →  budget + fee leave available
                           budget → wallet.reserved + campaign escrow (reserved_remaining)
Worker claims task      →  submission moves that reward to worker.pending
                           and takes it out of campaign escrow
Verifier approves       →  worker.pending → worker.available (+ lifetime earned)
                           buyer escrow is spent, campaign approved_count++
Verifier rejects        →  worker.pending reversed, reward back to campaign escrow
Campaign completed      →  unused escrow refunded to the buyer
Worker withdraws        →  amount moves available → reserved (held)
Finance approves        →  status approved → processing
Finance pays on-chain   →  a real 0x…64 transaction hash is required
                           reserved is cleared, lifetime withdrawn++
Finance rejects         →  amount returns to available
```

Every movement above writes an immutable `ledger_entries` row with balances after the move. Fees and referral payouts are recorded in `platform_ledger`.

**Deposits:** the deposit page always shows the platform address `0xaa23b4c36b00a1edefb4b12449a472de56cb843f` (with a copy button). The user sends USDT on BEP-20, enters the amount and transaction hash, and presses **Confirm payment**. `src/lib/bsc.js` reads the real receipt from a public BNB Smart Chain node, checks the USDT contract, the recipient address, the amount, the success status and 12 confirmations, and only then calls `submit_verified_deposit`, which credits the wallet once per hash. A hash that is not a real USDT transfer to the address is never credited.

Browser-side verification is real, but a determined user could call the RPC directly with a forged amount. For a large platform, add an admin review step or a server-side (Edge Function) re-check of the same transaction before release.

**Withdrawals:** the withdraw page shows available balance, amount, the fixed BEP-20 network and the tiered fee — 5% below 5 USDT, free from 5 USDT (settings `withdrawal_fee_pct` and `withdrawal_fee_free_min`) — plus the net amount, then a confirmation dialog. Requests move through Pending → Approved → Processing → Paid, or Rejected with the amount refunded, and every step is visible with its transaction hash.

**Earn home** shows the level/tier strip, USDT balance, TG Coin balance, tasks done, a trust score out of 1000 with tier progress, a daily check-in (TG Coin, streak preserved), and a live payouts list built from real paid withdrawals (`recent_payouts`), never simulated names.

**Profile photo:** there is no avatar URL field. `ensure_profile()` and `refresh_my_identity()` copy the name and photo from the Google account used to sign in, and *Profile → Sync account* refreshes them.

**Task runner:** a claimed task opens a modal with the target link, a 30-second countdown, an optional proof link, an optional note and a drag-and-drop zone for up to 6 screenshots (JPG/PNG/WebP, 5 MB each). Submit stays disabled until the countdown ends and at least one screenshot is attached. `submit_proof` stores the paths in `submissions.proof_paths` (the first one is also kept in `proof_path`), the proof link, the seconds spent on the task and the hash of the first screenshot for duplicate detection.

**Telegram notifications:** proof approval/rejection, deposit approval/rejection, withdrawal approval/payment/rejection, referral bonuses and (optionally) announcements are copied to Telegram when the user has a linked Telegram account and delivery is on. The message text comes from editable templates in **Admin → Telegram**, filled with values read back from the committed database row — never from the browser. The wallet is always credited first; the message is a copy, not the payment. Failed deliveries are logged with the provider's reason and can be retried without touching balances.

**Task controls:** **Admin → Tasks → Add task slots** generates extra claimable slots on a funded campaign, capped at the campaign's remaining quantity so it can never promise more rewards than the escrow holds. Individual available or cancelled tasks can be disabled and re-published; a task that already has a decision is final.

**Maintenance mode:** *Admin → Overview* has a switch (OWNER / SUPER_ADMIN) that sets the public `maintenance` setting. Every non-admin then gets a blurred "We'll be back soon" screen; the sign-in page stays reachable so an admin can get in and turn it off. The check runs on every route, and a user's browser picks up a flip when it reloads or when they press **Try again**.

---

## 4. Roles and permissions

| Role | Can do |
| --- | --- |
| `OWNER` | Everything, including granting admin roles and all settings |
| `SUPER_ADMIN` | Everything except changing admin roles |
| `ADMIN` | Users, campaigns, tasks, proofs, services, countries, settings, audit |
| `MODERATOR` | Users, campaigns, tasks and proof review |
| `VERIFIER` | Proof verification only |
| `SUPPORT` | Support inbox, announcements, read-only finance |
| `FINANCE` | Deposits, withdrawals, ledger, balance adjustments |

The sidebar hides what a role cannot use, and every RPC checks the role again on the server. Admin actions write `audit_logs` rows.

---

## 5. The AI assistant

`Admin → AI assistant` calls `admin_ai_snapshot(p_days)` — a server function that returns real totals, finance figures, the proof queue, withdrawal queue, support queue, open fraud flags and top services.

- **Default behaviour (no key):** a deterministic analyst answers from that snapshot. It has no external dependency, cannot invent numbers, and works offline of any AI provider.
- **Optional:** connect an OpenAI-compatible endpoint (`base URL`, `model`, API key) in the browser. The key is stored in this browser's `localStorage` only, and the snapshot JSON is sent with each question. If the provider blocks browser requests (CORS) or fails, the app falls back to the computed answer and says so.
- The assistant **cannot** approve proofs, send payouts, adjust balances or change roles. Those actions live in their queues, require explicit confirmation dialogs, and are permission-checked again by the database.

If you later want a server-side AI proxy (no key in the browser), that needs a server component; the project is intentionally a plain static site today.

---

## 6. Verification status

This project has **not been executed or tested** from the build environment: there is no way to run a browser, a Node process or the Supabase SQL here. What was done instead:

- Every screen was written against the real RPC signatures and return shapes in `supabase/schema.sql`.
- All module imports/exports were cross-checked (`src/app.js` ↔ views ↔ lib).
- Every `rpc(...)` name and every table read used by the frontend exists in the schema and is granted to `authenticated`.
- The SQL was reviewed for the escrow, pending, refund, payout and referral paths; a cancellation bug that would have stranded reversed pending rewards in the buyer's reserved balance was found and fixed (`set_campaign_status` now refunds `reserved_remaining + reversed pending`).
- The Telegram module was reviewed for the two properties that matter: a message row is written in the same transaction as the wallet movement, and its dedupe key is unique per event (`task_completed:<submission id>` and so on), so retries cannot double-post or double-pay. The database sender in `supabase/telegram-db-sender.sql` has not been executed against a database from here, so its first real run and first delivered message are unverified.

Expect to spend a short session on first-run checks: sign in, promote the owner, set the deposit address, create one small campaign, submit one proof, approve it, request a withdrawal and mark it paid. That single pass exercises every ledger path.

---

## 7. Deliberately out of scope

- Automatic proof verification (the UI says "human review" everywhere).
- On-chain deposit/payout automation (needs an indexer and a custodial wallet — a paid, security-critical service).
- Email or push notifications (in-app notifications only, plus the optional Telegram bot described above).
- Telegram sign-in has two paths. Inside the bot (Mini App) it is fully implemented by `supabase/functions/telegram-auth` and works as soon as the function is deployed with `TELEGRAM_BOT_TOKEN`. On the open web the Telegram button calls Supabase's `custom:telegram` provider, which needs an OIDC bridge because Telegram is not an OIDC issuer, so it stays inactive until you add one; Google and email are unaffected.
- Any real TG Coin chain contract: TG Coin is a platform-internal bonus balance with no cash value.
- A server-side AI proxy.
