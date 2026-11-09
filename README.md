# nono + Aauth public demo

This demo uses nono's self-hosted Aauth agent-token flow to call https://whoami.aauth.dev/.

```text
Sandboxed application → local nono proxy → signed HTTPS request → whoami API
                                                                     ↓
                                              issuer discovery → public JWKS
```

nono issues a one-hour `aa-agent+jwt` token at startup and signs each HTTP request with Ed25519.
The request uses the `jwt` scheme. The metadata's `jwks_uri` locates the public key used to verify the token.
The API verifies the token, then the HTTP request using its bound confirmation key.

## Public resources

| Resource | URL |
|---|---|
| Issuer | https://sequei.github.io |
| Discovery | https://sequei.github.io/.well-known/aauth-agent.json |
| JWKS | https://sequei.github.io/aauth-demo/jwks.json |
| Agent identifier | `aauth:nono-demo@sequei.github.io` |
| Test API | https://whoami.aauth.dev/ |

GitHub Pages project sites live under a repository path. Aauth discovery must be at the issuer origin's root, so the tiny [sequei.github.io](https://github.com/SequeI/sequei.github.io) repository hosts discovery and this repository hosts the demo and JWKS.

## Verified against the public API

Tested at **2026-11-09T11:44:01Z** with a real nono binary and the publicly hosted HTTPS metadata/JWKS.
No local receiver, TLS bypass, or verifier parser adapter was used.

| Check | Result |
|---|---|
| Public discovery and JWKS over trusted HTTPS | HTTP 200 |
| Unsigned request to `whoami.aauth.dev` | HTTP 401 |
| JWT-bound signed `GET /` through nono | HTTP 200 |
| `GET /not-permitted` through nono | HTTP 403, denied locally |
| `POST /` through nono | HTTP 403, denied locally |
| Reading the private key from the sandbox | Blocked |

The public API returned:

```json
{
  "iss": "https://sequei.github.io",
  "sub": "aauth:nono-demo@sequei.github.io"
}
```

See [verification evidence](evidence/verification.json) and the [sanitized audit events](evidence/audit.json).
These record one actual run; API availability and authorization remain controlled by the receiving service.

## Run with the enrolled demo key

Use a nono binary built from the branch implementing `scheme.type: jwt`.
The profile template is [profiles/whoami.example.json](profiles/whoami.example.json).
It permits only `GET /`; other paths and methods are denied locally.

Copy it to `.local/whoami.json`, replacing `key_ref` with the full absolute path to the matching private key. JSON does not expand `$HOME` or `~`.

```sh
mkdir -p .local
cp profiles/whoami.example.json .local/whoami.json
# Edit .local/whoami.json: set key_ref to your actual absolute key path.
nono run --profile .local/whoami.json --no-rollback -- \
  sh -c 'curl --noproxy "" --max-time 30 --fail-with-body "$AAUTHAPI_BASE_URL/"'
```

The private key used by this deployment is deliberately absent from the repository.
A new random key will not match this hosted JWKS. To reproduce with your own identity, use a domain/Pages issuer you control, generate a dedicated key, publish its JWKS, and update issuer and agent_id together.

## Set up your own issuer

```sh
nono aauth keygen --keyref "$HOME/.config/nono/aauth-keys/aauth-demo.pem"
nono aauth show --keyref "$HOME/.config/nono/aauth-keys/aauth-demo.pem" --jwks
```

Publish the second command's public JSON output at your JWKS URL. At your issuer's `/.well-known/aauth-agent.json`, publish:

```json
{
  "issuer": "https://yourname.github.io",
  "jwks_uri": "https://yourname.github.io/aauth-demo/jwks.json"
}
```

Configure `scheme.type` as `jwt`, `scheme.issuer` as that bare HTTPS origin, and `agent_id` as `aauth:nono-demo@yourname.github.io`.
The same persistent Ed25519 key signs the self-issued token and HTTP requests.

## Audit and limits

```sh
nono audit list --json
nono audit show <session-id> --json
```

Signed events include scheme `jwt`, agent identifier, issuer, and key thumbprint. Policy denials occur before signing and have no upstream status. Private key files must remain outside sandbox filesystem grants.

Tokens last one hour in this MVP; restart the session to renew. External agent-provider enrollment, automatic renewal, and person/access-server flows are not implemented. The receiving service controls identity acceptance and authorization.

Only public metadata, public keys, example configuration, and sanitized test results belong in this repository. `.local/` is ignored; private keys must stay outside the checkout.
