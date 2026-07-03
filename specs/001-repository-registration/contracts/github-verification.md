# Contract: GitHub Existence Verification (consumed)

**Feature**: 001-repository-registration | **Date**: 2026-07-03

This is an **outbound** contract — the app depends on GitHub's REST API to verify a repository
exists before create/edit (FR-017, FR-018). Encapsulated in
`app/services/github_verification_service.py`.

## Request

```
GET https://api.github.com/repos/{owner}/{repo}
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2022-11-28
Authorization: Bearer {GITHUB_TOKEN}   # optional; omitted when not configured
```

- `{owner}` and `{repo}` are the trimmed submitted values.
- Timeout: 10 seconds (connect + read).
- `GITHUB_TOKEN` is read from settings; when absent the request is unauthenticated
  (60 req/hr limit) — acceptable for a low-volume single-user tool.

## Response → outcome mapping

| GitHub response | Service result | User-facing effect |
|-----------------|----------------|--------------------|
| `200 OK` | `EXISTS` | Proceed to persist. |
| `404 Not Found` | `NOT_FOUND` | Reject: "Repository not found on GitHub." (FR-017) |
| `403` / `429` with `X-RateLimit-Remaining: 0` | `UNAVAILABLE` (rate-limited) | Reject: "Verification could not be completed, please retry." (FR-018) |
| `401` (bad/expired token) | `UNAVAILABLE` | Reject with retry message; log for operator. |
| Timeout / connection error | `UNAVAILABLE` | Reject: "Verification could not be completed, please retry." (FR-018) |
| Other `5xx` | `UNAVAILABLE` | Reject with retry message. |

## Service interface (conceptual)

```
class VerificationResult(Enum): EXISTS, NOT_FOUND, UNAVAILABLE

async def verify_repository_exists(owner: str, repo: str) -> VerificationResult
```

- The service returns a typed result; it does **not** raise for expected outcomes (NOT_FOUND,
  UNAVAILABLE). The calling service maps the result to persistence or a validation error.
- Never persists a repository when the result is not `EXISTS` (FR-017, FR-018; SC-008).

## Testing contract

- All tests stub GitHub (respx / httpx `MockTransport`) — no live network calls in the suite.
- Required test cases: `200`→EXISTS, `404`→NOT_FOUND, `403 rate-limit`→UNAVAILABLE,
  timeout→UNAVAILABLE, and (integration) that a NOT_FOUND/UNAVAILABLE result leaves the DB
  unchanged.
