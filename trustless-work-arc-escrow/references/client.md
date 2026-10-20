# Client: creating and funding an escrow

Read this when you create or fund the escrow. For a multi-release escrow, also read `multi-release.md`. If you are
also an approver or a release signer, read `approver.md` too.

## Choosing the setup

Use **single-release** for one payment once every milestone is approved, and **multi-release** for staged payments
or when different workers deliver different parts.

The roles are the human's decision: use the wallets the human gives you for each one. The contract only rejects
overlaps: a dispute resolver cannot hold any other role, and the admin cannot be approver, service provider, release
signer, dispute resolver or receiver (it can be the platform). If the human's wallets break these rules, say which
role is missing a separate wallet instead of guessing. Two setups that work:

| Wallet | Four wallets | With an independent evaluator who approves |
| --- | --- | --- |
| Client | Funds; approver and release signer | Funds; release signer |
| Worker | Service provider and receiver | Service provider and receiver |
| Evaluator | Dispute resolver | Approver |
| Arbiter | (not needed) | Dispute resolver |
| Admin | Admin and platform | Admin and platform |

Each role list takes 1 to 5 addresses. One Circle login can hold up to 5 Agent Wallets per chain.

## Creating it

Build `deploy` and send it (see "How every action is sent" in `SKILL.md`). Any wallet can deploy; the `signer` needs
no role. Single-release body:

```json
{
  "signer": "<YOU>",
  "engagementId": "<unique id, max 100 chars>",
  "title": "<short title>",
  "description": "<what must be delivered, item by item, and how a dispute will be settled>",
  "roles": {
    "approvers": ["<CLIENT>"],
    "serviceProviders": ["<WORKER>"],
    "platform": "<ADMIN>",
    "releaseSigners": ["<CLIENT>"],
    "disputeResolvers": ["<EVALUATOR>"],
    "receiver": "<WORKER>",
    "admin": "<ADMIN>"
  },
  "amount": 1,
  "platformFee": 0,
  "milestones": [{ "description": "<deliverable>" }]
}
```

- The roles above follow the four-wallet setup. With an independent evaluator, use
  `"approvers": ["<EVALUATOR>"]`, `"releaseSigners": ["<CLIENT>"]` and `"disputeResolvers": ["<ARBITER>"]`.
- The build returns `contractId`: the address the escrow will have once you send it. Every build gets its own
  `contractId`, so keep the one from the build you actually send.
- `engagementId` is your own reference for the job: any string you have not used before.
- Length limits: `engagementId` and `title` up to 100 characters; `description`, each milestone `description`, a
  dispute `reason` and `newEvidence` up to 500; `newStatus` up to 50. Put long details in the milestones or in a
  linked file.
- `platformFee` is a percentage (`5` = 5%).
- Each milestone needs 1 approval by default. To require more (N of M approvers), add `"approvalsTarget": 2` to the
  milestone. It cannot exceed the number of approvers.

**Write the description well.** It is the agreement: the approver judges the work against it, and the dispute
resolver settles any disagreement with it.
- **Write only what the human agreed.** Never invent a term: a deadline or a dispute method decides who gets paid, so
  it must come from the parties, not from you.
- List the things to deliver one by one, so everyone counts them the same way.
- If the human gave you terms for when something goes wrong, write them in: deadlines (the chain records when each
  step happened), how long the approver has to answer, a method for disputes (a score, per item, all or nothing, a
  fixed split…; see `disputes.md`) and whose judgment decides.
- If the human gave none, leave them out: the dispute resolver then judges what the agreement entitles each side to.
  If you think one is needed, ask the human before you deploy instead of choosing it yourself.
- Everything on-chain is public and permanent: the title, the description, the evidence and any dispute reason. Keep
  confidential details in a linked file.

## Funding it

1. **Check the terms right before you fund.** The admin can change them until the first funding. Compare the amount,
   the token, the roles, the description and the milestones with what was agreed. If they changed without
   agreement, do not fund; tell the human.
2. **Approve the USDC** directly with the CLI, for the exact amount in base units (amount × 1,000,000).
3. **Build and send `fund`:** `{"contractId":"<ESCROW>","signer":"<YOU>","amount":1}`. If the approval is missing,
   the build answers `TOKEN_ALLOWANCE_TOO_LOW`.
4. **Read the escrow:** `balance` should now equal `amount`.

Then follow the loop in `SKILL.md`: the worker delivers next.
