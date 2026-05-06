# VRF v2.5 Security and Best Practices

## Use requestId for Fulfillment Matching

Match randomness fulfillments to requests using `requestId`, not request order. Validators can control the order in which transactions are included, so assuming first-in first-out is unsafe.

## Choose a Safe Block Confirmation Count

`requestConfirmations` controls how many blocks must be mined before your request is fulfilled. Higher values make rewrite attacks more expensive. Tune to your application's risk level:
- Low-stakes games: 3 (minimum)
- NFT mints, mid-value prizes: 5–10
- High-value lotteries: 20+

## Do Not Allow Re-Requesting or Cancellation

Once a request is made, do not allow re-requesting before fulfillment. Allowing cancellation lets a party discard unfavorable outcomes and retry until they get a good one.

```solidity
// BAD — cancellation allows bias
function rollDie() external {
    if (pendingRequest) { cancelRequest(); }
    requestRandomWords(false);
}

// GOOD — enforce one-pending-at-a-time, no cancellation path
function rollDie() external {
    require(!s_requests[lastRequestId].exists || s_requests[lastRequestId].fulfilled, "request pending");
    requestRandomWords(false);
}
```

## Cut Off Inputs After Requesting

Do not accept bids, bets, or user-supplied inputs after a randomness request is in flight. An attacker who learns a rewrite is happening can submit additional inputs to exploit the changed outcome.

## fulfillRandomWords Must Not Revert

If `fulfillRandomWords` reverts, the VRF service will not retry — the randomness is lost. Keep callback logic minimal: store the random values and emit an event. Execute downstream logic (winner selection, token minting) in a separate transaction.

```solidity
// GOOD — store first, act later
function fulfillRandomWords(uint256 requestId, uint256[] calldata randomWords) internal override {
    s_results[requestId] = randomWords;
    emit RandomnessFulfilled(requestId);
    // winner selection happens in a separate claimPrize() call
}
```

## Never Use Block Randomness as Fallback

`block.prevrandao`, `block.difficulty`, `blockhash`, and `block.timestamp` are manipulable by validators and must never be used as randomness sources or fallbacks. See [Why not RANDAO / block.prevrandao?](https://stackoverflow.com/questions/73938799/chainlink-vrf-or-randao) for a detailed explanation.

```solidity
// NEVER DO THIS
uint256 rand = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender)));
uint256 rand = uint256(blockhash(block.number - 1));
```

## Access Control on requestRandomWords

Restrict who can trigger requests. An unrestricted `requestRandomWords` lets anyone drain your subscription.

```solidity
function requestRandomWords(bool enableNativePayment) external onlyOwner returns (uint256) { ... }
```

## Subscription Balance

Keep subscription balance well above the minimum buffer. If balance drops to minimum during concurrent requests, new requests may stall and restart only after funds are added — which itself can take time.

## Avoid ERC-4337 Smart-Account Wallets for Subscription Management

Pre-signed operations on account-abstracted wallets can execute during your callback, creating race conditions. Use an EOA or standard multisig for subscription management.

## Gas Limit Sizing

**Always measure `callbackGasLimit` on testnet before mainnet.**

If the callback exceeds the limit it reverts and the randomness is lost — you cannot re-request the same values. Rough estimates:
- Storage write: ~20,000 gas each
- Event emission: ~375 gas + 8 gas/byte
- External call: 2,300 gas minimum

Add a 20–30% buffer. Maximum allowed: 2,500,000 gas.

## Testing with VRFCoordinatorV2_5Mock

For unit tests, use the mock coordinator from `@chainlink/contracts`:

```solidity
import {VRFCoordinatorV2_5Mock} from "@chainlink/contracts/src/v0.8/vrf/mocks/VRFCoordinatorV2_5Mock.sol";

contract MyConsumerTest is Test {
    VRFCoordinatorV2_5Mock coordinator;
    MyConsumer consumer;

    function setUp() public {
        // baseFee, gasPriceLink, weiPerUnitLink
        coordinator = new VRFCoordinatorV2_5Mock(100000000000000000, 1000000000, 4113797966605025);
        uint256 subId = coordinator.createSubscription();
        coordinator.fundSubscription(subId, 10 ether);
        consumer = new MyConsumer(address(coordinator), subId, keyHash);
        coordinator.addConsumer(subId, address(consumer));
    }

    function test_requestAndFulfill() public {
        uint256 requestId = consumer.requestRandomWords(false);
        // Simulate fulfillment
        coordinator.fulfillRandomWords(requestId, address(consumer));
        // Assert results
        (, uint256[] memory words) = consumer.getRequestStatus(requestId);
        assertEq(words.length, 2);
    }
}
```

## Production Checklist

- [ ] Tested end-to-end on testnet with real VRF fulfillments
- [ ] `callbackGasLimit` measured on testnet with a 20–30% buffer
- [ ] `requestConfirmations` set for the risk level of your application
- [ ] No re-requesting or cancellation after a request is made
- [ ] No input acceptance after a request is in flight
- [ ] `fulfillRandomWords` cannot revert; downstream logic is deferred
- [ ] No fallback to block-based randomness
- [ ] Subscription balance monitoring and alerting configured
- [ ] Security audit completed by an independent auditor

This example code is **unaudited** and provided for educational purposes only.
