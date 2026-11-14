# tacita-dist

Public distribution channel for Tacita's bundled Pi agent binary. Kept separate
from the private Tacita app repo on purpose: this repo is the **approval gate**
between upstream [earendil-works/pi](https://github.com/earendil-works/pi)
releases and what Tacita ships/updates to.

## What lives here

- `pi-channel.json` — the channel manifest. A single `approved` entry points at
  exactly one upstream pi release, with per-platform asset names and SHA256
  digests copied verbatim from that release's `SHA256SUMS`.

## What does *not* live here

Binaries. The manifest references upstream release assets directly; the digests
in this file are what clients verify against, so upstream hosting changes or
tampering is detected client-side.

## Approval process (the point of this repo)

1. A new pi release appears upstream.
2. Test it against Tacita: RPC bridge smoke test, bundled skills
   (wayfinder/grilling), tool policy (no bash), rg/fd pre-staging.
3. If good: update `pi-channel.json` `approved` to the new tag + digests from
   upstream `SHA256SUMS`, set `minTacita` if the pi release requires a newer
   Tacita bridge, add a `notes` line, commit.
4. Tag the manifest revision: `git tag channel-vN && git push --tags`. Tags are
   the immutable audit trail of what any Tacita release ever saw.

If a release turns out bad after approval: move `approved` back to the known
good version, commit, tag. Clients revert on their next channel check
(automatic updates only apply an approved version newer than what is installed;
a pointer moving backwards is honored as a rollback).

## Schema (v1)

```jsonc
{
  "schema": 1,
  "approved": {
    "version": "1.5.2",              // upstream semver
    "tag": "v1.5.2",                 // upstream release tag
    "release_base": "…/download/v1.5.2",
    "assets": {
      "<rust-target-triple>": { "asset": "<file>", "sha256": "<digest>" }
    }
  },
  "minApp": "0.1.0",                 // client-app version floor for this release
  "severity": "normal",              // "normal" | "critical" (forced upgrade)
  "notes": "…"
}
```

The manifest schema is deliberately app-agnostic (`minApp`, not
`minTacita`) so other applications can run their own channels against the
same `release-channel` crate without schema drift.

Clients fetch:

```
https://raw.githubusercontent.com/neiljohnston/tacita-dist/main/pi-channel.json
```

Any fetch/parse/validation failure is a silent no-op client-side — the binary
bundled with the Tacita installer remains the floor. Rollback via tag-pinned
raw URLs (`/tag/channel-vN/pi-channel.json`) is for forensics, not runtime.

## Future

A `signature` field (ed25519 over the canonical manifest JSON, pubkey embedded
in Tacita) can be added without a schema break when tamper-resistance beyond
git-history integrity is needed.
