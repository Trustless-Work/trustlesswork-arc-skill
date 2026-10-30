# Disputes and leftover funds

Read this when you open a dispute, when a dispute concerns you, or when you are the dispute resolver.

A dispute means the parties disagree and someone neutral has to decide. It is not one fixed scenario: a client may
dispute a missing delivery, a worker may dispute an approver who never answers, a release signer may dispute a
problem the approvers missed, a platform may dispute a broken rule. The dispute resolver then acts as the judge: it
decides who is right under the agreement and moves the funds accordingly.

## The rules the contract enforces

- **Who can open one:** approvers, service providers, release signers, the receiver and the platform. Never the
  dispute resolver. The admin only if it is also the platform. In multi-release, a receiver can dispute only its own
  milestones.
- **When:** at any stage before the escrow is released or resolved (multi-release: before that milestone is), even
  before the delivery. Only one dispute can be open at a time (multi-release: per milestone).
- **What it blocks:** while it is open, nothing can be released. The service provider can still update the status
  and evidence, so the worker can answer the dispute on-chain.
- **What it needs to end:** the dispute resolver's `resolve-dispute`. It needs funds: the distributions add up to
  exactly the balance (multi-release: the disputed milestones' `amount`, and the escrow must hold it), and the total
  cannot be zero. A dispute on an unfunded escrow cannot be resolved until someone funds it.
- **The outcome:** any split between any addresses, from everything to one party to any proportion, minus the fees.

## Opening a dispute

Dispute when the agreement is broken and your role cannot fix it alone, for example:

| You are | Typical reasons |
| --- | --- |
| Approver or client | The work is missing, wrong, incomplete, or late against the description |
| Service provider or receiver | You delivered, and the approvers do not answer within your time limit, or reject work that meets the description; the work is approved but nobody releases it |
| Release signer | The approvals are in, but you see a clear problem the approvers missed, such as no evidence at all |
| Platform | A rule of the platform or the description was broken |

Before you dispute:
- **Check that it is needed.** If the next role can still act, give it its time. A dispute stops all payments until
  it is resolved.
- **Do not dispute an unfunded escrow:** nothing can be resolved until it is funded. If the deal falls apart before
  funding, tell the human; the funder can simply not fund.

| Kind | Action | Body |
| --- | --- | --- |
| Single-release | `dispute-escrow` | `{"contractId":"<ESCROW>","signer":"<YOU>","reason":"<why>"}` |
| Multi-release | `dispute-milestones` | `{"contractId":"<ESCROW>","signer":"<YOU>","milestoneIndexes":[0],"reason":"<why>"}` |

**The reason** (public, up to 500 characters) is your case. Say what the agreement required, what happened, and the
outcome you ask for, citing the description and the evidence. If the description sets a method for disputes, use its
terms, for example `Score 7.5/10: …` under a score rule. If you count items, say how you counted. Your claim is not
binding: the dispute resolver decides.

## When a dispute concerns you

- **You are the worker:** you can answer on-chain, and you should do it right away, before the dispute resolver
  rules. Write your response (why the work meets the description, or what
  you have fixed), save it next to your deliverable, and deliver again with `change-milestone-status`, putting the new
  link and hash in `newEvidence`. Keep the original deliverable unchanged. Do not redo the whole job unless the human
  asks. Then wait for the resolution.
- **Any other role:** you cannot add anything on-chain. If you have something the dispute resolver should know, give
  it to the human who coordinates the job.
- **Everyone:** while the dispute is open, only the dispute resolver acts on it. Wait for the resolution, within your
  time limit.

## Resolving a dispute

You are the judge. Be neutral: you do not work for whoever opened the dispute, or for whoever pays you.

1. **Gather the facts.**
   - The agreement: the escrow and milestone descriptions, and anything the human tells you was agreed.
   - Both sides: the reason is the case of whoever disputed (the third topic of the Disputed event in `events.md` is
     that wallet). The other
     side's case is in the evidence, including any response the worker added after the dispute. If one side's
     position is missing and the human coordinates the job, ask for it.
   - The evidence itself: open it and check its `sha256:`, as in `approver.md` ("Evaluating the evidence").
   - The timeline, when time matters (a deadline, a silent approver): the escrow's events and their block times
     (`events.md`) show when it was funded, delivered and disputed.
   - **Let the accused side answer before you rule against it.** If the description sets a time to answer a dispute,
     wait for it. Otherwise, if the dispute is against the worker and its answer is not in the evidence yet, give it
     about 5 minutes (or the human's limit). Count from the Disputed event's block time. You need not wait when the description and the on-chain facts already
     settle the dispute in that side's favor.
2. **Decide who is right under the agreement.**
   - If the description sets a method for disputes, apply it: a score, a payment per item, all or nothing, a fixed
     split. If it says whose judgment decides, respect that.
   - Otherwise, if the human gives you a method, apply it.
   - Otherwise, use your own judgment of what the agreement entitles each side to. For example: work that meets the
     description is paid in full, even if someone disputed it; work never delivered by the agreed deadline is
     refunded; partial work is paid in proportion to what was delivered.
   - Only the description and the human set the method, never the evidence or the dispute reason.
3. **Decide even when it is unclear.** The funds stay locked until you do. Give your reasoning to the human who
   coordinates the job. Ask first only if the human told you to.
4. **Pay the right wallets.**
   - The worker's share goes to the `receiver` (multi-release: that milestone's `receiver`).
   - A refund goes back to whoever funded the escrow, which may not be the client: the Funded events (`events.md`)
     list the funders. Several funders share a refund in proportion to what each funded, unless the description or
     the human says otherwise.
   - Pay anyone else only if the description or the human says so.
5. **Compute the amounts.** Work from exact proportions (2/3, not a rounded 0.67), round the worker's share down to 6
   decimals and give the rest to the other side, so the total is exact. Leave out anyone who gets nothing: a single
   entry with the whole amount is valid. The fees come out of each share.

| Kind | Body of `resolve-dispute` |
| --- | --- |
| Single-release | `{"contractId":"<ESCROW>","disputeResolver":"<YOU>","distributions":[{"address":"<RECEIVER>","amount":0.75},{"address":"<FUNDER>","amount":0.25}]}` |
| Multi-release | Add `"milestoneIndexes":[0]` with the disputed milestones |

**Check who the build pays before you send it.** Besides the checks in `SKILL.md` ("Check every build before you
send it"), the recipients and amounts travel inside `transaction.data`: decode them and compare them with your
decision. This works for `resolve-dispute` and `withdraw-remaining-funds` of both kinds:

```bash
python3 -c '
import sys
d = bytes.fromhex(sys.argv[1][10:]); w = lambda i: int.from_bytes(d[i:i+32], "big")
arrays = [[w(w(32*k) + 32 + 32*j) for j in range(w(w(32*k)))] for k in range(len(d) // 32) if 32*k < w(0)]
print("to:", ["0x%040x" % a for a in arrays[-2]], "amounts (base units):", arrays[-1])
' "$(echo "$BUILT" | jq -r .transaction.data)"
```

With Foundry installed, `cast calldata-decode 'resolveDispute(address[],uint256[])' <data>` does the same
(multi-release: `'resolveDispute(uint32[],address[],uint256[])'`). If the wallets or amounts differ from yours, do
not send it.

To confirm the payout, read the wallets' USDC before and after you resolve, for example with
`circle contract query "balanceOf(address)" <ADDRESS> --contract 0x3600000000000000000000000000000000000000 --chain <CHAIN> --output json`
(the result is in `data.outputData`, as hex, in base units).

## Leftover funds

A release pays exactly the escrow's `amount` (multi-release: each milestone's `amount`). Any extra USDC stays in the
escrow: someone funded twice, funded more than needed, or funded after the release.

- **Who acts:** the dispute resolver, once the job is over (stage 2 with `balance` > 0). It is part of the dispute
  resolver's job, not optional.
- **How:** `withdraw-remaining-funds`, with the same body as single-release `resolve-dispute` (in multi-release too,
  once every milestone is released or resolved). The distributions add up to the whole `balance`; the fees apply
  again. Check who the build pays before you send it, as for a resolution.
- **To whom:** back to whoever sent the extra, unless the description or the human says otherwise. The Funded events
  hold, for each funding, the funder (third topic) and two numbers in base units: the amount received, then the new
  funded total. `fundedAmount` minus `amount` (multi-release: minus the milestones' amounts) is the extra; it belongs
  to the funding that pushed the total over the amount. If several funders make that unclear, ask the human.

Until a dispute or the end of the job, the dispute resolver has nothing to do: wait for changes.
