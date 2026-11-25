<!-- SPDX-License-Identifier: LGPL-3.0-only -->
<!-- SPDX-FileCopyrightText: Copyright (C) 2026 mplx <mplx@jennifer-lang.dev> -->

# @deckyard/oauthext

Vendor-specific OAuth2 for the bundled [`oauth`](https://jennifer-lang.dev/modules/oauth.html)
module. `oauth` keeps the standard mechanics: the client-credentials (with a
secret or a client assertion), refresh, device, authorization-code with PKCE
and JWT-bearer grants, endpoint discovery, and token storage. What belongs to one provider
lives here, so a new provider is a deck change, never a change to the
interpreter:

- **Presets**: `oauth.Config` values for Google™ and Microsoft Entra™ ID, for
  the browser sign-in, the device flow and refresh (mail through IMAP and
  SMTP, Microsoft Graph™, Google APIs as a user), including Entra's
  multi-tenant endpoints, which `oauth.discover` cannot read.
- **Google service accounts**: a server logs in as a service account from its
  JSON key file, with no user and no browser. This opens every Google Cloud™
  API (Cloud Storage, BigQuery™, Pub/Sub) and, with domain-wide delegation,
  Google Workspace™.

The presets were `oauth.google` and `oauth.microsoft` in the bundled module up
to Jennifer 0.25; [Moving from oauth](migrating.md) shows the change.

## Conventions

Snippets assume these imports, and add the `use` lines and other imports
they need:

```jennifer
import "oauth.j" as oauth;
import "@deckyard/oauthext/" as oauthext;
```

The trailing slash resolves to the entry module.

The deck needs the `net` capability (the requests go through `http`), so it
runs on the default `jennifer` binary, not `jennifer-tiny`.

## Errors

A problem with the deck's own input is an `Error` of kind `"oauthext"` whose
message starts with `oauthext: `: a key file that is not a service-account key
or lacks a field, a private key that is not one PKCS#8 PEM key or cannot sign,
a token endpoint that is not https (plain http only on this machine, with no
`user@` part), a tenant that is not a name, an empty scope, a subject or
`client_email` that is not an e-mail address. A key file is checked completely before
any request is sent; one nested past `json`'s decoding caps is kind `"limit"`.
A refusal from the token endpoint keeps the `oauth` module's `oauth.OAuthError`
(kind `"oauth"`), with Google's error code and description in the message
(`invalid_grant: Invalid JWT Signature.`) and as the `errorCode` and
`description` fields. The deck passes everything `oauth` throws on unchanged:
an endpoint that cannot be reached (refused, reset, DNS, TLS) is `http`'s
`net.NetError` (kind `"net"`, with `op`, `reason`, `sent` and `replied`), a
reply past `json`'s decoding caps kind `"limit"`, and a request that stalls,
runs out of time or sends an oversized or malformed reply is whatever the
bundled `oauth` reports for it (an `oauth.OAuthError` when this was
written).

```jennifer
use io;

try {
    def tok as oauth.Token init oauthext.googleServiceAccount($keyJson, $scope);
} catch (e) {
    io.printf("%s: %s\n", $e.kind, $e.message);
}
```

## Moving from `@jennifer/oauthext`

Up to 0.3.0 this deck was `@jennifer/oauthext`. Change the `[decks]` entry to
`"@deckyard/oauthext" = "^0.4.0"` and the imports:

```jennifer
import "@deckyard/oauthext/" as oauthext;
```

## Trademarks

Google™, Google Cloud™ and Google Workspace™ are trademarks of Google LLC.
Microsoft™ and Microsoft Entra™ are trademarks of Microsoft Corporation. Other
product and company names may be trademarks of their respective owners. They
are used only to identify the products this deck works with. This deck is an
independent project, not affiliated with, sponsored or endorsed by Google LLC
or Microsoft Corporation.
