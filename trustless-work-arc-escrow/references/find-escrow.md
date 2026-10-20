# Finding a new escrow

Read this when someone else is about to create the escrow and you do not know its address.

Watch the factory for the `EscrowCreated` event of the wallet that will send the `deploy`. That is usually the
client; if an orchestrator deploys for the client, ask the human for that wallet.
- Start about 1,000 blocks back (roughly 8 minutes), so you do not miss an escrow created just before you started.
- The new escrow is the second topic of the event.

```bash
RPC=https://rpc.testnet.arc.io
FACTORY=0x829fE7040797Df140636901e79F1bA448B7e9ae0   # single-release. Multi-release: 0x7B04302DCEC430D74e8e38B35Cef96EbaD51b7fC
CLIENT_TOPIC=0x000000000000000000000000<client address in lowercase, without 0x>
SKIP=''   # escrows you already checked and rejected: lowercase addresses separated by commas
# First run only: compute FROM and note it. Later runs: FROM='<the value you noted>'
NOW=$(curl -s -X POST $RPC -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' | jq -r .result)
FROM=$(printf '0x%x' $(( NOW - 1000 )))
for i in $(seq 1 10); do
  FOUND=$(curl -s -X POST $RPC -H 'content-type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"eth_getLogs","params":[{
    "address": "'$FACTORY'", "fromBlock": "'$FROM'", "toBlock": "latest",
    "topics": ["0x6e235dc8d7bb9d0676726c636a9423d8a06a9948a695ffc1e48550812a7ad8ad", null, "'$CLIENT_TOPIC'"]}]}' \
    | jq -r --arg skip "$SKIP" '(.result // []) | reverse | .[].topics[1] | "0x" + .[26:]
        | select(. as $a | ($skip | split(",") | index($a)) | not)')
  [ -n "$FOUND" ] && { echo "$FOUND" | while read -r a; do echo "ESCROW=$a"; done; break; }
  sleep 10
done
```

Repeat runs until it prints `ESCROW=…` lines (newest first), within your time limit, keeping the same `FROM` and
adding to `SKIP` from one run to the next. One query covers at most 9,999 blocks (about 80 minutes): for a longer
watch, move `FROM` forward. Then read each escrow through the API:
- **Yours:** it names you in the role you expect and it is not closed.
- **Not yours** (closed, for example a job the client finished earlier, or it does not name you as expected): add it
  to `SKIP` and keep watching. The loop then ignores it and waits for a newer one.
- **Before working on it:** if the human told you the agreed price or terms, check that the escrow matches them. If
  it does not, tell the human instead of working.
- **Several open escrows from that client name you:** each one is a separate job. Compare each description with the
  job the human gave you, and work on the ones that match. If more than one matches and the human gave you a single
  job, ask the human which one.

Anyone can create an escrow that names you, so only work on escrows created by the client, or the admin, that the
human told you about.
