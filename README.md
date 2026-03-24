# wai-session-clientsession-deferred

A fork of [wai-session-clientsession](https://github.com/singpolyma/wai-session-clientsession) with deferred session decryption. Uses [wai-session-maybe](https://github.com/digitallyinduced/wai-session-maybe) instead of `wai-session`.

## Why this exists

The upstream `wai-session-clientsession` package has not been updated since 2012. It eagerly decrypts the session cookie on every request and re-encrypts it on every response, even when the session is never accessed. We submitted [PR #5](https://github.com/singpolyma/wai-session-clientsession/pull/5) with a deferred implementation but the repository appears dormant.

Since the change depends on the modified `SessionStore` type from `wai-session-maybe`, we publish this as a separate package.

## How it works

The patched `clientsessionStore` uses an `IORef` with three states:

| Session state | Decrypt | Encrypt | Set-Cookie |
|---|---|---|---|
| Never accessed (most routes) | No | No | Skipped |
| Read-only (e.g. auth check) | Yes | **No** | Yes (original bytes) |
| Written (login/logout/flash) | Yes | Yes | Yes (new ciphertext) |

- **Deferred decryption**: The cookie is not decrypted until the session is first read or written. Routes that never access the session pay zero AES cost.
- **Dirty tracking**: An `IORef` tracks whether the session was modified. Read-only access returns the original encrypted cookie bytes without re-encrypting.
- **Cookie expiry refresh**: Read-only sessions echo back the original ciphertext, which refreshes the browser's `max-age` timer (so the cookie expires 30 days from last visit, not 30 days from login) with zero crypto overhead.

## Performance

Benchmarked with a minimal WAI app using `withSession` + `clientsessionStore`, comparing the original eager implementation to the deferred one:

| Scenario | Before | After | Improvement |
|---|---|---|---|
| No cookie (new visitor) | 43,858 req/s, p50=2.2ms | 128,990 req/s, p50=0.4ms | **2.9x throughput, 5.5x lower latency** |
| With cookie (read-only) | 39,533 req/s, p50=2.5ms | 123,160 req/s, p50=0.4ms | **3.1x throughput, 6.3x lower latency** |

In a full [IHP](https://ihp.digitallyinduced.com/) application, auth overhead (session decrypt + DB user lookup) is approximately 0.04-0.08ms per request — negligible compared to the ~2ms per request the eager implementation added.

## Module name

| wai-session-clientsession | wai-session-clientsession-deferred |
|---|---|
| `Network.Wai.Session.ClientSession` | `Network.Wai.Session.ClientSession.Deferred` |

## Usage

```haskell
import Network.Wai.Session.ClientSession.Deferred (clientsessionStore)
```

The `clientsessionStore` function has the same signature as the original, except it returns the `SessionStore` type from `wai-session-maybe` (with `Maybe ByteString`).

## License

ISC (same as the original wai-session-clientsession).
