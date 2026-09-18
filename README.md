# Centrifuge Deployer

One page, `public/index.html`, that performs the one-time hardening of the
deployer key's namespace inside the `DeployGate` on a new chain. It is done once
per chain, before anything is committed there.

Served at **https://deployer.centrifugelabs.io** by a Cloudflare Worker with
static assets (`centrifuge-deployer`, Labs account). There is no Worker code and
no build step: `wrangler.jsonc` points at `public/`, and Cloudflare serves what
is there without invoking a script. A push to `main` is the whole deploy.

Everything about the hosting is declared here, including the Custom Domain — so
Cloudflare owns the DNS record for the hostname and nothing about it lives in
[`centrifuge/ops-v3`](https://github.com/centrifuge/ops-v3). See that
repository's `cloudflare/terraform/README.md` for the rule this follows.

## What it does

Pick a chain the gate is deployed on, or paste an RPC endpoint for one that is
not, and the page reads every prerequisite off that endpoint and puts a button
on each step that is still missing:

- **CreateX** and the **DeployGate** itself, deploying the gate where there is
  none.
- **Chain compatibility** — Cancun opcodes (`TSTORE`/`TLOAD`, `MCOPY`, `PUSH0`,
  `BLOBBASEFEE`), a block explorer with a verification API, the LayerZero DVNs a
  deployment needs live, and the rest of the infrastructure checklist.
- **The admin Safes** — deploying each where it is missing by replaying its
  original mainnet creation, so it lands at the same address, then reconciling
  its owner set and threshold against Ethereum's in one batched transaction.
- **The namespace** — the 6h delegate delay, and the Protocol and Ops Safes as
  delegates.

The deployer key stays the account every protocol address derives from, while
the Safes are the accounts that actually sign a commit phase. A leaked delegate
can then commit, but not before the delay has run out, and the namespace revokes
it in one transaction.

`script/deploy/README.md` in the protocol repository has the shape of the thing;
this page is the recipe with a wallet behind it.

## Editing it

It is deliberately one self-contained file: no build, no dependencies, no
bundler. Open `public/index.html`, change it, commit it.

Two things to keep in mind:

- **Everything it signs is one-shot and irreversible.** Every action simulates
  against the endpoint with `eth_call` before it is sent, and the confirmation
  screen is the only place a human sees what they are about to sign — so what it
  displays has to be decoded from the same bytes that get sent, not from a
  constant that happens to agree.
- **It reads a chain it may know nothing about.** Balances, gas prices and
  opcode support are probed rather than assumed, and a chain that answers
  strangely should degrade to a visible "unknown", never to a confident wrong
  number.

## Verifying what is served

The page is also pinned on IPFS, which is what makes a served copy checkable
against a reviewed one: hash the bytes and compare the CID.

```bash
curl -s https://deployer.centrifugelabs.io/ -o served.html
ipfs add --only-hash --cid-version 1 served.html
```

A local file and the deployed page should give the same CID as the pin recorded
for that commit.
