ctf_sec

high

# Escrow record not cleared on cancellation and order fill

## Summary
In `DirectBuyIssuer.sol`, a market buy requires the operator to take the payment token as escrow prior to filling the order. Checks are in place so that the math works out in terms of how much escrow has been taken vs the order's remaining fill amount. However, if the user cancels the order or fill the order, the escrow record is not cleared. 

The escrow record will exists as a positive amount which can lead to accounting issues.

## Vulnerability Detail
Take the following example:

- Operator broadcasts a `takeEscrow()` transaction around the same time that the user calls `requestCancel()` for the order
- Operator also broadcasts a `cancelOrder()` transaction
- If the `cancelOrder()` transaction is mined before the `takeEscrow()` transaction, then the contract will transfer out token when it should not be able to.

`takeEscrow()` simply checks that the `getOrderEscrow[orderId]` is less than or equal to the requested amount:
```solidity
        bytes32 orderId = getOrderIdFromOrderRequest(orderRequest, salt);
        uint256 escrow = getOrderEscrow[orderId];
        if (amount > escrow) revert AmountTooLarge();


        // Update escrow tracking
        getOrderEscrow[orderId] = escrow - amount;
        // Notify escrow taken
        emit EscrowTaken(orderId, orderRequest.recipient, amount);


        // Take escrowed payment
        IERC20(orderRequest.paymentToken).safeTransfer(msg.sender, amount);
```

Cancelling the order does not clear the `getOrderEscrow` record:
```solidity
    function _cancelOrderAccounting(OrderRequest calldata order, bytes32 orderId, OrderState memory orderState)
        internal
        virtual
        override
    {
        // Prohibit cancel if escrowed payment has been taken and not returned or filled
        uint256 escrow = getOrderEscrow[orderId];
        if (orderState.remainingOrder != escrow) revert UnreturnedEscrow();


        // Standard buy order accounting
        super._cancelOrderAccounting(order, orderId, orderState);
    }
}
```

This can lead to an good-faith and trusted operator accidentally taking funds from the contract that should not be able to leave.

coming up with the fact that the transaction does not have deadline or expiration date:

consider the case below:

1. a good-faith operator send a transaction, takeEscrow
2. the transaction is pending in the mempool for a long long long time
3. then user fire a cancel order request
4. the operator help user cancel the order
5. the operator send a transcation cancel order
6. cancel order transaction land first
7. the takeEscrow transaction lands

because escrow state is not clear up, the fund (other user's fund) is taken 

It's also worth noting that the operator would not be able to call `returnEscrow()` because the order state has already been cleared by the cancellation. `getRemainingOrder()` would return **0**.

```solidity
    function returnEscrow(OrderRequest calldata orderRequest, bytes32 salt, uint256 amount)
        external
        onlyRole(OPERATOR_ROLE)
    {
        // No nonsense
        if (amount == 0) revert ZeroValue();
        // Can only return unused amount
        bytes32 orderId = getOrderIdFromOrderRequest(orderRequest, salt);
        uint256 remainingOrder = getRemainingOrder(orderId);
        uint256 escrow = getOrderEscrow[orderId];
        // Unused amount = remaining order - remaining escrow
        if (escrow + amount > remainingOrder) revert AmountTooLarge();
```

## Impact
- Insolvency due to pulling escrow that should not be allowed to be taken

## Code Snippet
https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/DirectBuyIssuer.sol#L130-L142

## Tool used
Manual Review

## Recommendation
Clear the escrow record upon canceling the order.