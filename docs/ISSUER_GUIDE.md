# Canton Stablecoin Framework: Issuer's Guide

This guide provides a comprehensive walkthrough for financial institutions, fintechs, and other organizations looking to issue their own stablecoin on the Canton Network using this reference framework. It covers the core concepts, deployment procedures, and operational best practices.

## Table of Contents

1.  [Introduction](#1-introduction)
2.  [Prerequisites](#2-prerequisites)
3.  [Framework Components](#3-framework-components)
    -   [Core Models: `Stablecoin` and `Account`](#core-models-stablecoin-and-account)
    -   [Compliance: `ComplianceGate`](#compliance-compliancegate)
    -   [Transparency: `ReserveAttestation`](#transparency-reserveattestation)
    -   [Innovation: `YieldBearing`](#innovation-yieldbearing)
4.  [Getting Started: Local Setup](#4-getting-started-local-setup)
5.  [Configuration & Deployment](#5-configuration--deployment)
    -   [Step 1: Define Your Asset](#step-1-define-your-asset)
    -   [Step 2: Allocate Parties](#step-2-allocate-parties)
    -   [Step 3: Deploy the DAR](#step-3-deploy-the-dar)
    -   [Step 4: Instantiate the Stablecoin Contract](#step-4-instantiate-the-stablecoin-contract)
6.  [Operational Workflows](#6-operational-workflows)
    -   [User Onboarding (KYC/AML)](#user-onboarding-kycaml)
    -   [Minting Process](#minting-process)
    -   [Burning/Redemption Process](#burningredemption-process)
    -   [Publishing Reserve Attestations](#publishing-reserve-attestations)
    -   [Updating Yield Rates (for Yield-Bearing Tokens)](#updating-yield-rates-for-yield-bearing-tokens)
7.  [Integrating with Off-Chain Systems](#7-integrating-with-off-chain-systems)
    -   [Using the JSON API](#using-the-json-api)
    -   [Querying with the Participant Query Store (PQS)](#querying-with-the-participant-query-store-pqs)
8.  [Brale Infrastructure](#8-brale-infrastructure)
9.  [Security Best Practices](#9-security-best-practices)

---

## 1. Introduction

The Canton Stablecoin Framework is a production-ready, open-source Daml project designed to accelerate the launch of regulated and transparent stablecoins. It provides a modular architecture that supports various models:

-   **Fiat-Backed:** Pegged 1:1 to currencies like EUR, USD, GBP, JPY.
-   **Commodity-Backed:** Backed by assets like gold (XAU) or silver (XAG).
-   **Yield-Bearing:** Tokens that automatically accrue yield based on the underlying reserve assets.

This framework enforces on-ledger compliance checks, provides hooks for transparent reserve reporting, and uses standard Canton patterns for secure and atomic settlement.

## 2. Prerequisites

Before you begin, ensure you have:

1.  **A Canton Participant Node:** You will need access to a participant node connected to a Canton network (e.g., DevNet, TestNet, or MainNet). We recommend using a managed infrastructure provider like [Brale](https://brale.xyz).
2.  **Daml SDK (DPM):** The Digital Asset Package Manager (DPM) is required to build and test the smart contracts. Follow the [official installation instructions](https://docs.digitalasset.com/dpm/index.html).
3.  **Legal & Regulatory Clearance:** Issuing a stablecoin is a regulated activity. Ensure you have the necessary licenses, legal opinions, and a clear operational structure.
4.  **Custody & Auditing Partners:** You must have established relationships with a custodian to hold the reserve assets and an independent auditor to perform attestations.

## 3. Framework Components

The framework is composed of several key Daml modules that work together.

### Core Models: `Stablecoin` and `Account`

-   **`Stablecoin.Stablecoin`:** This is the central control contract for your token. It is a singleton contract (only one should exist per asset) signed by the `issuer`. It tracks the `totalSupply` and defines the core administrative actions:
    -   `Mint`: Creates new tokens.
    -   `Burn`: Destroys tokens.
    -   `SetCompliance`: Updates the compliance officer party.
    -   `SetAuditor`: Updates the auditor party.

-   **`Stablecoin.Account`:** This template represents a user's balance. Each token holder has one or more `Account` contracts. It includes the fundamental `Transfer` choice for peer-to-peer payments. The proposal/accept pattern is used for transfers to ensure the recipient agrees to receive the funds.

### Compliance: `ComplianceGate`

This module provides a flexible on-chain allowlist mechanism to enforce KYC/AML rules.

-   **`ComplianceGate.Rule`:** A control contract, created by the `issuer`, that designates a `compliance` officer party. This officer is authorized to manage KYC statuses.
-   **`ComplianceGate.KYCStatus`:** A contract created by the `compliance` officer for each user who has passed verification. Core choices like `Mint` and `Transfer` require the existence of a valid `KYCStatus` contract for the involved parties, preventing unauthorized activity.

### Transparency: `ReserveAttestation`

This module enables auditors to publish proof-of-reserve reports directly on the ledger.

-   **`ReserveAttestation.Rule`:** A control contract, created by the `issuer`, that designates an `auditor` party.
-   **`ReserveAttestation.Attestation`:** The `auditor` creates instances of this contract to publish reports. It contains the `reserveAmount`, the `reportDate`, a URL to the full report, and a cryptographic hash of the report for integrity. The `Stablecoin.Stablecoin` contract can be updated to reference the latest attestation, providing a transparent link between the on-chain supply and off-chain reserves.

### Innovation: `YieldBearing`

This optional component adds functionality for yield-bearing stablecoins.

-   **`YieldBearing.Rule`:** A control contract that defines the yield-bearing properties and is controlled by the `issuer`.
-   **`YieldBearing.Rebase` Choice:** This choice on the `Rule` contract allows the issuer to update the `exchangeRate`. This rate represents the value of one token unit in terms of the underlying reserve asset. By increasing the rate, all token holders' balances effectively grow in value without requiring any transactions on their part.

## 4. Getting Started: Local Setup

Clone the repository and run a local test environment to familiarize yourself with the code.

```bash
# 1. Clone the repository
git clone https://github.com/digital-asset/canton-stablecoin-framework.git
cd canton-stablecoin-framework

# 2. Install the correct Daml SDK version (as specified in daml.yaml)
# Example: dpm install 3.4.0

# 3. Build the Daml code into a DAR (Daml Archive) file
dpm build

# 4. Run the Daml Script tests to see workflows in action
dpm test
```

The tests in `daml/test/StablecoinTest.daml` provide excellent examples of how to orchestrate the different templates and choices.

## 5. Configuration & Deployment

### Step 1: Define Your Asset

Before deployment, customize the stablecoin's properties. This is done when you instantiate the main `Stablecoin.Stablecoin` contract. You will need to define:

-   `issuer`: Your organization's `Party` ID.
-   `custodian`: The `Party` ID of your banking or custody partner.
-   `assetSymbol`: e.g., "EURC", "GBPC", "XAUC".
-   `assetDescription`: e.g., "Canton-native Euro Stablecoin".
-   `isYieldBearing`: `True` or `False`.

### Step 2: Allocate Parties

You need to allocate unique `Party` identifiers on the Canton network for each role. Your Canton infrastructure provider (e.g., Brale) will provide a dashboard or API for this.

-   `Issuer Party`: Your primary operational party.
-   `Compliance Party`: The party responsible for KYC/AML checks.
-   `Auditor Party`: The party for your independent auditor.
-   `Custodian Party`: The party representing the reserve holder (optional but good practice).

End-users will also need their own parties to hold `Account` contracts.

### Step 3: Deploy the DAR

Once built, the DAR file (`.daml/dist/canton-stablecoin-framework-*.dar`) must be uploaded to your participant node. This is typically done via your provider's dashboard or API.

### Step 4: Instantiate the Stablecoin Contract

After deploying the DAR, you must create the singleton contracts that govern your stablecoin. This is a one-time setup process. You can use Daml Script or the JSON API.

**Example Daml Script for Instantiation:**

```daml
-- In a setup script
module Setup where

import Daml.Script
import Stablecoin.V1.Stablecoin
import Stablecoin.V1.ComplianceGate
import Stablecoin.V1.ReserveAttestation

setup : Script ()
setup = do
  issuer <- allocateParty "Issuer"
  compliance <- allocateParty "Compliance"
  auditor <- allocateParty "Auditor"
  custodian <- allocateParty "Custodian"

  -- Create the core stablecoin contract
  stablecoinCid <- submit issuer do
    createCmd Stablecoin.Stablecoin with
      issuer
      custodian
      compliance = Some compliance
      auditor = Some auditor
      assetSymbol = "EURC"
      assetDescription = "Canton-native Euro Stablecoin"
      totalSupply = 0.0
      isYieldBearing = False

  -- Create the compliance rule
  complianceRuleCid <- submit issuer do
    createCmd ComplianceGate.Rule with
      issuer
      compliance

  -- Create the attestation rule
  attestationRuleCid <- submit issuer do
    createCmd ReserveAttestation.Rule with
      issuer
      auditor

  return ()
```

## 6. Operational Workflows

### User Onboarding (KYC/AML)

1.  A new user provides their identity information to you off-chain.
2.  Your compliance team verifies the user's identity.
3.  The designated `Compliance Party` creates a `ComplianceGate.KYCStatus` contract on the ledger for the user's `Party` ID.
    -   This contract acts as a prerequisite for most other actions.

### Minting Process

The minting process is a two-step, "push" model to ensure the user receives exactly what they expect.

1.  **Off-Chain Deposit:** A user wires fiat currency (e.g., 10,000 EUR) to your designated bank account.
2.  **Mint Request:** Your backend system detects the deposit and, acting as the `issuer`, creates a `Stablecoin.MintRequest` contract with the user as the `owner`.
3.  **User Approval:** The user sees the pending mint request in their wallet/application and exercises the `ApproveMint` choice on the `MintRequest` contract.
4.  **Token Issuance:** Atomically, the `MintRequest` is consumed, a `Stablecoin.Account` contract for 10,000 EURC is created for the user, and the `totalSupply` on the main `Stablecoin` contract is increased.

### Burning/Redemption Process

Redemption follows a similar two-step process.

1.  **Burn Request:** A user, through their wallet, initiates a redemption by exercising the `RequestBurn` choice on their `Stablecoin.Account` contract. This archives their `Account` and creates a `Stablecoin.BurnRequest` contract, with the `issuer` as the controller.
2.  **Off-Chain Payout:** Your backend system is notified of the `BurnRequest`. You process the fiat payout (e.g., wire 10,000 EUR) to the user's bank account.
3.  **Burn Approval:** Once the payout is confirmed, your system, acting as the `issuer`, exercises the `ApproveBurn` choice on the `BurnRequest`.
4.  **Token Destruction:** Atomically, the `BurnRequest` is consumed, and the `totalSupply` on the main `Stablecoin` contract is decreased.

### Publishing Reserve Attestations

1.  Your `Auditor` completes their periodic review of your reserve assets.
2.  The `Auditor Party` creates a new `ReserveAttestation.Attestation` contract on the ledger, containing the report details.
3.  (Optional) You, as the `issuer`, can update the `Stablecoin` contract to point to this new attestation contract ID, making it easy for anyone to find the latest report.

### Updating Yield Rates (for Yield-Bearing Tokens)

1.  Based on the performance of your reserve assets, you calculate a new `exchangeRate`.
2.  As the `issuer`, you exercise the `Rebase` choice on the `YieldBearing.Rule` contract, providing the new rate.
3.  The contract is updated. From this point forward, the redemption value of all tokens reflects the new, higher rate.

## 7. Integrating with Off-Chain Systems

Your existing infrastructure will interact with your Canton participant node primarily via the JSON API and the Participant Query Store (PQS).

### Using the JSON API

Your application server will make authenticated calls to the JSON API to perform actions like:
-   Creating `KYCStatus` contracts.
-   Creating `MintRequest` contracts after fiat deposits.
-   Exercising `ApproveBurn` after fiat payouts.
-   Querying for active contracts (e.g., pending `BurnRequest`s).

### Querying with the Participant Query Store (PQS)

For analytics, reporting, and populating user-facing dashboards, the PQS is the ideal tool. It synchronizes ledger data to a PostgreSQL database, allowing for complex SQL queries.

**Example PQS Query:** Get the total balance for a specific user.
```sql
SELECT sum(payload ->> 'amount') AS total_balance
FROM active('Stablecoin.V1.Stablecoin:Account', '${userPartyId}');
```

## 8. Brale Infrastructure

[Brale](https://brale.xyz) provides enterprise-grade infrastructure for Canton, simplifying many of the operational complexities:

-   **Managed Participants:** Fully managed, high-availability participant nodes.
-   **Dashboard:** An intuitive UI for uploading DARs, managing parties, and monitoring your node.
-   **Wallet Gateway:** A CIP-0103 compliant gateway that allows you to connect your stablecoin to a wide range of dApps and wallets in the Canton ecosystem.
-   **APIs:** Robust APIs for programmatic party management and node operations.

Using a provider like Brale allows you to focus on your business logic and customer experience rather than managing low-level infrastructure.

## 9. Security Best Practices

-   **Key Management:** The private keys for the `issuer`, `compliance`, and `auditor` parties are extremely sensitive. Use a hardware security module (HSM) or a trusted digital asset custody solution.
-   **API Security:** Secure the connection between your application servers and the Canton participant node. Use JWTs with short expiry times and whitelist IP addresses.
-   **Daml Code Audits:** Before deploying to production, have your Daml code audited by a reputable third-party firm.
-   **Disaster Recovery:** Have a clear disaster recovery plan for your participant node and the associated off-chain systems.
-   **Role Separation:** Strictly enforce the principle of least privilege. The `compliance` party should only be able to perform compliance actions, and the `auditor` only auditing actions. The Daml authorization model enforces this on-ledger.