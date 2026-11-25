# @deckyard/oauthext

[![test](https://github.com/deckyard/deck-oauthext/actions/workflows/test.yml/badge.svg)](https://github.com/deckyard/deck-oauthext/actions/workflows/test.yml)
[![Documentation](https://img.shields.io/badge/docs-github%20pages-blue)](https://deckyard.github.io/deck-oauthext/)
[![release](https://img.shields.io/github/v/release/deckyard/deck-oauthext)](https://github.com/deckyard/deck-oauthext/releases/latest)

Vendor-specific OAuth2 for the bundled `oauth` module of
[Jennifer](https://jennifer-lang.dev/): `Config` presets for Google™ and
Microsoft Entra™ ID, and Google service-account logins from a JSON key file
(the RFC 7523 JWT bearer grant), which open Google Cloud™ and Workspace™ APIs to
a program with no user. The presets were `oauth.google` / `oauth.microsoft` in
the bundled module up to Jennifer 0.25.

Up to 0.3.0 this deck was `@jennifer/oauthext`. To switch, change the `[decks]`
entry to `"@deckyard/oauthext" = "^0.4.0"` and the imports to `"@deckyard/oauthext/"`.

**[Documentation](https://deckyard.github.io/deck-oauthext/)**

## Install

```sh
jvc add @deckyard/oauthext
```

This vendors the deck to `vendor/<scope>/oauthext/`. Without `jvc`, unpack the
`src/` subtree of a release archive (`<scope>-oauthext-<version>.tar.gz`) there.

## Usage

```jennifer
use fs;

import "oauth.j" as oauth;
import "@deckyard/oauthext/" as oauthext;

def tok as oauth.Token init oauthext.googleServiceAccount(
    fs.readString("service-account.json"),
    "https://www.googleapis.com/auth/cloud-platform");

def cfg as oauth.Config init oauthext.microsoft("common", $clientId, "",
    "https://outlook.office.com/IMAP.AccessAsUser.All offline_access");
```

## Requirements

Jennifer 0.27.0 or newer, the default `jennifer` binary (the deck needs
`net`; the stock `jennifer-tiny` has no network driver). RS256 signing for
service accounts is in the default binary.

## Development

```sh
tools/check-all.sh                   # the full gate, as CI runs it
jennifer test src/oauthext_test.j      # the test overlay
tools/check-docs.sh                  # docs against the code
tools/pack.sh dist                   # build the release archive
```

## Licence

LGPL-3.0-only, see [LICENSE](LICENSE). Copyright (C) 2026 mplx &lt;mplx@jennifer-lang.dev&gt;.

`tools/check-headers.sh` checks every source file's SPDX header against
`deck.toml`.

## Trademarks

Google™, Google Cloud™ and Google Workspace™ are trademarks of Google LLC.
Microsoft™ and Microsoft Entra™ are trademarks of Microsoft Corporation. Other
product and company names may be trademarks of their respective owners. They
are used only to identify the products this deck works with. This deck is an
independent project, not affiliated with, sponsored or endorsed by Google LLC
or Microsoft Corporation.
