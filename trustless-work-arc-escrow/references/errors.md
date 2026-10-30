# Errors

API errors come back as JSON with a `code`, a `detail` and an HTTP status. Fix the cause and **build again**; do not
resend an old build.

| Code | Meaning | What to do |
| --- | --- | --- |
| `TOKEN_ALLOWANCE_TOO_LOW` | The escrow may not pull that much USDC yet | Send the USDC approval directly with the CLI (see "How every action is sent" in `SKILL.md`), then build `fund` again. The error's `detail` mentions an `approve-token` endpoint: that one is for other wallets; ignore it |
| `BAD_REQUEST`, `VALIDATION_ERROR` | The body is invalid: a missing field, a wrong type, a text too long | Read `detail` and `extensions.errors`, fix the body, build again |
| `ESCROW_INSUFFICIENT_FUNDS_FOR_FUNDING`, `TOKEN_INSUFFICIENT_BALANCE` | Your wallet lacks the USDC | Tell the human. On testnet: `circle wallet fund --address <YOU> --chain ARC-TESTNET`. On mainnet only the human adds USDC |
| `ESCROW_UNAUTHORIZED_APPROVER`, `ESCROW_ONLY_SERVICE_PROVIDER_CAN_CHANGE_MILESTONE_STATUS`, `ESCROW_ONLY_RELEASE_SIGNER_CAN_RELEASE`, `ESCROW_ONLY_DISPUTE_RESOLVER_CAN_EXECUTE`, `ESCROW_SIGNER_MUST_BE_APPROVER_AND_RELEASE_SIGNER` | Your wallet does not hold the role for this action | Check `roles`; another wallet must act |
| `ESCROW_NOT_COMPLETED` | Not every milestone is approved | Wait for, or ask for, the approvals |
| `ESCROW_MILESTONE_HAS_ALREADY_BEEN_APPROVED`, `ESCROW_APPROVER_ALREADY_APPROVED_MILESTONE` | Already done | Read the escrow and move to the next step |
| `ESCROW_ALREADY_RELEASED`, `ESCROW_ALREADY_RESOLVED` | The escrow is closed | Nothing to do |
| `ESCROW_ALREADY_IN_DISPUTE`, `ESCROW_OPENED_FOR_DISPUTE_RESOLUTION` | A dispute is open | Only the dispute resolver can act now |
| `ESCROW_DISTRIBUTIONS_MUST_EQUAL_BALANCE` | The split does not add up | Single-release: make it add up to `balance`. Multi-release: to the disputed milestones' `amount` |
| `ESCROW_DISPUTE_RESOLVER_OVERLAPS_WITH_OTHER_ROLE`, `ESCROW_ADMIN_ADDRESS_OVERLAPS_WITH_OTHER_ROLE` | The roles break the overlap rules | Tell the human which role needs a separate wallet (`client.md`) |
| `ESCROW_PROPERTIES_MISMATCH` | The escrow changed since the build; or, on `update-escrow`, it is already funded and its properties are locked | Read the escrow again and rebuild; a funded escrow cannot be edited |
| `ESCROW_ONLY_ADMIN_CAN_EXECUTE` | Only the admin may change the escrow | Ask the admin's agent or the human |
| `ESCROW_ADMIN_ADDRESS_CANNOT_BE_CHANGED`, `ESCROW_PLATFORM_ADDRESS_CANNOT_BE_CHANGED` | `update-escrow` tried to change the admin or the platform | Keep them as they are |
| `ESCROW_MILESTONE_UPDATE_NOT_ALLOWED_WITH_FUNDS` | Milestones can be edited only before the first funding | You can still add new milestones |
| `ESCROW_NOT_FOUND` | No escrow of that kind at that address | Check the address and the kind |
| `ESCROW_MILESTONE_ALREADY_RELEASED`, `ESCROW_MILESTONE_ALREADY_RESOLVED` | Multi-release: that milestone is closed | Move on to the open milestones |
| `ESCROW_MILESTONE_ALREADY_DISPUTED` | Multi-release: that milestone is already in dispute | Only the dispute resolver acts on it |
| `ESCROW_BALANCE_NOT_ENOUGH_TO_SEND_EARNINGS`, `ESCROW_INSUFFICIENT_FUNDS_FOR_RESOLUTION` | The escrow lacks USDC for this payout | The funder must fund the missing amount |
| `ESCROW_UNAUTHORIZED_TO_CHANGE_DISPUTE_FLAG` | Multi-release: you may not dispute that milestone | A receiver can only dispute its own milestones |
| `ESCROW_NOT_IN_DISPUTE`, `ESCROW_NOT_FULLY_PROCESSED` | Nothing to resolve, or milestones are still open | Read the escrow and find the stage again |
| `RATE_LIMITED` (HTTP 429) | Too many requests for this key or address | Wait the seconds in the `Retry-After` header, then retry. Keep waits at one check every 10 seconds |
| `API_KEY_ROLE_INSUFFICIENT` | The key cannot use this endpoint | Ask the human for a key with the `ESCROW_MANAGER` role |
| `ARC_NETWORK_NOT_CONFIGURED` (HTTP 503) | The API does not serve that network yet | Tell the human. Do not switch to the other network |
| `ARC_NETWORK_PREFIX_REQUIRED` (HTTP 404) | `TW_API` does not end with the network | Add `/testnet` to `TW_API` (`/mainnet` only if the human chose mainnet) and retry; `extensions` lists both paths |
| `API_KEY_INVALID` | The key is missing, wrong, revoked or expired | Send `-H "x-api-key: $TW_API_KEY"` with `TW_API_KEY` set in the same command (see "Before you start" in `SKILL.md`). If it still fails, ask the human for the key |

Circle CLI errors:

| Error | Meaning | What to do |
| --- | --- | --- |
| `Fails to perform transaction estimation` or `ESTIMATION_ERROR` | The call would fail on-chain | Build it again with the API to get the exact `code`. Do not retry blindly |
| Not logged in, or the token expired | The CLI session is gone | Ask the human to log in again (Circle's `setup.md`) |
| Terms not accepted | First use of the CLI | Ask the human to accept them. Never accept them yourself |
