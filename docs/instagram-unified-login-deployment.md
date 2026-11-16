# Instagram Unified Login deployment

## Source and scope

Imported from [Chatwoot PR #16645](https://github.com/chatwoot/chatwoot/pull/16645), head `8e5c639ed4d3afa3c91729047dd32153754bd2bd`, against base `d943f045c6e17e53ad334981d6983bd1ad867617`. Applied to this fork at `aed04e65f5c088d42c2b85304293e47367b6fe96` without unrelated upstream commits. The PR was open when inspected on 2026-11-16; this is a custom build, not a released upstream feature.

Adds Unified Login credentials, account selection, API routing, webhook signature support and a database migration. Existing Instagram inboxes retain their login method. Migration of existing inboxes from another login method is outside this change. No dependency manifests changed.

## Before production

- Run changed backend specs, including Enterprise inbox limits, with Ruby 3.4.4, PostgreSQL and Redis. Run changed frontend specs and lint with Node 24 and pnpm 10.2.0.
- Build the production image from `docker/Dockerfile`, tag with the actual fork commit, and use the same image for Rails and Sidekiq. Keep the previous image digest for rollback. Do not use the official `chatwoot/chatwoot:latest` image for this custom feature.
- Rehearse on staging with a backup of the production database. This fork's develop branch may include changes beyond the currently deployed release; review and test those migration differences before upgrading.
- Back up the database and preserve storage volumes, Redis configuration and existing secrets. Confirm a restore procedure.
- Both app and worker need the existing stable `SECRET_KEY_BASE` and the three Active Record encryption settings: `ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY`, `ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY`, `ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT`. Preserve any existing values. Generate missing values privately using Rails' encryption initialization task; never commit them. Pending OAuth tokens use encrypted Redis storage and expire after 10 minutes.
- Run `RAILS_ENV=production bundle exec rails db:chatwoot_prepare` with the custom image before starting the upgraded app/worker. The imported migration adds `auth_type` and `configuration_id` and allows a null token expiry. Check migration logs and both service health checks.

## Meta and Super Admin configuration

In Super Admin → App Config → Instagram:

| Setting | Value |
| --- | --- |
| Login method | Unified Login |
| Meta App ID | `1812535373408044` (Channel; confirm against the selected Meta app) |
| Meta App Secret | Secret belonging to that same Meta app; enter privately |
| Configuration ID | The Instagram Unified Login / Facebook Login for Business configuration ID from that app |
| Verify token | A private value matching the Instagram webhook verification token in Meta |

Do not substitute the WhatsApp signup configuration ID for the Instagram configuration ID. Keep existing Facebook/Messenger settings intact.

- OAuth redirect URI: `https://workspace.multiplusonline.nl/instagram/callback`
- Instagram webhook callback: `https://workspace.multiplusonline.nl/webhooks/instagram`
- Existing Messenger callback remains `/bot`.
- Configure the Meta app's Instagram Unified Login use case and relevant messaging access. Verify the webhook challenge with the matching token and subscribe to supported message events. The connection service subscribes the selected account to `messages,message_reactions,messaging_seen`; this does not imply that every event has a corresponding UI feature.
- Create an Instagram inbox, authorize the intended professional account, select it in the account picker, and assign agents. A successful OAuth redirect alone is not proof of working messaging.

## Acceptance checks

Using an authorized test account, verify one inbound message and one outbound reply, no duplicate echo, profile retrieval, a media message, and read status where supported. Test cancellation, expired account selection, and reauthorization of the same inbox. Confirm existing Messenger send/receive still works. Inspect app/worker logs without recording access tokens or app secrets.

If rollback is needed, stop new Instagram onboarding first. Do not blindly reverse the migration after creating non-expiring-token inboxes: the old NOT NULL expiry constraint may no longer be satisfiable. Rehearse rollback against a snapshot; restoring a database snapshot after live writes can lose subsequent messages.

## Local verification limits

The patch applies cleanly and passes `git diff --check`. Ruby syntax checks pass for all 35 changed Ruby files using Ruby 4.0.1; this is not a substitute for tests on the required Ruby 3.4.4. Backend tests and image build are blocked locally by the missing required Ruby runtime and denied Docker socket access. Production configuration and end-to-end Instagram messaging remain unverified.

Frontend verification: all 27 tests across the five changed spec files passed, and ESLint passed for all changed JavaScript/Vue files. These ran on available Node 25.8.1 with pnpm 10.2.0; the project targets Node 24, so production validation should use Node 24. Dependencies were installed from the frozen lockfile without lifecycle scripts. Non-fatal warnings concerned Browserslist data and an existing package source map.

## Deployment progress — 2026-11-16

Production target is now `deploy/instagram-v4.19.1` at `937607cc82d7335398f81873b4793e988aef4e72`, based on the deployed release. It contains only the Instagram migration; it does not include the ten unrelated develop migrations. The migration uses Active Record 7.2 for release compatibility. The develop-based feature branch remains available separately.

Production-target validation passed: 113 backend examples, 27 frontend tests, lint, and loading the v4.19.1 schema then migrating it on PostgreSQL 12. Build: https://github.com/multiplusonline/chatwoot/actions/runs/43882326786 . Image publication succeeded. Verified anonymous registry access and pinned image: `ghcr.io/multiplusonline/chatwoot:instagram-937607cc82d7335398f81873b4793e988aef4e72@sha256:3314152083e6778970d44114d54a8d7718c3b0781527fd0dda89dda0a75795c8`. The older develop image build was cancelled intentionally.

One-time database backup created and archive catalog verified: `/var/lib/postgresql/data/manual-backups/chatwoot-pre-instagram-20261116T140434Z.dump` (425 KB, mode 600). This is on the database volume, not off-server storage. No recurring schedule was created.

Coolify compose now supplies the three Active Record encryption settings to both Rails and Sidekiq through generated, reusable `SERVICE_PASSWORD_64_CWENCRYPTPRIMARY`, `SERVICE_PASSWORD_64_CWENCRYPTDETERMINISTIC` and `SERVICE_PASSWORD_64_CWENCRYPTSALT` variables. No secret values are recorded here. Both image references are saved to the pinned custom image. Deployment completed. Runtime revision is `937607cc82d7335398f81873b4793e988aef4e72`; no pending migrations; both new Instagram columns exist; all three encryption variables are present. Rails and Sidekiq report healthy, and the public API returns HTTP 200. PostgreSQL and Redis images were not pulled or upgraded.

Meta configuration **Chatwoot Instagram Login**, ID `1698393818310865`, is created with Instagram assets and exactly `instagram_basic` and `instagram_manage_messages`. Token lifetime is 60 days. Both permissions are ready for testing; Advanced Access approval is not established. OAuth redirect URI is saved and selected. Final account authorization, webhook verification, and live messaging remain outstanding.

Rollback image: `chatwoot/chatwoot:v4.19.1@sha256:a0d571a3a2f78f4fa4b1d7c21a53309c0184068d6021b3664ab37788faef04c1`.

Unified Login settings are saved in Chatwoot, reusing the existing Channel app secret privately. Meta accepted the Instagram webhook verification and subscriptions for `messages`, `message_reactions`, and `messaging_seen` (v27.0). The Meta messages sample delivery test succeeded. This verifies transport, not live inbox messaging. Authorization for `multiplusonline` in the Multiplus BV portfolio succeeded, but creating the inbox is blocked by Meta account subscription (details below).

## Remaining Meta blocker — 2026-11-16 14:50 UTC

No Instagram inbox was created. Meta returns HTTP 400, GraphMethodException code 100/subcode 33, for POST `/17841429480492119/subscribed_apps`. The same failure was reproduced directly using Meta’s documented form-body request, outside the Chatwoot service. GET of the account succeeds and returns username `multiplusonline`. Token debugging confirms app ID `1812535373408044`, valid SYSTEM_USER token, and granted `instagram_basic` and `instagram_manage_messages` scopes. No credentials are included here.

Changed Instagram API version from the inherited v22.0 to v27.0 and retried; same subscription error. Wichard is an app Administrator. Instagram Testers list is empty; no tester invitation was sent. Advanced Access remains unverified; the two permissions are ready for testing. Investigate tester eligibility/Instagram message access and Meta app access restrictions; the exact cause of code 100/33 is not established. Do not bypass subscription failure or create a working-looking inbox manually.

Direct POST failure trace ID: `ATRp_oPVgvYGeYSk1SiKey6o` (14:49 UTC). Use this, app/config IDs and the account ID for Meta support if eligibility checks do not resolve the error. Live Instagram inbound/outbound messaging is NOT verified.

Existing Messenger inbox 1 remains present, its page API responds HTTP 200, and `reauthorization_required` is false. This is a connection health check, not a new send/receive test. WhatsApp settings remain untouched.
