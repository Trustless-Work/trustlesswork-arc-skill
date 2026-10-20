# Escrow events on Arc

Read this when you need to know who did what, or to confirm something that happened before you arrived. To wait for
your turn, the snapshot check in `SKILL.md` is simpler.

Every action emits an event from the escrow's address. Read them with `eth_getLogs` on the Arc RPC.

```bash
RPC=https://rpc.testnet.arc.io
FROM=$(curl -s -X POST $RPC -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' | jq -r .result)
# Later, as often as needed:
curl -s -X POST $RPC -H 'content-type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"eth_getLogs","params":[{
  "address": "<ESCROW>", "fromBlock": "'$FROM'", "toBlock": "latest",
  "topics": ["<EVENT_TOPIC>", null, "<WHO_AS_TOPIC>"]}]}' | jq '(.result // []) | length'
```

- To watch from now on, note the current block as `FROM` before you start. To look back, set `FROM` to a block before
  the event (Arc makes about two blocks per second).
- **One query covers at most 9,999 blocks** (about 80 minutes). A longer range may come back as an empty list instead
  of an error, which looks like "nothing happened": split it into windows of under 9,999 blocks.
- `<WHO_AS_TOPIC>` is a wallet padded to 32 bytes: `0x000000000000000000000000` followed by the address in
  lowercase without `0x`. Use `null` to accept anyone.
- One or more results means the event happened. If the RPC answers without a `result` (it limits bursts of
  queries), wait a few seconds and ask again.

In every event below the wallet that acted is the **third topic** (`topics[2]`); the second is the hashed `engagementId`.

| Event (emitted by the escrow) | `<EVENT_TOPIC>` | `<WHO_AS_TOPIC>` (third topic) is |
| --- | --- | --- |
| Funded | `0x006d268809962dd9189e3397ca104a8badcff21cf02a88952e7b9209a9506952` | The funder |
| Milestone status changed (delivered) | `0x00a8d8c498227bb3223788365fb5916fdcfe3af83856f2896cd79cc3250cbff7` | The service provider |
| Milestones approved | `0x847ab1098a1f167bc9ec8f7c7b5bb492ae18b63730283841d2efdf6ff4d720e5` | The approver |
| Released (single-release) | `0xe27309e0f154d08e8a038ada2637155af73c4e1b2341679a004e180980b50960` | The release signer |
| Released (multi-release) | `0x7fea0778d781d380c260633bb73596acbcf32bf830fcfafc038ba96c73e24157` | The release signer |
| Disputed (single-release) | `0x5f610a9f802495e3f111f40e1eeee03788ef757060d842b50a0e0148b3da3131` | Who disputed |
| Disputed (multi-release) | `0x8e7959eaaab499e7d93f7f1c5ba2b73f4489dbb0b1b40e0f28ade7f1384c7a0d` | Who disputed |
| Dispute resolved (single-release) | `0x13868768fe1b1830400d5fdbb0e547f1720db228fd5cab195be146974bb713d1` | The dispute resolver |
| Dispute resolved (multi-release) | `0x99eac35394f026dd2f3f77bd5b380f3c87738178a583f56939ba28d2b5356e44` | The dispute resolver |

In the single-release Released event, the fourth topic is the receiver. Use it to confirm a payment that happened
before you arrived.

An event tells you that something happened, not the whole state. Before you act, read the escrow through the API to
confirm it, for example that the approvals reached `target`.

## When it happened

To time a deadline, read the block time of the event (or of any transaction): each log has a `blockNumber`.

```bash
curl -s -X POST $RPC -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_getBlockByNumber","params":["<BLOCK_NUMBER_HEX>",false]}' \
  | jq -r .result.timestamp | xargs printf '%d\n'   # Unix seconds, UTC
```

## Who got paid

A release, a resolution or a withdrawal moves USDC out of the escrow. To see each payment of one transaction, read
the USDC `Transfer` logs in its receipt:

```bash
curl -s -X POST $RPC -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_getTransactionReceipt","params":["<TX_HASH>"]}' \
  | jq -r '.result.logs[] | select((.address | ascii_downcase) == "0x3600000000000000000000000000000000000000"
      and .topics[0] == "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef")
      | "to 0x\(.topics[2][26:]) amount \(.data)"'
```

The amount is hex, in base units (`printf '%d\n' <amount>`). One of the transfers is the fee: 0.3% to Trustless Work,
plus the platform fee if there is one. This is more precise than comparing balances, which other payments can change.
