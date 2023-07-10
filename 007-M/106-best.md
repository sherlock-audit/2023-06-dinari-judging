auditsea

medium

# `orderId` needs to be unique

## Summary
`orderId` can be duplicated, which will cause issues on off-chain operating service.

## Vulnerability Detail
`orderId` is generated from `orderRequest` and `salt` both given by a user, it means there can be multiple orders with same `orderId` that can cause some problems on off-chain operating service.(Even though there is only one active order with one orderId)

## Impact
This will cause some problems on off-chain operating service, especially when `orderId` is used as a primary key of database used by off-chain operating service. For example:

1. `requestCancel` would cause confusions to the off-chain operating service. When a user creates an order and tries to cancel it, but let's say requestCancel is called twice by a mistake, off-chain service cancels the order, and after a delay, off-chain service tries to handle 2nd cancellation event. If user creates a same order between two cancellation events, 2nd order will also be cancelled even though both cancellation events were for the first order.
2. When `orderId` is used as a primary key on off-chain database, user's order history with same fields will not be recorded, causing lose of history.

## Code Snippet
https://github.com/sherlock-audit/2023-06-dinari/blob/main/sbt-contracts/src/issuer/OrderProcessor.sol#L244-L264

## Tool used

Manual Review

## Recommendation
Use randomly on-chain generated salt (not recommended) or implement user nonce that increases whenever orders are created.
