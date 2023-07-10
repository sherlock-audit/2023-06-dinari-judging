ctf_sec

high

# Blocklisted address or address(0) recipient can lock protocol fund when order is partially filled

## Summary

Blocklisted address or address(0) recipient can lock protocol fund when order is partially filled

## Vulnerability Detail

When creating the order, the asset token and payment token is whitelited and validated

```solidity
   function requestOrder(OrderRequest calldata orderRequest, bytes32 salt) public nonReentrant whenOrdersNotPaused {
        // Reject spam orders
        if (orderRequest.quantityIn == 0) revert ZeroValue();
        // Check for whitelisted tokens
        _checkRole(ASSETTOKEN_ROLE, orderRequest.assetToken);
        _checkRole(PAYMENTTOKEN_ROLE, orderRequest.paymentToken);
        bytes32 orderId = getOrderIdFromOrderRequest(orderRequest, salt);
        // Order must not already exist
        if (_orders[orderId].remainingOrder > 0) revert DuplicateOrder();

        // Get order from request and move tokens
        Order memory order = _requestOrderAccounting(orderRequest, orderId);

        // Send order to bridge
        emit OrderRequested(orderId, order.recipient, order, salt);

        // Initialize order state
        uint256 orderAmount = order.sell ? order.assetTokenQuantity : order.paymentTokenQuantity;
        _orders[orderId] = OrderState({requester: msg.sender, remainingOrder: orderAmount, received: 0});
        _numOpenOrders++;
    }

```

but the order.recipient is never validated

the payment token can be USDC and support blocklist, the user can set order.recipient to a blocked token address

the user can also set the recipient address to address(0)

then later when the operator want to fill in the order

when order is partially filled, the transaction proceeded, the share is burned and the payment token is transferred in the OrderProcessor contract

```solidity

    /// @inheritdoc OrderProcessor
    function _fillOrderAccounting(
        OrderRequest calldata orderRequest,
        bytes32 orderId,
        OrderState memory orderState,
        uint256 fillAmount,
        uint256 receivedAmount
    ) internal virtual override {
        // Accumulate fee obligations at each sill then take all at end
        uint256 collection = getPercentageFeeForOrder(receivedAmount);
        uint256 feesEarned = _feesEarned[orderId] + collection;
        // If order completely filled, clear fee data
        uint256 remainingOrder = orderState.remainingOrder - fillAmount;
        if (remainingOrder == 0) {
            // Clear fee state
            delete _feesEarned[orderId];
        } else {
            // Update fee state with earned fees
            if (collection > 0) {
                _feesEarned[orderId] = feesEarned;
            }
        }

        // Burn asset
        IMintBurn(orderRequest.assetToken).burn(fillAmount);
        // Transfer raw proceeds of sale here
        IERC20(orderRequest.paymentToken).safeTransferFrom(msg.sender, address(this), receivedAmount);
        // Distribute if order completely filled
        if (remainingOrder == 0) {
            _distributeProceeds(
                orderRequest.paymentToken, orderRequest.recipient, orderState.received + receivedAmount, feesEarned
            );
        }
    }
```

note the code section:

```solidity
   // Burn asset
        IMintBurn(orderRequest.assetToken).burn(fillAmount);
        // Transfer raw proceeds of sale here
        IERC20(orderRequest.paymentToken).safeTransferFrom(msg.sender, address(this), receivedAmount);
        // Distribute if order completely filled
        if (remainingOrder == 0) {
            _distributeProceeds(
                orderRequest.paymentToken, orderRequest.recipient, orderState.received + receivedAmount, feesEarned
            );
        }
```

only if the remaining order is 0, meaning when the order is fully filled, the payment token is distributed

However, even when the order is filled, and when calling _distributeProceeds, transaction still revert because the order.recipient is a blocklisted addres or address(0)

```solidity
   function _distributeProceeds(address paymentToken, address recipient, uint256 totalReceived, uint256 feesEarned)
        private
    {
        // Check if accumulated fees are larger than total received
        uint256 proceeds = 0;
        uint256 collection = 0;
        if (totalReceived > feesEarned) {
            // Take fees from total received before distributing
            proceeds = totalReceived - feesEarned;
            collection = feesEarned;
        } else {
            // If accumulated fees are larger than total received, then no proceeds go to recipient
            collection = totalReceived;
        }

        // Transfer proceeds to recipient
        if (proceeds > 0) {
            IERC20(paymentToken).safeTransfer(recipient, proceeds);
        }
        // Transfer fees to treasury
        if (collection > 0) {
            IERC20(paymentToken).safeTransfer(treasury, collection);
        }
    }
```

transaction revert when calling this [line of code](https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L171)

```solidity
 IERC20(paymentToken).safeTransfer(recipient, proceeds);
```

there is just no way for protocol to get the fund back because even when canceling the order,

 transaction still revert in this [line of code](https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L171)
 because _distributeProceeds when there are partially fill order
 
```solidity
    /// @inheritdoc OrderProcessor
    function _cancelOrderAccounting(OrderRequest calldata orderRequest, bytes32 orderId, OrderState memory orderState)
        internal
        virtual
        override
    {
        // If no fills, then full refund
        uint256 refund;
        if (orderState.remainingOrder == orderRequest.quantityIn) {
            // Full refund
            refund = orderRequest.quantityIn;
        } else {
            // Otherwise distribute proceeds, take accumulated fees, and refund remaining order
            _distributeProceeds(
                orderRequest.paymentToken, orderRequest.recipient, orderState.received, _feesEarned[orderId]
            );
            // Partial refund
            refund = orderState.remainingOrder;
        }

        // Clear fee data
        delete _feesEarned[orderId];

        // Return escrow
        IERC20(orderRequest.assetToken).safeTransfer(orderRequest.recipient, refund);
    }
```

note, if this line of code:

```solidity
  // Return escrow
        IERC20(orderRequest.assetToken).safeTransfer(orderRequest.recipient, refund);
```

if the orderRequest.recipient address is a blocklisted address by dShares transferRestrictor, the transaction revert as well, blocking order canceling.
 
## Impact

Blocklisted address or address(0) recipient can lock protocol fund when order is partially filled

## Code Snippet

https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/BuyOrderIssuer.sol#L214

https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L171

## Tool used

Manual Review

## Recommendation

We recommend the protocol validate the recipient address to not let user set a blocklisted address or address(0)