<!-- SPDX-License-Identifier: LGPL-3.0-only -->
<!-- SPDX-FileCopyrightText: Copyright (C) 2026 mplx <mplx@jennifer-lang.dev> -->

# Design

## Scope

The deck holds the parts of OAuth2 that belong to one provider, on top of
the bundled `oauth` module:

- **Presets**: `google` and `microsoft` return an `oauth.Config` with the
  provider's token, device-authorization and authorization endpoints, for
  the device flow, the browser sign-in (`oauth.authorize`), refresh and
  client credentials.
- **Google service accounts**: `serviceAccount` reads and checks a JSON key
  file, `serviceAccountAssertion` signs the RFC 7523 assertion with Google's
  conventions, and `googleServiceAccount` / `googleServiceAccountAs` trade it
  for a token through `oauth.jwtBearer`.

It deliberately is not:

- **A second OAuth implementation.** The grants, token handling, storage,
  discovery and reply checks stay in `oauth`; the deck only fills in
  provider facts and calls it.
- **An API client.** What the token is used for (Cloud Storage, Graph, IMAP)
  is another deck's or the caller's business.
- **A token cache.** A service-account token lasts about an hour and has no
  refresh token; the caller asks for a new one when `oauth.isExpired` says
  so.
- **Other providers yet.** Okta, Salesforce, Oracle and the like are a deck
  change when someone needs them, never a core one. Providers that publish
  standard metadata are already served by `oauth.discover`.

## Decisions

- **Provider knowledge lives outside core.** Core `oauth` keeps the standard
  mechanics; endpoints, key formats and claim conventions of one vendor
  change on the vendor's schedule, so they belong in a deck. The presets
  were `oauth.google` and `oauth.microsoft` in the bundled module up to
  Jennifer 0.25 and moved here with the same parameters (a breaking change in
  0.26.0; [Moving from oauth](migrating.md) shows it).
- **Generic grant in core, conventions here.** The JWT-bearer grant
  (`oauth.jwtBearer`) and the standard claim set (`oauth.jwtAssertion`, over
  `jwt.signWithKid`) are provider-neutral and live in `oauth`. The deck adds
  only Google's choices: RS256, `kid` the key's `private_key_id`, `iss` the
  account, `aud` the token URI, `scope` as an extra claim, `sub` the user for
  domain-wide delegation, and a lifetime of `SERVICE_ACCOUNT_LIFETIME`
  (3600 s, Google's maximum).
- **Microsoft tenants by name.** `microsoft` builds the v2.0 endpoint URLs
  from the tenant instead of reading metadata, because the multi-tenant
  `common`, `organizations` and `consumers` metadata names a placeholder
  issuer that `oauth.discover` cannot use.
- **The key file is checked completely before any request.** It must be a
  service-account key (`"type": "service_account"`) with an e-mail
  `client_email`, exactly one PKCS#8 PEM `private_key`, a `private_key_id`
  without spaces or control characters, and a usable `token_uri`; a
  `project_id`, when present, must be a string. A user's credentials file or
  an OAuth client file is refused rather than guessed at.
- **Constants for Google's endpoints.** `GOOGLE_TOKEN_URL`,
  `GOOGLE_AUTHORIZE_URL` and `GOOGLE_DEVICE_URL` are exported so a caller
  building its own `Config` uses the same values.

## Errors

- **The deck's own input** is an `Error` of kind `"oauthext"` with a message
  starting `oauthext: `: a key file that is not a service-account key or
  lacks a field, a private key that is not one PKCS#8 PEM key or cannot sign,
  a `token_uri` that is not https (or http on this machine), a tenant that is
  not a name, an empty or control-character scope, a subject or
  `client_email` that is not an e-mail address. These are plain `Error`
  values: they carry no server status, so no struct is needed. A key file
  nested past `json`'s decoding caps is kind `"limit"` instead.
- **The token endpoint's answer** is `oauth`'s error, passed on unchanged:
  `oauth.OAuthError` (kind `"oauth"`) with Google's or Microsoft's
  `errorCode` (`invalid_grant`, ...), `description` and `status`, also for a
  reply `oauth` cannot trust (not JSON, no token, a stall, the 60 s limit).
- **Generic kinds** come from `oauth` as it throws them. On interpreters with
  transport error kinds (M28.65) a socket failure (refused, reset, DNS, TLS)
  is `http`'s `net.NetError` (kind `"net"`) and a reply past `json`'s caps
  `"limit"`; older ones report those as `oauth.OAuthError` too. A caller that
  must tell "never reached Google" from "Google said no" matches the kind.

## Security and limits

- **Keys stay with the caller.** The deck reads a key file's content handed to
  it and never writes it anywhere. The tests make fresh keys with
  `crypto.keypair("rsa2048")`, so no private key is checked in.
- **Token endpoints are https.** A `token_uri` must be https, or plain http
  only to `127.0.0.1`, `localhost` or `[::1]` (an emulator). It may hold no
  `user@` part, fragment, space or control character, since a login part
  could make a local-looking URL go elsewhere. `oauth` applies the same rule
  again before sending.
- **Text from input is shown safely.** Values quoted in an error message are
  cut at 80 characters with control characters replaced, so a hostile key
  file cannot forge log lines.
- **Tenant names are restricted** to names Microsoft accepts in the URL path
  (at most 253 characters of dot-separated letter, digit and hyphen labels),
  so a tenant cannot add a path, query or host to the endpoint URLs.
- Reply size (1 MiB), read and overall deadlines are `oauth`'s.

## Testing and validation

`src/oauthext_test.j` covers both presets, the tenant check, every way a key
file is refused (type, fields, types of fields, PEM shape, key id,
`token_uri` scheme, host, login part), the assertion's header and claims
(verified with the public key through `jwt.verify`), refused scopes and
subjects, a key that cannot sign, and the full trade against a loopback
`httpd` token endpoint: the posted form (`grant_type` and `assertion` only),
a token reply, a refusal that keeps kind `"oauth"` with Google's message,
and a non-JSON `502`. A bad key file is shown to fail before any request.

The tests never reach Google or Microsoft. The examples
(`examples/service_account.j`, `examples/device_flow.j`) run against the real
services when their environment variables are set and print how to run them
otherwise, so CI runs them offline.

## Known limitations and open questions

- Only RS256 PKCS#8 keys, the form Google issues; a P12 key file is not read.

## Dependencies on the interpreter

- The floor is `>=0.27.0`, for the UTF-8 codecs of `encoding.encode` /
  `decode` and `strings.replicate`. `oauth.jwtBearer`, `oauth.jwtAssertion`
  and `jwt.signWithKid` are not in 0.25.0, and `oauth.Config.authorizeUrl`
  and `oauth.Token`'s `idToken` / `extra` came with 0.26.0.
- `oauth.OAuthError` and its `errorCode` field (M28.51); the `tls` field of
  `oauth.Config` (M28.47) may be left out of a literal, so the presets work
  with and without it.
- RS256 signing and `http` need the default `jennifer` binary (the `net`
  capability); `jennifer-tiny` cannot run the deck.
