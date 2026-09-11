# Git Updater Lite: New Download Flow & Security Features

This document explains the recent changes to how `git-updater-lite` handles plugin and theme updates. We have moved to a **Two-Step Download Flow** with **Domain Validation** to fix reliability issues and add security features for private packages.

## The Problem: "Why were my updates failing?"
Previously, when `git-updater-lite` checked for an update, the server provided a download link that was valid for **12 hours**. However, `git-updater-lite` caches this information for **6 hours** to prevent slowing down your site.

If a user waited longer than 12 hours to click the "Update Now" button (even though the update notice was still cached), the download link would expire, resulting in a `403: Download link has expired` error. 

## The Security Goal: Keeping Credentials Safe
Before diving into the solution, it's important to understand *why* we use a proxy at all. If you host private repositories on GitHub, GitLab, or Bitbucket, your server requires an **Access Token** (a secret key) to download the files.

If we sent this Access Token directly to the user's WordPress site to handle the download, we would risk exposing it. If the user's site was hacked, or if they viewed the network traffic, they could steal your private Access Token and use it to access your other private repositories.

To solve this, `git-updater` acts as a **secure middleman**:
1. The user's site asks for the update.
2. Your `git-updater` server uses its **private, stored Access Token** to fetch the file from GitHub/GitLab.
3. The server immediately streams the file to the user's site, **without ever sending them the Access Token**.

## The Solution: The Two-Step Download
We have fixed the reliability issue by changing *how* the download happens behind the scenes, while maintaining this high level of security.

Instead of receiving a long-term download link, `git-updater-lite` now receives a **"Token URL"**. 
1. When you click "Update Now", the plugin uses this Token URL to instantly request a **fresh, 60-second download link** from the server.
2. The server verifies the request and, if valid, generates a temporary link to its secure proxy endpoint.
3. The plugin uses that fresh link to download the file.

Because the download link is generated *at the exact moment* you update, it can never expire before you use it. This completely eliminates the cache mismatch error while ensuring the upstream access tokens remain perfectly safe on your server.

## Security Feature: Domain Validation
For developers distributing **private packages** (plugins/themes that require authentication), the server enforces **Domain Validation**.

A package is treated as private based on what the git host reports about the repository — this is detected for every supported host (GitHub, GitLab, Gitea, Bitbucket, Bitbucket Server). Domain validation is therefore **not** a flag you switch on per package; it applies automatically to every private package.

### How it works
The server checks the **domain name** of the website requesting the update (e.g., `example.com`). A private package is withheld unless the requesting site matches the "allowed list" you have configured, or it presents the shared REST API key.

> **Private packages need at least one configured domain.** If a private package has no domains configured, it is withheld from the API — including git-updater-lite clients. Add its client domains on the **Lite Client Domains** tab, or they will stop receiving updates.

### Smart Subdomain Handling
You only need to add your base domain (e.g., `example.com`) to the allowed list. The system will automatically accept updates from:
- `www.example.com`
- `staging.example.com`
- `dev.example.com`
- Any other subdomain

## Who is Affected?

### For `git-updater-lite` Users (The Client)
**You don't need to do anything** for public packages.
The update process is now more reliable. When you click "Update Now", you might notice a tiny, split-second delay as the plugin requests the fresh download token, but it is virtually unnoticeable. Public repositories will continue to update exactly as they always have.

A package must carry an `Update URI:` header to be served to a lite client — that header is how the client finds its update server.

### For `git-updater` Developers (The Server)
If you host private packages for your clients, you now have a new **"Lite Client Domains"** tab in your Git Updater settings.
- **Automatic Detection**: The system will automatically scan your private repositories and suggest adding them to the domain list.
- **The "Uses Git Updater Lite" Checkbox**: In the **Additions** tab, you can check a box labeled "Uses Git Updater Lite" for any package. This flags it for distribution to lite clients. It does not make a package private — privacy comes from the repository itself.

## What is NOT Affected?

- **Public Repositories**: If your plugin or theme is public, domain validation does not apply. Updates will work for everyone, everywhere.
- **The Main Git Updater Plugin, public packages**: Public packages keep returning direct download links with no credential required.
- **Hard-Blocked Packages**: If you marked a package as a "Private Package" on the Additions tab, that setting overrides everything else — the package is never served to anyone, whether or not a key or domain is presented.

## What DID Change

- **Private packages now require a credential on every API route**, including `plugins-api`, `themes-api` and `update-api`. Previously only a "Private Package" hard block was enforced, so a private repository that was never added on the Additions tab was served like a public one. Callers must now present the shared REST API key or an authorized lite client domain.
- **git-updater-lite must be 3.1.0 or newer** for private packages. From 3.1.0 the client sends `X-GU-Site-Domain` on its `update-api` request; earlier clients send no headers on that route and are denied once the gate applies. See `docs/private-package-rollout.md` for the enforcement order.
- **The key reset on the Remote Management tab is POST-only** (nonce and capability were already required).

## Summary
- **Security First**: Upstream Access Tokens are never exposed to the client.
- **Reliability**: Updates no longer fail due to cached, expired links.
- **Security for Clients**: Private packages are locked to specific customer domains.
- **Simplicity**: Public repos need no configuration.