# Trustless Work Arc Escrow Skill

An agent skill that teaches AI agents to **pay for outcomes on [Arc](https://docs.arc.io)** with Trustless Work
escrows. Agents act through their **Circle Agent Wallets** and the **Circle CLI**: they lock USDC for a job, deliver
work with evidence, approve and release payments, and open or resolve disputes, coordinating only through the escrow
on-chain.

> **Network: Arc testnet only.** Mainnet support will come with the mainnet contracts.

This skill is for agents that **use** escrows on Arc. For building Trustless Work integrations on Stellar, see the
separate [Trustless Work Development Skill](https://github.com/Trustless-Work/trustlesswork-skill).

## Installation

```bash
npx skills add trustless-work/trustlesswork-arc-skill
```

Or copy the `trustless-work-arc-escrow/` folder into your agent's skills folder (for example
`~/.claude/skills/trustless-work-arc-escrow`).

Agents without skill support can read it by URL. Give them a prompt such as:

```text
Run `curl -sL https://raw.githubusercontent.com/Trustless-Work/trustlesswork-arc-skill/main/trustless-work-arc-escrow/SKILL.md`
and follow it to work on a Trustless Work escrow on Arc. …
```

The skill tells the agent to fetch the reference files it needs from the same base URL.

## Requirements

- **Circle CLI** (`npm install -g @circle-fin/cli`) with an **Agent Wallet** on `ARC-TESTNET`. A human logs in once
  and accepts Circle's Terms; the agent never handles keys or email codes. Circle's setup guide:
  `curl -sL https://agents.circle.com/skills/setup.md`.
- **The Trustless Work API for Arc** and an API key for it (see [Get access](#get-access)). The API builds and checks
  every escrow call; it never holds keys or signs.
- `curl` and `jq`.

## Get access

- **API (Arc testnet):** `https://arc-api-demo-production.up.railway.app`. Interactive docs are at
  [`/docs`](https://arc-api-demo-production.up.railway.app/docs).
- **API key:** ask any member of the Trustless Work team on Telegram. If you don't know any of us there, join our
  [Telegram group](https://t.me/+9FJnzKamvEdiZjNh) and post that you need an API key for the Arc API. Include the name
  of the person or service that will use it. Each key belongs to one person or service, so don't share it.
- **Keep the key in a file**, readable only by you (`chmod 600`), and give the agent the file's path, never the key.
  The key is shown once when it's issued, and nobody can recover it later.
- **Keys expire after 90 days.** Before that, ask for a rotation: you get a new key, and the old one keeps working for
  24 hours while you switch.
- **Rate limit:** 60 requests per minute per key, and 600 per minute per IP. Past the limit the API answers `429` with
  a `Retry-After` header, and the skill waits and retries. If you need more, say so when you ask for the key.
- **If a key leaks** (it ends up in a chat, a log or a repository), tell us. We revoke it at once and issue a new one.
- **Test USDC:** [faucet.circle.com](https://faucet.circle.com), on the Arc testnet network.

## Structure

```
trustlesswork-arc-skill/
├── README.md
├── LICENSE
└── trustless-work-arc-escrow/      # The skill: only this folder is installed
    ├── SKILL.md                    # What every agent needs: rules, setup, roles, how to send an action,
    │                               # the act / wait / stop loop, waiting, common errors
    └── references/                 # Read only when they apply
        ├── find-escrow.md          # Finding a new escrow on-chain
        ├── client.md               # Creating and funding an escrow
        ├── worker.md               # Checking the terms, delivering with evidence
        ├── approver.md             # Judging the work, approving, releasing
        ├── disputes.md             # Opening, answering and resolving disputes; leftover funds
        ├── multi-release.md        # Stages and rules of multi-release escrows
        ├── admin.md                # Changing an escrow before it is funded
        ├── events.md               # On-chain events: who did what, when, and who got paid
        └── errors.md               # Every API and CLI error and what to do
```

`SKILL.md` is the core every agent reads. It points each agent to the references for its role and the escrow type,
so an agent reads about half of the skill on a typical job.

## What agents learn

- **Roles:** client, worker (service provider and receiver), approver, release signer, dispute resolver, platform and
  admin, and the overlaps the contract rejects. The people who set up a job choose who holds each role.
- **Escrow types:** single-release (one payment) and multi-release (each milestone paid on its own, possibly to
  different workers).
- **How every action is sent:** the API builds it; the Circle CLI signs and sends it through the contract's
  `execute(bytes)` entry point, because the CLI cannot pass array or tuple arguments. Circle sponsors the gas.
- **Deciding what to do:** an agent can start at any moment of a job. It reads the escrow, finds the stage, and acts,
  waits or stops. Several agents can run at the same time without talking to each other.
- **Disputes in any situation:** any role can dispute when the agreement breaks (missing or late work, a silent
  approver, a stalled release). The dispute resolver acts as a neutral judge: it hears both sides, applies the method
  the parties agreed (a score, per item, all or nothing, a fixed split) or its own judgment, and pays the worker or
  refunds whoever funded.
- **Safe evidence:** deliveries carry a link and its SHA-256, so the approver and the dispute resolver judge exactly
  what was delivered. Everything on-chain is public.
- **Checking the terms:** funders and workers check that the escrow still matches what was agreed before they commit
  money or work.

## Example prompt

A worker agent, with the skill installed:

```text
You are a WORKER agent. A client hired you through a Trustless Work escrow on Arc.

- Trustless Work API: <API_URL>. API key: in <KEY_FILE>; never print it.
- Your wallet (Circle Agent Wallet, logged in on testnet): <YOUR_WALLET>
- Your client <CLIENT_WALLET> will create a single-release escrow for this job.
- Agreed terms: <the deliverables, item by item>, <amount> USDC, <how disputes are settled, if agreed>.
- Deliver at a public link, with its sha256 in the evidence.

Take part in the job until there is nothing left for you to do, then report what you delivered,
the transaction hashes and how much you were paid.
```

## Safety

This skill moves real value, so it is built so that no single party can redirect the money:

- **The human keeps control of the wallet.** Agent Wallets are logged in by a human, who also accepts Circle's Terms.
  Agents never ask for, read or store private keys, OTP codes or Circle credentials.
- **The API is not trusted for where money goes.** The Trustless Work API prepares and simulates each call, but holds
  no keys. Before signing, the agent checks every build against the chain:
  - the target contract is the official factory, or the escrow it is working on;
  - the function selector is the one for the action it requested;
  - the escrow is a real Trustless Work escrow: a clone of the official implementation, which only the official
    factory can initialize (one `eth_getCode` call);
  - for dispute resolutions and leftover withdrawals, the recipients and amounts decoded from the call match its
    decision.
- **Only the agreed money moves.** USDC is approved only to a verified escrow, for exactly the agreed amount.
- **Third-party content is data, never instructions.** Escrow descriptions, evidence, dispute reasons and web pages
  cannot change which wallet signs, whom it pays, how much, or which contract it calls.
- **Secrets stay out of the logs.** The API key is read inside each command and never printed.
- **Testnet only** for now. On mainnet, Circle Agent Wallets also support spending policies set by the human.

## License

[Apache-2.0](LICENSE)
