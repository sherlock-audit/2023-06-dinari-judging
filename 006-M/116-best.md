peanuts

high

# Double accounting of remainingOrder when filling order which may result in underflow

## Summary

`orderState.remainingOrder` deducts fillAmount twice when filling order, resulting in accounting error

## Vulnerability Detail

When `fillOrder()` is called, `remainingOrder` is calculated by subtracting `fillAmount` from `orderState.remainingOrder`. If `remainingOrder` is not equal to 0, `_orders[orderId].remainingOrder is updated to the `remainingOrder` amount. Then, `_fillOrderAccounting` is called.

```solidity
        // Update order state
>      uint256 remainingOrder = orderState.remainingOrder - fillAmount;
        // If order is completely filled then clear order state
        if (remainingOrder == 0) {
            // Notify order fulfilled
            emit OrderFulfilled(orderId, orderRequest.recipient);
            // Clear order state
            delete _orders[orderId];
            _numOpenOrders--;
        } else {
            // Otherwise update order state
 >         _orders[orderId].remainingOrder = remainingOrder;
            _orders[orderId].received = orderState.received + receivedAmount;
        }
      _fillOrderAccounting(orderRequest, orderId, orderState, fillAmount, receivedAmount);
```

When `_fillOrderAccounting()` is called, it calls `_fillBuyOrder()` and the function subtracts `remainingOrder` from `fillAmount` again.

```solidity
    function _fillBuyOrder(
        OrderRequest calldata orderRequest,
        bytes32 orderId,
        OrderState memory orderState,
        uint256 fillAmount,
        uint256 receivedAmount
    ) internal virtual {
        FeeState memory feeState = _feeState[orderId];
>      uint256 remainingOrder = orderState.remainingOrder - fillAmount;
        // If order is done, close order and transfer fees
```

For example, let's say for a buy market order, `order.paymentTokenQuantity` is 1000 so `remainingOrder` is 1000.
```solidity
        // Initialize order state
        uint256 orderAmount = order.sell ? order.assetTokenQuantity : order.paymentTokenQuantity;
        _orders[orderId] = OrderState({requester: msg.sender, remainingOrder: orderAmount, received: 0});
```

`fillAmount` is 700. When the operator calls  `fillOrder()`, 1000 is subtracted from 700 twice, which results in underflow and revert. By right, `orderAmount` should be 300 ( how much is left to fill), and should not be subtracted again in `_fillBuyOrder`.

## Impact

`remainingOrder` is subtracted by `fillAmount` twice, which might make the calculation underflow.

## Code Snippet

https://github.com/sherlock-audit/2023-06-dinari/blob/main/sbt-contracts/src/issuer/OrderProcessor.sol#L289-L302

## Tool used

Manual Review

## Recommendation

Recommend only subtracting `remainingOrder` once and not subtracting it again in `_fillBuyOrder`.