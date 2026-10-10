# Security and reliability review

Status: offline source review and regression testing only. No Honda/Acura account,
vehicle connection, or Home Assistant deployment has been validated.

## Compatibility gate

This is a HondaLink MY21 custom integration. It uses HondaLink client identity,
business ID, system ID, and User-Agent values. AcuraLink / 2026 MDX support is
unverified. Before deployment for Acura, obtain current AcuraLink app/API evidence,
validate authentication and account-owned vehicle discovery, and validate cached
telemetry. Physical vehicle commands require a separate deliberate test.

## Findings addressed

- **High — sensitive data in errors:** invalid JSON, command failures, and polling
  timeouts exposed remote response bodies. Error messages could contain account
  data, VIN, location, PIN, or tokens even when selected keys were redacted.
  Errors now omit remote text and preserve only explicitly known error codes.
  VIN was also removed from a setup warning and the coordinator log name.
- **High — credentials across redirects:** HTTP requests followed redirects,
  potentially forwarding form data or custom authentication headers. Redirects
  are now refused. TLS certificate verification remains enabled.
- **Medium — broken authentication recovery:** HTTP 401 bypassed the existing
  refresh code; stale registration could persist indefinitely. HTTP/body 401 now
  allows one registration refresh and one retry. HTTP 403 and ambiguous transport
  failures are not replayed. Concurrent expired-token logins are serialized.
- **Medium — false locked indication:** one reported locked door was enough for
  an all-doors-locked result. All four known states are now required; missing or
  unrecognized states produce unknown unless a door is explicitly unlocked.
- **Medium — unavailable settings:** current Home Assistant owns OptionsFlow's
  config_entry property. Removed the assignment and added an update listener so
  settings apply without restarting. Poll intervals are bounded to 1–1440 minutes;
  lock command settings are restricted to the already implemented variants.
- **Medium — transport failures:** added a 30-second per-request timeout,
  sanitized transport errors, and rejected non-object JSON. Setup and polling
  distinguish authentication failure from temporary connectivity failure.
- **Privacy:** password/PIN inputs use password selectors. Removed exploratory
  dashboard attributes from the battery entity. Refreshed auth data is persisted,
  and reauthentication checks that the configured vehicle belongs to the account.
- **Resource handling:** close the integration HTTP session on initial auth/poll
  failure and successful unload.

## Upstream work reviewed

- [Options-flow PR #3](https://github.com/daviddelahoz/ha-hondalink/pull/3), by berkman:
  adapted the options-flow and live-options fixes.
- [Authentication PR #5](https://github.com/daviddelahoz/ha-hondalink/pull/5), by bcrook:
  adapted gateway headers, app version, timestamp format, auth persistence, and
  registration recovery. Reworked retries to cap mixed HTTP/body 401 responses.
  Its author reports testing on a 2026 Honda Pilot; that does not establish Acura
  compatibility.
- Open issues include no-PIN setup, pre-MY21 dashboard errors, and gateway 401s.
  This change does not claim to solve unsupported vehicle platforms or no-PIN
  account setup.

## Remaining limits

Credentials and tokens are stored in Home Assistant config entries; protect the
configuration directory and backups. This review is not a penetration test or a
certification. No live authentication, telemetry freshness, remote command behavior,
or physical lock state has been verified. The default upstream lock behavior is a
**two-minute delayed lock**, and cached telemetry can lag the vehicle's actual state.
Do not use cached lock state as proof that the vehicle is secured.

The upstream application credentials are public mobile-app identifiers, not personal
credentials. Honda can change or revoke them. Cloud API availability and subscriptions
remain external dependencies. No package requirements are added to the integration;
aiohttp and other libraries are supplied by Home Assistant.

## Validation

Run offline regression tests:

```sh
python -m unittest discover -s tests -v
```

Tests cover response privacy, redirect refusal, timeouts, malformed response handling,
HTTP/body 401 retry bounds, no 403/network replay, concurrent login, command errors,
and partial door-lock telemetry. Home Assistant UI/lifecycle and live vehicle tests
remain required before deployment. Upstream HACS and hassfest workflows are retained.
