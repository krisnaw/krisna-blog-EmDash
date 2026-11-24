# EmDash upgrade work order

This file is the handoff for upgrading the project at `/workspace` from EmDash **1.0.1** to **1.6.0**. The direct packages were bumped together, the EmDash agent skills were refreshed, and the core migration manifests were compared. Application code was assessed against the crossed release entries. This file does not apply a database migration or deploy the site.

## Package changes

- `emdash`: `^1.0.1` (1.0.1) → `^1.6.0` (1.6.0)
- `@emdash-cms/cloudflare`: `^1.0.1` (1.0.1) → `^1.6.0` (1.6.0)

Astro and other peer dependencies already satisfy EmDash 1.6.0 (`astro >= 6`).

## Project assessment (required actions)

Assessed against this Cloudflare Workers blog (D1 + R2, no sandboxed plugins, no Cloudflare Access, no custom Portable Text block renderers, no `reference` fields, co-editing left off):

| Release note | Applies? | Action |
| --- | --- | --- |
| Co-editing Durable Object wiring (1.5.0) | Optional feature, off by default | None — do not enable without adding `EMDASH_ROOMS` / `EmDashRoom` exports |
| Plugin KV shared-store migration (1.5.0) | No sandboxed/marketplace plugins in config | None beyond normal core migrations |
| `095_plugin_storage_unique_scope` maintenance window | No plugin storage unique indexes in use | Normal deploy is fine; still take a D1 backup |
| Reference field → relation binding | Seed has no `reference` fields | None |
| Custom `html` / `iframe` / `video` renderers | Site uses stock `PortableText` | None |
| `getSeoMeta` + non-`title` `titleField` | Collections use `title` | None |
| External auth `defaultRole` hover-text fix | Passkeys only | None |
| Menu orphan promotion (1.5.0) | Possible if nested items were deleted earlier | After deploy, review menus in admin |
| Media URL usage tracking fix | Media usage may omit PT/HTML/string URL links | After deploy, run media usage repair (below) |

No application source or Wrangler config changes are required for a compatible upgrade.

## Core database migrations

Installed EmDash changed from `1.0.1` to `1.6.0`.

Added core migrations:

- `090_redirect_enable_loop_guard`
- `091_redirect_artifacts`
- `092_plugin_kv_options`
- `093_menu_item_orphans`
- `094_media_usage_file_url_reindex`
- `095_plugin_storage_unique_scope`
- `096_revision_kind_and_editors`
- `097_room_storage`
- `098_user_last_login`

Build the upgraded project and inspect the exact database target:

```sh
npm run build
npm exec -- emdash migrate --status
```

If the status reports pending migrations, stop before starting or deploying the upgraded application. Follow [Backups and recovery](https://docs.emdashcms.com/guides/backups/) to create a restorable recovery point (D1 export). A JSON backup is not a recovery point. After a human has reviewed the target and backup, either:

- Deploy with default `auto` runtime migrations (pending migrations apply on first request; large content migrations may return 503 until `emdash migrate` finishes), or
- Apply first, then deploy:

```sh
npm exec -- emdash migrate --wrangler-config wrangler.jsonc
npm run deploy
npm exec -- emdash migrate --check --wrangler-config wrangler.jsonc
```

### After deploy: media usage repair

Media usage tracking now counts `/_emdash/api/media/file/` links in Portable Text, HTML blocks, and string/URL fields. Existing entries are not re-indexed automatically. After the upgraded Worker is live:

```sh
npx emdash media repair-usage --all --url https://blog.krisnawijaya.com --json
```

Requires an Admin auth token. See [Update EmDash → Fixed: media URL link usage](https://docs.emdashcms.com/deployment/updating/#fixed-media-url-link-usage).

## Verification

1. `npm run typecheck` and `npm run build`
2. Inspect migration target / pending names before applying them to production D1
3. After deploy: public site, admin sign-in, edit/publish, media upload/read
4. Review menus; run media usage repair
