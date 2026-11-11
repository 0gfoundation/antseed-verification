# 0G Confidential on AntSeed — ownership and TEE evidence

Peer ID `375f87baeb630772ff2d82292bc96a71dc333929` (display name "0G Confidential") is operated by 0G.
The ownership proof in [antseed.json](antseed.json) and this DNS TXT record bind that identity to the
`0gfoundation` GitHub organization and the `0g.ai` domain:

```text
_antseed.0g.ai TXT "antseed-peer=375f87baeb630772ff2d82292bc96a71dc333929"
```

## What the seller sells

`z-ai/glm-5.3`, served by 0G's own GLM-5.3 deployment running inside a TDX confidential VM with the
GPUs in NVIDIA Confidential Computing mode. Prices (USD per million tokens): input 1.40, cached input
0.35, output 4.40.

The seller forwards each request to `https://router-api.0g.ai`, whose TLS terminates inside the 0G E2EE
gateway (a dstack CVM). For GLM-5.3 the gateway seals the request to the model provider's enclave key
before it leaves the gateway, so the 0G router in between only relays ciphertext. Only models on the
gateway's sealing list are listed on this peer, with category `confidential`; models the gateway relays
in the clear are not sold here.

OpenAI Chat Completions is the advertised protocol; the AntSeed buyer's protocol adapter also serves
Anthropic Messages (Claude Code, including streaming tool use) and OpenAI Responses (Codex) requests to
this peer.

## How the seller node is deployed

The seller runs Antseed's official TEE seller image
(`ghcr.io/antseed/tee-seller@sha256:6ea1b6ef3d644e7fa0fd8c9e4c860d68ec136244cc379012029ac88172996f54`,
release 0.1.240, built by Antseed's `tee-seller-image.yml` workflow from `main` with signed build
provenance) as a dstack app with KMS on the dstack 0.6.1 production OS (`os_image_hash`
`ec43cb8c5e04b6bb6c53b33b16076a01d83d01f7f4b703f2f78a0d947d6247e6`), with no pre-launch or init script.
The compose file, including the seller config, the upstream URL and the prices, is sent to buyers with
every attestation. Only the seller's identity key and its 0G router API key come in as encrypted
environment variables.

## What a buyer can verify

1. **Seller node** (`seller-node-tee-genuine`, `seller-bound`):

   ```bash
   antseed buyer start \
     --peer 375f87baeb630772ff2d82292bc96a71dc333929 \
     --verifiers antseed-verifier \
     --require-verifier
   ```

   The node claim DCAP-verifies the seller CVM's TDX quote against Intel's root, checks the TCB status and
   that debug mode is off, and checks that the quote is bound to the buyer's nonce, this peer ID and the
   buyer's own encrypted connection. With antseed-verifier 0.1.6 or later it also checks the CVM's OS
   measurements (MRTD, RTMR0-2) against the dstack releases the verifier approves, replays RTMR3 to the
   app compose, and checks that the compose runs an approved Antseed seller image under the TEE seller
   rules. `seller-bound` checks the seller's AntSeed identity signature over the nonce and the evidence
   bundle.

   **Known issue (November 2026):** this seller's host runs QEMU 9.2.1. Its RTMR1 and RTMR2 match the
   official dstack 0.6.1 release, but its MRTD and RTMR0 depend on the host's QEMU version and virtual
   hardware and are not yet in the verifier's approved list, which was computed for one host profile
   (QEMU 8.2.2). Buyers on antseed-verifier 0.1.6 or later therefore currently report
   `seller-node-tee-genuine` as failed for this peer. Supporting other host profiles is being discussed
   in [Antseed/antseed#1571](https://github.com/Antseed/antseed/pull/1571).

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

- The seller's TLS connection to the gateway is not pinned to the attested key: the seller uses ordinary
  certificate validation and does not check that the gateway's TLS key equals the attested `e2e_pubkey`.
  The attestations therefore do not prove that your request took that path.
- On the AntSeed path the gateway's quote is checked for genuineness and for its binding to your nonce and
  its TLS key, but not against approved gateway measurements; use `pcverify` (below) for that.
- The seller's identity key is provisioned by the operator through an encrypted environment variable, not
  generated inside the CVM. Signatures made with it (the evidence bundle and the per-response receipts)
  prove possession of that key, not that the signing happened inside the enclave.
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
- Seller discovery metadata: `antseed network peer 375f87baeb630772ff2d82292bc96a71dc333929`.
- AntSeed verifier: [`packages/antseed-verifier`](https://github.com/AntSeed/antseed/tree/main/packages/antseed-verifier)
  (`caps/tee-tdx.ts`, `caps/seller-bound.ts`, `report-data.ts`).
- Gateway verification guide: [`docs/verifying-the-gateway.md`](https://github.com/0gfoundation/0g-pc-e2ee/blob/main/docs/verifying-the-gateway.md).
