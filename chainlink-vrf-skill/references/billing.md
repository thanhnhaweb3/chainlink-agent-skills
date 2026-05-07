# VRF v2.5 Billing

## Premium Percentages

VRF v2.5 charges a premium on top of the base gas cost:

| Payment Token | Premium (Ethereum Mainnet) |
|---|---|
| LINK | 20% |
| Native coin (ETH, MATIC, etc.) | 24% |

Native coin payment carries a slightly higher premium than LINK. If cost efficiency matters, pay in LINK.

Note: Premium percentages may differ on other networks. Check https://docs.chain.link/vrf/v2-5/billing.md for network-specific values.

## Billing Timing

### Subscription Method
Billing is **post-fulfillment**. The actual gas consumed during the callback is measured and deducted from the subscription balance after `fulfillRandomWords` completes. You are charged for gas actually used, not an estimate.

### Direct Funding Method
Billing is **upfront**. The cost is estimated at request time and charged when `requestRandomWords` is called. The contract must hold sufficient balance before the call, or it will revert.

## Cost Formula

### Subscription
```
total cost = gas_price × (verification_gas + callback_gas_used) × ((100 + premium%) / 100)
```

### Direct Funding
```
total gas cost = (Gas price * (Coordinator gas overhead + Callback gas limit + Wrapper gas overhead))
total request cost = (Coordinator premium + (total gas cost) * ((100 + Premium percentage) / 100))
```

Where `callback_gas_limit` is the value you set — you pay for the full limit even if your callback uses less.

## Choosing LINK vs Native Coin

Ask the user which option they prefer (LINK or native coin), default to LINK.

Per-request payment method is controlled by the `nativePayment` flag in `extraArgs` (subscription) or the `requestRandomnessPayInNative` function (direct funding). Subscriptions must be funded with the corresponding token.

## Funding a Subscription

**With LINK (ERC-677 transferAndCall):**
```solidity
// Programmatic funding
LINK.transferAndCall(
    coordinatorAddress,
    linkAmount,
    abi.encode(subscriptionId)
);
```

Or instruct the user to fund via the UI at https://vrf.chain.link.

**With native coin:**
```solidity
coordinator.fundSubscriptionWithNative{value: amount}(subscriptionId);
```

## Withdrawal from Subscription

```solidity
// Withdraw LINK
coordinator.cancelSubscription(subscriptionId, receivingAddress);

// Or just withdraw excess without cancelling — use the VRF UI
```

## PegSwap: Polygon and BNB Chain

On **Polygon** and **BNB Chain**, the LINK token from the canonical bridge is **not ERC-677 compatible**. You must convert it to ERC-677 LINK using PegSwap before it can be used to fund a VRF subscription or direct-funding contract.

- PegSwap: https://pegswap.chain.link
- Convert bridged LINK (ERC-20) → native LINK (ERC-677) before funding.

This only applies to bridge-sourced LINK. LINK purchased directly on these chains is already ERC-677.
