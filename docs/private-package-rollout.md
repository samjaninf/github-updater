# Private Package Rollout: enforcing the REST API gate

Git Updater withholds private repositories from every REST API route unless the caller
presents the shared REST API key or an authorized Lite client domain for that slug.
That gate is described in [lite-update-flow.md](./lite-update-flow.md). This note covers
how to turn it on without breaking sites that are still running an older client.

## Why order matters

`git-updater-lite` is a **Composer library vendored into a consuming plugin or theme** —
it is not a plugin that updates itself. A site only gets a newer client when the
*consuming* plugin ships a new bundle and the site updates that plugin.

Consequently, enforcement is server-side but the fix is client-side. If you enforce
before your users have the newer client, a private package stops updating, and the
failure is **silent**: on a non-200 the client returns before registering any hooks, so
there is no update row, no "View details" link, and no negative caching. Nothing appears
in the admin to explain it, and because the client is what fetches `update-api`, an
affected site cannot pull down its own fix.

## Required client version

| Client | Behaviour |
|---|---|
| `git-updater-lite` ≥ 3.1.0 | Sends `X-GU-Site-Domain` on the `update-api` request. Passes the gate when its domain is authorized. |
| `git-updater-lite` < 3.1.0 | Sends no headers on `update-api`. Denied for private packages. |

No released client sends any header on `update-api` before 3.1.0, so **the presence of
`X-GU-Site-Domain` on that request is itself the signal that a caller has the updated
client.** You do not need a separate version marker to check coverage.

## Recommended order

1. **Publish `git-updater-lite` 3.1.0.** The header is inert against an older server —
   servers ignore unknown request headers — so this is safe ahead of enforcement.
2. **Re-vendor and ship.** Update the consuming plugin/theme's `vendor/` copy and release
   it. Your customers pick up the new client when they update that plugin/theme.
3. **Check coverage.** Before enforcing, log the inbound `X-GU-Site-Domain` header on
   `update-api` requests. Requests carrying it are clients that have the update; requests
   carrying nothing are either legacy clients or non-Lite consumers.
4. **Enforce.** The gate is on by default in this release. If your fleet is not ready,
   see the escape hatch below.

## Escape hatch

Return `false` from `gu_enforce_private_package_gate` to serve private metadata
unauthenticated while you finish the rollout:

```php
add_filter( 'gu_enforce_private_package_gate', '__return_false' );
```

The filter receives the package slug as its second argument if you need to stage this
per package:

```php
add_filter(
	'gu_enforce_private_package_gate',
	function ( $gate, $slug ) {
		return 'my-plugin' === $slug ? false : $gate;
	},
	10,
	2
);
```

It applies to both `get_api_data()` and `get_download_token()`, so a staged rollout
cannot leave the two endpoints disagreeing. Default is `true`.

Treat this as a **stopgap, not a steady state** — while it returns `false`, private
package metadata is readable by anyone who knows the slug. Note also that the
`private_package` hard block on the Additions tab is **not** filterable; it always denies.

## Caveat: one site, several consumers

`Lite` is defined behind a `class_exists()` guard, so **whichever consuming plugin or
theme loads first defines the class for the whole site.** A site with two plugins that
each vendor a different copy can hold a 3.1.0 bundle in one and have an older copy win.

The practical effect: updating a single plugin may not be enough for such a site, and the
symptom is confusing — "one plugin updated fine, another silently stopped". If you ship
to customers who bundle more than one of your packages, either ship them together or
point affected users at the escape hatch above until every consumer is updated.
