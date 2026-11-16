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
