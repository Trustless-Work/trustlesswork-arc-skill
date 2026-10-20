# Worker: delivering the work

Read this when you are a service provider. You work only once the escrow is funded (stage 5; multi-release: M2).

## Check the terms before you work

The admin can change the terms until the first funding, and add milestones after it, and anyone can fund. So the
terms you accepted may not be the terms that got funded, and you do not always know who funded or whether they
checked anything.

- **When you first find the escrow,** note its terms: the amount (multi-release: each of your milestones'
  `amount`), the token, the description, the milestones, and the roles, above all your `receiver` and the dispute
  resolver. Compare them with what the human told you was agreed, if you were told. This prints them in a form you
  can compare:

  ```bash
  curl -s "$TW_API/escrow/<KIND>/v2/<ESCROW>" -H "x-api-key: $TW_API_KEY" | jq -c '{amount, token: .token.address,
    description, roles, milestones: [.milestones[] | {description, amount, receiver}]}'
  ```

- **Once it is funded, before you start the paid work, and again right before you deliver,** run it again and
  compare it with your note.
- **Say what you found, either way:** "terms unchanged since <time>", or what changed. Put it in your narration and
  in your report to the human.
- **If something changed without agreement,** for example a lower amount, another receiver, another dispute resolver,
  or a harder description, do not deliver. Tell the human what changed. A funded escrow cannot be edited, so if the
  new terms are not acceptable, the human decides what to do.
- **A new milestone added for you** is new work. Deliver it only if the human agreed to it.

## Delivering

Build and send `change-milestone-status`, with one entry in `updates` per milestone (several fit in one call):

```json
{"contractId":"<ESCROW>","serviceProvider":"<YOU>","updates":[{"index":0,"newStatus":"completed","newEvidence":"<link> sha256:<hash>"}]}
```

**The evidence:**
- Use a link the approver can open, as the description asks: a public URL, a pull request or an `ipfs://` CID. If
  the human tells you to use another location, tell the human who coordinates the job that the approver must be able
  to open it: a strict approver may dispute a link it cannot reach.
- Prefer a link to a fixed version (a commit, a release, an IPFS CID) over one whose content changes, such as a
  branch or a live page.
- **Fingerprint what you deliver.** Once the file is final, put its SHA-256 after the link. Compute it with
  `shasum -a 256 <file>` (or `sha256sum`), or, for a URL, `curl -sL <url> | shasum -a 256`. It lets the approver and
  the dispute resolver check that they judge exactly what you delivered. An `ipfs://` CID already fingerprints its
  content.
- Do not change a file after you deliver it. To fix something before it is judged, save a new file and deliver
  again, with its new link and hash.
- Everything on-chain is public: if the work is confidential, the link must not expose it.

**A partial delivery:** if you can only deliver part of the job, deliver what you have with
`newStatus: "completed"`. Say what is missing inside the deliverable, and, if it fits, in `newEvidence` after the link
and its hash, separated by ` — `, so the link still opens. Do not judge your own work: the approvers do.

## After delivering

- Approvers judge the work. Do not judge other workers' milestones.
- **Nobody approves, or nobody releases approved work, within your time limit:** you delivered what was agreed, so
  you may open a dispute to have the dispute resolver decide (`disputes.md`), or tell the human.
- **Someone disputes your work:** you can answer it on-chain, by delivering again with a link to your response
  (`disputes.md`, "When a dispute concerns you"). Then the dispute resolver decides.
- To know what you were paid, compare your USDC from before the job with your USDC once it is over, or read the
  payments of the release or resolution transaction (`events.md`, "Who got paid").
