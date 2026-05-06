# VRF v2.5 Official Sources

Use these URLs when reference files do not contain the specific information needed. Prefer `.md` endpoints — they return markdown directly and consume far fewer tokens than HTML pages.

## Core Documentation (use .md endpoints)

| Topic                                | URL                                                                    |
| ------------------------------------ | ---------------------------------------------------------------------- |
| VRF Overview                         | https://docs.chain.link/vrf.md                                         |
| VRF v2.5 Overview                    | https://docs.chain.link/vrf/v2-5/overview.md                           |
| Subscription — Get a Random Number   | https://docs.chain.link/vrf/v2-5/subscription/get-a-random-number.md   |
| Direct Funding — Get a Random Number | https://docs.chain.link/vrf/v2-5/direct-funding/get-a-random-number.md |
| Migrating from V2 to v2.5            | https://docs.chain.link/vrf/v2-5/migration-from-v2.md                  |
| Billing                              | https://docs.chain.link/vrf/v2-5/billing.md                            |
| Supported Networks & Addresses       | https://docs.chain.link/vrf/v2-5/supported-networks.md                 |
| Security Considerations              | https://docs.chain.link/vrf/v2-5/security.md                           |
| Subscription Management (UI)         | https://vrf.chain.link                                                 |

## Contract Source Code

Contracts live in the [chainlink-evm](https://github.com/smartcontractkit/chainlink-evm) repo. Use a tagged release, not `develop`:

```
https://github.com/smartcontractkit/chainlink-evm/blob/contracts-v1.5.0/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol
https://github.com/smartcontractkit/chainlink-evm/blob/contracts-v1.5.0/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol
https://github.com/smartcontractkit/chainlink-evm/blob/contracts-v1.5.0/contracts/src/v0.8/vrf/dev/VRFV2PlusWrapperConsumerBase.sol
https://github.com/smartcontractkit/chainlink-evm/blob/contracts-v1.5.0/contracts/src/v0.8/vrf/mocks/VRFCoordinatorV2_5Mock.sol
```

See [chainlink-evm releases](https://github.com/smartcontractkit/chainlink-evm/releases) for available `contracts-v*` tags.

## npm Package

```bash
npm install @chainlink/contracts
```
