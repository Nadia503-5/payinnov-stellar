# Payinnov on Stellar: milestones and budget

Three tranches from 5 December 2026 to 21 March 2027. Total budget: $125,000 (tranche 1: $48,900; tranche 2: $47,800; tranche 3: $28,300). Components and building blocks carry the names used in the architecture.

---

## Tranche 1 — The settlement contract and Stellar acceptance: Soroban, Stellar Wallets Kit, Privy and Soroswap implemented ($48,900)

### 1.1 MerchantRegistry and PaymentSplitter

**Description.** The two Soroban contracts of the architecture, in Rust on `soroban-sdk` 29 and the OpenZeppelin `stellar-contracts` 0.8.0 libraries (roles, pause, upgrade with migration). MerchantRegistry: per-merchant configuration (Stellar destination, per-asset Bridge addresses confirmed by the merchant, settlement asset EURC or USDC, kept share, payout threshold, commission), 5% commission cap hard-coded, fee changes delayed by 24 hours with an event, every destination reserved to the merchant's signature. PaymentSplitter: `settle_sale` signed by the customer with an exact amount, `receive_and_settle` under the attribution role (receipt of the Circle message and settlement in the same invocation), `settle_received` for a message presented by a third party, split between commission, accrued local share and kept share, threshold-based `payout` callable by anyone, `PaymentSettled` and `LocalPaidOut` events, replays rejected by `sale_ref`, pause with no withdrawal function, upgrades under a 48-hour delay. Unit tests with mainnet limits, authorisations asserted, fuzzing on the split invariant, mutation testing, continuous integration with `stellar contract build`, `clippy` and Scout static analysis.

**How to measure completion.**
- Both contracts are deployed on testnet, identifiers and WASM hash published in the repository.
- A test sale is split between commission, local share and kept share; the same sale replayed is rejected (hash and test public).
- A fee above the cap is rejected; a destination signed by any account other than the merchant is rejected (public tests).

**Budget.** $17,600

**Delivery date.** 20 December 2026

### 1.2 Stellar payment page and Stellar Wallets Kit

**Description.** The payment page gains the "Pay with a Stellar wallet" button: Stellar Wallets Kit v2 (xBull, Freighter, Hana, Albedo, Lobstr via WalletConnect 2.0), construction of the `settle_sale` invocation with an exact amount in EURC or USDC, simulation through Stellar RPC, checks of the network passphrase, the contract address and the customer's trustline before signing, submission and tracking up to the event. Till QR code that opens the page on the customer's phone with the sale identifier.

**How to measure completion.**
- A complete testnet payment run from the page with two wallets, including Lobstr from a phone (hash published).
- A till QR code scanned with a phone opens the right sale (short video published).
- An invocation against a wrong network or a wrong contract is blocked before signing (public test).

**Budget.** $8,500

**Delivery date.** 27 December 2026

### 1.3 Merchant accounts: Privy, own wallet, sponsored reserves, OpenZeppelin Relayer

**Description.** Merchant onboarding in the dashboard: creation of a Stellar account behind their login through Privy, or connection of their own wallet through the Stellar Wallets Kit; account creation and EURC and USDC trustlines through sponsored reserves from Payinnov's sponsoring account; registration in the MerchantRegistry signed by the merchant's destination. Self-hosted OpenZeppelin Relayer: fee-bumps for the merchant's transactions (slider, destinations) and submission of the orchestrator's transactions with sequence channels. Payinnov's accounts (commission, sponsoring, attribution, onboarding, pricing, pauser, upgrader) created, sensitive roles on multisignature, keys in a secrets manager.

**How to measure completion.**
- A merchant onboarded through Privy and a merchant with their own wallet each have their account and trustlines, sponsored by Payinnov (hashes published).
- The Privy merchant changes their configuration without holding any XLM, fees paid by the relayer (hash published).
- Payinnov's role accounts are published; sensitive roles require two signatures (configuration published).

**Budget.** $11,400

**Delivery date.** 6 January 2027

### 1.4 Soroswap conversion inside the PaymentSplitter and fallback

**Description.** Call to the Soroswap Aggregator from the PaymentSplitter: `swap_tokens_for_exact_tokens` for the exact amount, in the merchant's settlement asset, of a sale paid in the other asset by a customer (surplus returned), `swap_exact_tokens_for_tokens` with a minimum amount for a sale arriving through CCTP; distribution and price taken from the Soroswap API by the payment page or the orchestrator; aggregator address and slippage cap in the contract instance; fallback: call without a route, pass-through in USDC, `converted = false` event. Tests on a network snapshot against the real WASM of the aggregator and of the USDC and EURC SACs.

**How to measure completion.**
- A sale paid in test USDC delivers the exact amount of EURC requested, surplus returned to the customer (hash published).
- With no route available, the sale settles in USDC with the "not converted" event (hash published).
- The tests against the real aggregator contract pass in continuous integration (output published).

**Budget.** $11,400

**Delivery date.** 15 January 2027

---

## Tranche 2 — From the other chains and out to the bank: CCTP V2, Bridge, Mercuryo, Alfred Pay and DeFindex implemented ($47,800)

### 2.1 EVM contract in Stellar settlement mode, CCTP V2 and orchestrator

**Description.** Per-merchant "Stellar settlement" mode on the EVM split contract: `settleToStellar(saleRef, minUsdcOut)` function callable by the operator or the merchant, swap of the asset received into native USDC on the chain's DEX with a minimum amount bounded by the merchant's tolerance, then `depositForBurnWithHook` to domain 27 with `mintRecipient` and `destinationCaller` set to the `CctpForwarder` and the PaymentSplitter as recipient in the hook data, sale identifier as payload. Orchestrator: tracking of EVM sales, call to `settleToStellar`, polling of Circle's attestation, `receive_and_settle` on the PaymentSplitter (bridge receipt and settlement in one invocation), `settle_received` when the message was presented by a third party, six-to-seven decimal conversion, recovery after incidents, per-sale log. Ethereum, Base, Polygon and Sonic.

**How to measure completion.**
- A sale paid on Base Sepolia and one on Ethereum Sepolia arrive settled on Stellar testnet in a single invocation (hashes on both chains published).
- An already-settled sale, or a recipient other than our contract, is rejected (public tests).
- The orchestrator stopped then restarted completes the sale without counting it twice (log published).

**Budget.** $16,500

**Delivery date.** 29 January 2027

### 2.2 Bridge: merchant verification, liquidation addresses, onboarding states

**Description.** Payinnov as a Bridge platform customer; for each merchant: creation of the Bridge customer, hosted verification link, registration of the bank account (IBAN and BIC, or Pix key), creation of the liquidation addresses, one per asset (source Stellar, EURC and USDC, destination SEPA, Pix, SPEI, pounds or Colombian pesos), return address = the merchant's Stellar account; confirmation by the merchant in the dashboard, which writes account and muxed id into the MerchantRegistry with their signature; drain webhooks (`funds_received`, `payment_submitted`, `payment_processed`, `returned`) reconciled with the `PaymentSettled` event and shown in the dashboard; onboarding states `registered`, `Bridge verification in progress`, `payout active`, `alternative payout`; screening of refused activities at onboarding. Against the Bridge API in sandbox.

**How to measure completion.**
- In the Bridge sandbox, a test merchant completes verification, obtains their EURC and USDC addresses and confirms them with their signature (identifiers and hash published).
- A payout to their Bridge address shows up as a drain in the dashboard, next to the on-chain event (screenshots published).
- An unverified merchant receives their whole sale on their Stellar account; an invalid or replayed webhook is rejected (public tests).

**Budget.** $11,400

**Delivery date.** 8 February 2027

### 2.3 Mercuryo, Alfred Pay and DeFindex from the dashboard

**Description.** Three dashboard actions signed by the merchant from their account (Privy or own wallet): sale of USDC to card through the Mercuryo widget (iframe, off-ramp enabled, merchant KYC at Mercuryo); withdrawal in reais via Pix or pesos via SPEI through Alfred Pay (merchant verification at Alfred, order created by the dashboard, transfer to the order's deposit address, tracking up to payout); deposit and withdrawal of the kept share into a DeFindex vault (`depositToVault`, `withdrawFromVault`, vault shares on the merchant's account, balance displayed). Common payout adapter (`quote`, `initiate`, `status`), one implementation per partner.

**How to measure completion.**
- A DeFindex deposit and withdrawal run from the dashboard by a Privy merchant (hashes published).
- A test Alfred Pay order funded in Stellar USDC and tracked to its final state (identifiers published).
- The Mercuryo widget opens in sandbox with Stellar USDC preselected for sale (screenshot published).

**Budget.** $11,400

**Delivery date.** 18 February 2027

### 2.4 Detection, FEC export, threat model and monitoring plan

**Description.** Reading of `PaymentSettled` and `LocalPending` events through Stellar RPC with a persistent cursor, catch-up through Horizon after a stop, sale confirmation to the till and the plugin, reconciliation of `LocalPaidOut` payouts with Bridge drains, exceptions in the dashboard (sale without event, payout without drain, drain in error), FEC export fed by the same stream. Threat model and monitoring plan published: threats by category with their mitigation (Circle message, commission cap, destinations, replay, degraded swap, CCTP recipient, decimals, role keys, code upgrade), OpenZeppelin Monitor configured on both contracts with the architecture's alerts.

**How to measure completion.**
- A testnet sale is confirmed at the till when the event arrives and appears in the FEC export (file published).
- The event reader stopped then restarted catches the sale up without duplicates (log published).
- The threat model and the monitoring plan are public; a test alert fires in OpenZeppelin Monitor (public test).

**Budget.** $8,500

**Delivery date.** 25 February 2027

---

## Tranche 3 — Production: Soroban, Stellar Wallets Kit, Privy, Soroswap, CCTP V2, Bridge, Mercuryo, Alfred Pay and DeFindex live ($28,300)

### 3.1 Contracts and payment page in production, first merchants settled on Stellar

**Description.** MerchantRegistry and PaymentSplitter deployed on mainnet with a SEP-55 verified build (metadata, GitHub attestation, release WASM); Payinnov's accounts and roles in production, multisignature on sensitive roles; payment page in mainnet configuration; USDC/EURC pool seeded on Soroswap; the current merchants who wish to are onboarded (Stellar account, destinations) and switched to Stellar settlement for native Stellar payments.

**How to measure completion.**
- Both contracts are on mainnet with their SEP-55 verified build (identifiers and attestation published).
- A sale of a few euros run by the team from the payment page, with Soroswap conversion (hash published).
- Adoption, from go-live to the submission of this tranche: at least 5 distinct merchant wallets, excluding team accounts, have received a sale settled by the contract (list of accounts published).
- Adoption, same window: at least 300 settled transactions and a cumulative settled volume of at least EUR 20,000 equivalent (list of hashes published); any shortfall is reported as is.

**Budget.** $11,400

**Delivery date.** 7 March 2027

### 3.2 CCTP, Bridge, Mercuryo, Alfred Pay and DeFindex in production

**Description.** "Stellar settlement" mode of the EVM contract deployed on Ethereum, Base, Polygon and Sonic, orchestrator in production with Circle's mainnet addresses; Payinnov verified by Bridge in production, first merchants verified, SEPA and Pix liquidation addresses written into the registry; Mercuryo widget and Alfred Pay withdrawals in production; DeFindex vault in production; OpenZeppelin Monitor alerts active and alert channel documented.

**How to measure completion.**
- A sale paid in USDC on Base is settled on Stellar mainnet in a single invocation (hashes on both chains published).
- A real Bridge payout to a team merchant produces a processed drain (identifier and hash published).
- A DeFindex deposit, an Alfred Pay order and a Mercuryo widget opening in production (hashes and screenshots published); a test alert received on the documented channel (screenshot published).

**Budget.** $8,500

**Delivery date.** 14 March 2027

### 3.3 E-commerce plugins and till app released with Stellar payment

**Description.** Released versions of the WooCommerce, PrestaShop and Magento plugins and of the till app with Stellar payment switchable on by the merchant: Stellar button on the payment page, till QR code to the page, confirmation on the event, display of the kept share and payouts; public integration guide for a merchant (onboarding, Bridge verification, slider, withdrawals) and for a developer (API, confirmation webhooks, `PaymentSettled` event).

**How to measure completion.**
- The three plugins and the till app are released with Stellar payment; a payment of a few euros by the team from each of them on mainnet (hashes published).
- A developer from outside the team completes a full testnet payment by following the public guide (write-up published).
- The FEC export of a team merchant contains their Stellar sales for the month (anonymised file published).

**Budget.** $8,400

**Delivery date.** 21 March 2027
