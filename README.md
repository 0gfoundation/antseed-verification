# 0G Confidential on AntSeed — ownership and TEE evidence

Peer ID `4f79b161cade6fc250d89c81e21fef37385d8a42` (display name "0G Confidential") is operated by 0G.
The ownership proof in [antseed.json](antseed.json) and this DNS TXT record bind that identity to the
`0gfoundation` GitHub organization and the `0g.ai` domain:

```text
_antseed.0g.ai TXT "antseed-peer=4f79b161cade6fc250d89c81e21fef37385d8a42"
```

## What the seller sells

`z-ai/glm-5.3`, served by 0G's own GLM-5.3 deployment running inside a TDX confidential VM with the
GPUs in NVIDIA Confidential Computing mode. Prices (USD per million tokens): input 1.40, cached input
0.26, output 4.40.

The seller forwards each request to `https://router-api.0g.ai`, whose TLS terminates inside the 0G E2EE
gateway (a dstack CVM). For GLM-5.3 the gateway seals the request to the model provider's enclave key
before it leaves the gateway, so the 0G router in between only relays ciphertext. Only models on the
gateway's sealing list are listed on this peer, with category `confidential`; models the gateway relays
in the clear are not sold here.

OpenAI Chat Completions is the advertised protocol; the AntSeed buyer's protocol adapter also serves
Anthropic Messages (Claude Code, including streaming tool use) and OpenAI Responses (Codex) requests to
this peer.

## What a buyer can verify

1. **Seller node** (`seller-node-tee-genuine`, `seller-bound`):

   ```bash
   antseed buyer start \
     --peer 4f79b161cade6fc250d89c81e21fef37385d8a42 \
     --verifiers antseed-verifier \
     --require-verifier
   ```

   The node claim DCAP-verifies the seller CVM's TDX quote against Intel's root, checks the TCB status and
   that debug mode is off, and checks the quote's binding to the buyer's nonce and this peer ID.
   `seller-bound` checks the seller's AntSeed identity signature over the nonce and the evidence bundle.

2. **Gateway** (`seller-provider-tee-genuine`): the seller fetches a fresh
   quote from the 0G E2EE gateway using the buyer's nonce. The verifier DCAP-verifies it and checks that
   `report_data[0:32] = SHA-256(nonce_hex ‖ e2e_pubkey_b64)` (`nonce-pubkey-sha256-v1`), where
   `e2e_pubkey` is the SPKI of the gateway's TLS certificate. This claim is informational by default; to
   require all three claims, start the buyer with
   `ANTSEED_VERIFIER_REQUIRED_CAPS=seller-node-tee-genuine,seller-bound,seller-provider-tee-genuine`.

3. **Gateway, independently**: the gateway publishes a cert-binding attestation for `router-api.0g.ai`.
   `pcverify` (from [0g-pc-e2ee](https://github.com/0gfoundation/0g-pc-e2ee), build it from source) checks
   that the TLS certificate is held inside a genuine TDX enclave and that the CVM runs a published release:

   ```bash
   git clone https://github.com/0gfoundation/0g-pc-e2ee
   cd 0g-pc-e2ee/client/cmd/pcverify && go build -o pcverify .
   ./pcverify -gateway router-api.0g.ai -strict
   ```

## What is NOT verified

- The AntSeed verifier does not compare the seller CVM's measurements with approved software. With an
  acceptable TCB, `seller-node-tee-genuine` passes for any genuine, non-debug TDX VM bound to your nonce
  and this peer ID.
- The seller's TLS connection to the gateway is not pinned to the attested key, so the attestations do not
  prove that your request took that path.
- The gateway quote proves the gateway, not the model provider behind it. The gateway itself verifies the
  provider on every request (TDX quote, allowlisted OS image, on-chain signer, response signature), but a
  buyer cannot check that through AntSeed.
- Which models the gateway seals is gateway configuration that is not part of its measured compose. Each
  response carries the gateway's sealing marker (`x-0g-e2ee: sealed` at the time of writing); a buyer can
  only check it after the request has been sent.
- Responses are not individually signed by an attested key on the AntSeed path, and streamed responses
  are delivered before the provider's end-of-stream signature is checked.
- A CPU TDX quote does not by itself prove GPU execution; the `confidential` category is our declaration.

## Who sees plaintext

The buyer's local proxy, the seller CVM, the 0G E2EE gateway, and the model provider's CVM. All three
server-side parties run in TDX; the 0G router between the gateway and the provider sees ciphertext only.

## Primary sources

- Ownership proof: [antseed.json](antseed.json) and `dig TXT _antseed.0g.ai`.
- Seller discovery metadata: `antseed network peer 4f79b161cade6fc250d89c81e21fef37385d8a42`.
- AntSeed verifier: [`packages/antseed-verifier`](https://github.com/AntSeed/antseed/tree/main/packages/antseed-verifier)
  (`caps/tee-tdx.ts`, `caps/seller-bound.ts`, `report-data.ts`).
- Gateway verification guide: [`docs/verifying-the-gateway.md`](https://github.com/0gfoundation/0g-pc-e2ee/blob/main/docs/verifying-the-gateway.md).
