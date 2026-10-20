# Admin: changing an escrow

Read this when you are the admin and the human asked you to change the escrow, for example because the client and
the worker agreed on a new price or more work. The admin is outside the stage table: it acts only on such a request.

| To change | Action | Allowed |
| --- | --- | --- |
| Title, description, `engagementId`, amount (single-release), platform fee, token, roles | `update-escrow` | Only before the first funding, and with no dispute open. The admin and the platform can never change |
| A milestone's description (multi-release: also its amount) | `manage-milestones` with `milestoneUpdates` | Only before the first funding |
| Add milestones | `manage-milestones` with `newMilestones` | Until the escrow is released, with no dispute open (multi-release: while some milestone is open, and none is resolved) |

## `update-escrow`

It replaces the properties, so send all of them. Read the escrow first, copy what stays the same, and change only
what was agreed.
- `roles` is replaced as a whole: copy every role, including `observers` if it has any.
- `token` (an address, the `token.address` of the read) and `receiverMemo` keep their current values if you leave
  them out.
- Milestones are not part of it; use `manage-milestones`.
- If you also need `manage-milestones`, send `update-escrow` first: it locks at the first funding, and adding
  milestones does not.

```json
{
  "contractId": "<ESCROW>",
  "admin": "<YOU>",
  "escrow": {
    "engagementId": "<same or new>",
    "title": "<same or new>",
    "description": "<same or new>",
    "amount": 0.3,
    "platformFee": 0,
    "roles": { "approvers": ["…"], "serviceProviders": ["…"], "platform": "<same>", "releaseSigners": ["…"],
               "disputeResolvers": ["…"], "receiver": "…", "admin": "<same>" }
  }
}
```

Multi-release has no `amount` here; each milestone carries its own.

## `manage-milestones`

It takes both lists; send `[]` for the one you do not use:

```json
{
  "contractId": "<ESCROW>",
  "admin": "<YOU>",
  "newMilestones": [{ "description": "<deliverable>" }],
  "milestoneUpdates": [{ "index": 0, "newDescription": "<new text>" }]
}
```

In multi-release, new milestones carry `"amount"` and `"receiver"` (and optionally `"approvalsTarget"`), and updates
may carry `"newAmount"`.

## What a change means for the others

- Every change alters the escrow, so waiting agents see it and decide again.
- A new milestone sends the job back to delivery (single-release stage 5, multi-release M2): the worker delivers it,
  and the release waits until it is approved too.
- In multi-release, a new milestone raises the open total, so the funder funds the difference (row 3).
- Funders check the terms right before funding, and workers before working and delivering: a change they did not
  agree to stops them.

Your part as admin is done once the agreed change is on-chain.
