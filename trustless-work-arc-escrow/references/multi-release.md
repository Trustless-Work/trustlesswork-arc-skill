# Multi-release escrows

Read this when the escrow is multi-release. Use these stages instead of the single-release ones in `SKILL.md`.

Each milestone moves on its own, so different milestones can be at different stages at the same time. A milestone
is **open** until it is released or its dispute is resolved.

## Stages

First, the escrow as a whole:

| # | The escrow shows | Acts now | Everyone else |
| --- | --- | --- | --- |
| 1 | No escrow yet | The client, or the admin on its behalf: `deploy` | Wait for the new escrow |
| 2 | Every milestone is released or resolved | Nobody, except: if `balance` > 0, the dispute resolver returns the leftover (`disputes.md`) | Stop: the job is over |
| 3 | `balance` below the total `amount` of the open milestones | The funder, usually the client: approve USDC, then `fund` the missing amount | Wait (next: the funding) |

Then each open milestone, the first row that matches:

| # | The milestone shows | Acts now | Everyone else |
| --- | --- | --- | --- |
| M1 | `dispute.isDisputed` is `true` | The dispute resolver: evaluate it, then `resolve-dispute` with its index | Wait |
| M2 | `status` is not `completed` | Its service provider (with several, the one that is its `receiver`): check the terms, do the work, then deliver it | Wait |
| M3 | Delivered, but `approvalCount` below `target` | Each approver not yet in its `approvedBy`: evaluate, then approve it or dispute it | Wait |
| M4 | Approved, not released | A release signer: `release-funds` with its index | Wait |

To check whether the escrow is funded (row 3):

```bash
curl -s "$TW_API/escrow/multi-release/v2/<ESCROW>" -H "x-api-key: $TW_API_KEY" \
  | jq '.balance >= ([.milestones[] | select((.released or .dispute.resolved) | not) | .amount] | add // 0)'
```

## Rules

- **Do everything that is your turn, then wait.** Several milestones can need you at once. If they are at the same
  row, act on them in one call, for example `"milestoneIndexes": [0, 1]`. If they need different actions (approve
  one, dispute another), send one action after the other, each with its own build.
- **Several service providers:** each one delivers the milestones whose `receiver` is their wallet, unless the
  description assigns them otherwise.
- **Disputes are per milestone.** Approvers, service providers, release signers and the platform can dispute any open
  milestone; a receiver only its own; the dispute resolver never. A dispute stops only that milestone; the others
  keep going.
- **Resolving:** name the disputed milestones in `milestoneIndexes`. The distributions add up to exactly those
  milestones' `amount`, not to the whole balance.
- **Releasing** pays each milestone's own `receiver`, minus the fees. The escrow must hold enough USDC for the
  milestones you release.
- **Funding in stages:** if the description says the client funds milestone by milestone, a milestone's work starts
  when the balance covers it.

## How the bodies differ from single-release

- `deploy` has no `amount` and no `roles.receiver`. Each milestone carries its own:
  `"milestones":[{"description":"<deliverable>","amount":0.5,"receiver":"<WORKER>"}]`.
- `fund` takes the total of the open milestones' `amount`s: `{"contractId":"<ESCROW>","signer":"<YOU>","amount":0.4}`.
- `release-funds` takes the milestones: `{"contractId":"<ESCROW>","releaseSigner":"<YOU>","milestoneIndexes":[0]}`.
- The dispute action is `dispute-milestones`, and `resolve-dispute` also takes `milestoneIndexes` (`disputes.md`).
- `approve-milestones`, `approve-and-release-milestones` and `change-milestone-status` take the same bodies as in
  single-release, with each milestone's index.
- `withdraw-remaining-funds` takes the single-release body once every milestone is released or resolved.
