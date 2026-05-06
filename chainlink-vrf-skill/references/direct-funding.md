# VRF v2.5 Direct Funding Method

## Overview

The direct funding method lets a consumer contract pay for each VRF request directly, without maintaining a subscription. The contract must hold enough LINK or native tokens before calling `requestRandomWords`. Billing is **upfront** — the cost is estimated and charged at request time.

Use direct funding when:

- You need a one-off randomness request.
- You don't want to manage a subscription account.
- Each request is infrequent enough that subscription overhead is not worth it.

For recurring requests, prefer the subscription method (`subscription.md`).

**Official docs:** https://docs.chain.link/vrf/v2-5/direct-funding/get-a-random-number.md

## Complete Consumer Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {VRFV2PlusWrapperConsumerBase} from "@chainlink/contracts/src/v0.8/vrf/dev/VRFV2PlusWrapperConsumerBase.sol";
import {VRFV2PlusClient} from "@chainlink/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol";
import {LinkTokenInterface} from "@chainlink/contracts/src/v0.8/shared/interfaces/LinkTokenInterface.sol";
import {ConfirmedOwner} from "@chainlink/contracts/src/v0.8/shared/access/ConfirmedOwner.sol";

contract VRFDirectFundingConsumer is VRFV2PlusWrapperConsumerBase, ConfirmedOwner {
    event RequestSent(uint256 requestId, uint32 numWords);
    event RequestFulfilled(uint256 requestId, uint256[] randomWords, uint256 payment);

    error RequestNotFound(uint256 requestId);
    error WithdrawFailed();

    struct RequestStatus {
        uint256 paid;       // amount paid in juels (LINK) or wei (native)
        bool fulfilled;
        uint256[] randomWords;
        bool native;
    }

    mapping(uint256 => RequestStatus) public s_requests;
    uint256[] public requestIds;
    uint256 public lastRequestId;

    uint32 public callbackGasLimit = 100_000;
    uint16 public requestConfirmations = 3;
    uint32 public numWords = 2;

    // Pass only the wrapper address — no LINK token address (v2.5 change from V2)
    constructor(address wrapperAddress)
        ConfirmedOwner(msg.sender)
        VRFV2PlusWrapperConsumerBase(wrapperAddress)
    {}

    /**
     * @param enableNativePayment true = pay in native token, false = pay in LINK.
     * The contract must hold sufficient balance of the chosen token before calling.
     */
    function requestRandomWords(
        bool enableNativePayment
    ) external onlyOwner returns (uint256 requestId) {
        bytes memory extraArgs = VRFV2PlusClient._argsToBytes(
            VRFV2PlusClient.ExtraArgsV1({nativePayment: enableNativePayment})
        );

        uint256 reqPrice;
        if (enableNativePayment) {
            (requestId, reqPrice) = requestRandomnessPayInNative(
                callbackGasLimit,
                requestConfirmations,
                numWords,
                extraArgs
            );
        } else {
            (requestId, reqPrice) = requestRandomness(
                callbackGasLimit,
                requestConfirmations,
                numWords,
                extraArgs
            );
        }

        s_requests[requestId] = RequestStatus({
            paid: reqPrice,
            fulfilled: false,
            randomWords: new uint256[](0),
            native: enableNativePayment
        });
        requestIds.push(requestId);
        lastRequestId = requestId;
        emit RequestSent(requestId, numWords);
        return requestId;
    }

    // VRFV2PlusWrapperConsumerBase uses memory (not calldata) for randomWords
    function fulfillRandomWords(
        uint256 _requestId,
        uint256[] memory _randomWords
    ) internal override {
        if (s_requests[_requestId].paid == 0) revert RequestNotFound(_requestId);
        s_requests[_requestId].fulfilled = true;
        s_requests[_requestId].randomWords = _randomWords;
        emit RequestFulfilled(_requestId, _randomWords, s_requests[_requestId].paid);
    }

    function getRequestStatus(
        uint256 _requestId
    ) external view returns (uint256 paid, bool fulfilled, uint256[] memory randomWords) {
        if (s_requests[_requestId].paid == 0) revert RequestNotFound(_requestId);
        RequestStatus memory request = s_requests[_requestId];
        return (request.paid, request.fulfilled, request.randomWords);
    }

    function withdrawLink() external onlyOwner {
        LinkTokenInterface link = LinkTokenInterface(i_linkAddress);
        uint256 balance = link.balanceOf(address(this));
        bool success = link.transfer(msg.sender, balance);
        if (!success) revert WithdrawFailed();
    }

    function withdrawNative() external onlyOwner {
        uint256 balance = address(this).balance;
        (bool success, ) = payable(msg.sender).call{value: balance}("");
        if (!success) revert WithdrawFailed();
    }

    receive() external payable {}
}
```

## Key Differences from Subscription

| Aspect                     | Subscription                       | Direct Funding                                |
| -------------------------- | ---------------------------------- | --------------------------------------------- |
| Base contract              | `VRFConsumerBaseV2Plus`            | `VRFV2PlusWrapperConsumerBase`                |
| Funding                    | Central subscription account       | Contract holds tokens directly                |
| Billing timing             | Post-fulfillment (actual gas used) | Upfront (estimated at request time)           |
| Request return             | Single `uint256 requestId`         | Tuple `(uint256 requestId, uint256 reqPrice)` |
| Constructor                | Takes coordinator address          | Takes **only** wrapper address                |
| `fulfillRandomWords` param | `uint256[] calldata`               | `uint256[] memory`                            |

## v2.5 Constructor Change (Important)

V2 wrapper constructor required both LINK and wrapper addresses:

```solidity
// V2 — DO NOT USE
VRFV2WrapperConsumerBase(linkAddress, wrapperAddress)
```

V2.5 wrapper constructor takes only the wrapper address:

```solidity
// v2.5 — CORRECT
VRFV2PlusWrapperConsumerBase(wrapperAddress)
```

## Funding the Contract

**LINK:**
Fund the contract by transferring LINK to its address. The contract must hold LINK before calling `requestRandomWords(false)`.

**Native token:**

```solidity
// Send ETH to the contract; it accepts via receive()
(bool ok,) = contractAddress.call{value: 0.01 ether}("");
```

Costs vary significantly with gas prices — always check on the target network before deploying.

## Security Notes

- Ensure the contract holds sufficient balance **before** calling `requestRandomWords`. The call reverts if underfunded.
- The `paid` field in `RequestStatus` records the upfront cost — use it for accounting.
- Get the wrapper address from `supported-networks.md` or https://docs.chain.link/vrf/v2-5/supported-networks.md — never hardcode it without verifying.
- This example code is **unaudited**. Conduct a security audit before production deployment.
