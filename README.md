# Canton Stablecoin Framework

[![Daml Version](https://img.shields.io/badge/daml-3.1.0-blue)](https://daml.com)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![CI](https://github.com/your-org/canton-stablecoin-framework/actions/workflows/ci.yml/badge.svg)](https://github.com/your-org/canton-stablecoin-framework/actions/workflows/ci.yml)

The Canton Stablecoin Framework is a comprehensive, production-ready Daml reference implementation for issuing and managing stablecoins on the Canton Network. It provides a robust foundation for launching various types of stablecoins, including fiat-backed, commodity-backed, and yield-bearing tokens, with built-in workflows for issuance, redemption, compliance, and reserve attestation.

This framework is designed to accelerate the launch of regulated and transparent stablecoins for currencies like EUR, GBP, JPY, and commodities such as Gold. It is optimized for deployment on Canton infrastructure, including managed services from providers like [Brale](https://brale.xyz/).

## Key Features

*   **Multi-Currency Support:** Easily configure and launch stablecoins for different underlying assets (e.g., EURc, GBPc, GOLDc).
*   **Role-Based Access Control:** Granular permissions for Issuers, Minting Operators, Redemption Operators, and Compliance Officers.
*   **Atomic Workflows:** Secure and atomic minting, redemption, and transfer operations powered by Daml's smart contract model.
*   **Compliance-First Design:** Integrated compliance gates for KYC/AML, allowing issuers to enforce rules on-ledger.
*   **Reserve Attestation:** On-ledger mechanism for third-party auditors to post attestations, providing transparency to token holders.
*   **Canton-Ready:** Leverages Canton Network's privacy and interoperability features, ensuring transactions are confidential and scalable.

## Why Canton Network?

Canton Network provides the ideal platform for institutional-grade stablecoins:

*   **Privacy:** Canton's privacy model ensures that transaction details are only visible to the involved parties, protecting sensitive financial data.
*   **Scalability:** The network is designed for high-throughput financial workflows, capable of supporting a global stablecoin ecosystem.
*   **Interoperability:** Canton enables seamless composition and settlement across different applications and ledgers, creating a truly interconnected financial network.

## Core Concepts

The framework is built around a set of core Daml templates that model the lifecycle of a stablecoin.

| Template / Concept         | Description                                                                                             |
| -------------------------- | ------------------------------------------------------------------------------------------------------- |
| `IssuerRole`               | A singleton contract granting a party the authority to issue and manage a specific stablecoin.            |
| `OperatorRole`             | Delegated permissions for minting and redemption, allowing operational separation of duties.             |
| `ComplianceRule`           | On-ledger rules defining which parties are authorized to hold and transact the stablecoin (e.g., KYC).  |
| `Stablecoin`               | The digital token contract representing a 1:1 claim on the underlying reserve asset.                    |
| `MintingProposal`          | A proposal/acceptance workflow for creating new stablecoins against proof of reserves.                  |
| `RedemptionRequest`        | A workflow for token holders to request redemption of their stablecoins for the underlying asset.       |
| `ReserveAttestation`       | A contract signed by a trusted auditor, attesting to the state of the issuer's reserves.                |

---

## Issuer Quickstart Guide

This guide walks you through setting up and running the framework to issue your own stablecoin.

### Prerequisites

*   Daml SDK v3.1.0 ([Installation Guide](https://docs.daml.com/getting-started/installation.html))
*   A running Canton Network participant node. For development, you can use the Daml local ledger.

### 1. Clone the Repository

```bash
git clone https://github.com/your-org/canton-stablecoin-framework.git
cd canton-stablecoin-framework
```

### 2. Configure the Stablecoin

The primary configuration happens in the initialization script. Open `daml/Setup.daml` and review the `initialize` script. You can modify the parameters for the `Issuer`, `auditor`, `currencyCode`, etc.

**Example `daml/Setup.daml`:**
```daml
module Setup where

import Daml.Script
import Main.Roles qualified as Roles
import Main.Attestation qualified as Attestation
import Main.Compliance qualified as Compliance

-- ...

initialize : Script ()
initialize = script do
  issuer <- allocateParty "Issuer"
  auditor <- allocateParty "Auditor"
  -- ...

  let
    issuerRole = Roles.IssuerRole
      with
        issuer = issuer
        auditor = auditor
        currencyCode = "EURc"
        description = "Euro-backed Stablecoin"

  -- ... create contracts
```

### 3. Build the Daml Model

Compile the Daml code into a DAR (Daml Archive) file.

```bash
daml build
```
This command creates the file `.daml/dist/canton-stablecoin-framework-0.1.0.dar`.

### 4. Run the Initialization Script

Deploy the initial contracts (like the `IssuerRole`) to the ledger. This script will set up the foundational roles needed to operate the stablecoin.

```bash
daml script \
  --dar .daml/dist/canton-stablecoin-framework-0.1.0.dar \
  --script-name Setup:initialize \
  --ledger-host localhost \
  --ledger-port 6865
```
*Replace `localhost` and `6865` with your Canton participant's host and port.*

After running, the script will output the parties and contract IDs created, which you'll need for subsequent operations.

### 5. Interacting with the Ledger

You can use other `Daml.Script` functions to simulate workflows like minting and transferring.

**Example: Minting Tokens (in a new script)**

1.  Grant an `OperatorRole` to a party.
2.  The operator creates a `MintingProposal`.
3.  The issuer accepts the proposal, which creates new `Stablecoin` contracts.

```daml
-- in a script file
...
let
  operator = ...
  user = ...

-- Issuer gives Operator minting rights
issuerRoleCid <- ... -- Get IssuerRole contract ID
exercise issuerRoleCid Roles.DelegateOperator with operator, canMint = True, canRedeem = False

-- Operator proposes a mint
operatorRoleCid <- ... -- Get OperatorRole contract ID for the operator
exercise operatorRoleCid Roles.ProposeMint with quantity = 1000.0, owner = user

-- Issuer approves the mint
mintProposalCid <- ... -- Get MintingProposal contract ID
exercise mintProposalCid Workflows.AcceptMint
```

## Daml Model Overview

The project is structured into several Daml modules:

*   **`daml/Main/Roles.daml`**: Defines the primary actors and their permissions (`IssuerRole`, `OperatorRole`).
*   **`daml/Main/Stablecoin.daml`**: Contains the core `Stablecoin` template, including transfer and split/merge logic.
*   **`daml/Main/Workflows.daml`**: Implements the multi-stage minting and redemption processes (`MintingProposal`, `RedemptionRequest`).
*   **`daml/Main/Compliance.daml`**: Provides templates for on-ledger KYC/AML, such as the `AllowedPartyClaim`.
*   **`daml/Main/Attestation.daml`**: Defines the `ReserveAttestation` template for auditors to publish reserve data.
*   **`daml/Setup.daml`**: A `Daml.Script` for easy ledger initialization and setup.
*   **`daml/Test.daml`**: Unit and integration tests for all workflows.

## Running Tests

To ensure the correctness of the smart contracts and workflows, run the built-in tests:

```bash
daml test
```

## Integration with Brale

This framework is designed for seamless deployment on Brale's Canton-as-a-Service platform. Brale provides managed Canton participant nodes, JSON API access, and enterprise-grade infrastructure, allowing issuers to focus on their business logic while Brale handles the complexity of running a distributed ledger node.

## Contributing

Contributions are welcome! Please feel free to open a pull request or submit an issue.

1.  Fork the repository.
2.  Create a new feature branch (`git checkout -b feature/my-new-feature`).
3.  Make your changes and add tests.
4.  Ensure all tests pass (`daml test`).
5.  Submit a pull request.

## License

This project is licensed under the Apache License 2.0. See the [LICENSE](LICENSE) file for details.