# Issue H-1: Blocklisted address or address(0) recipient can lock protocol fund when order is partially filled 

Source: https://github.com/sherlock-audit/2023-06-dinari-judging/issues/55 

## Found by 
ctf\_sec, hals
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



## Discussion

**jaketimothy**

Agree and would argue further that even with an initial check, a user could be added to blocklist after order placement but before fulfillment, creating an unrecoverable state for that order as refunds would be blocked. Not sure how to handle that..

**ctf-sec**

I think when cancel the order, instead of pushing the fund to receiver, can let receiver claim the fund, if after a certain period and the receiver does not claim the fund, 

the protocol can help them claim the fund, we assume the protocol admin msg.sender will not be blocklisted

Also, I think when creating the order, it is ok to validate the receiver address is not address(0)

**ctf-sec**

@Oot2k 

My empathy for judge is high, but I want to suggest an alternative way to de-duplicate the issue.

I don't think #113 and #136 is a duplicate of this issue

they only say order cancellation will revert.

In buy order, this is not a problem, if a user use a blocklist address or address(0) as a recipient, he lock his own payout token, no loss of fund from protocol.

The impact of not able to cancel or fill order on protocol is the operator are force to waste gas by executing transaction can fail.

Only combining with the partially filled order, the blocklist recipient address becomes a problem, any partially filled payment token in sell order are lost forever

I suggest we can separate #113 and #136 and create a separate medium, in fact, the new medium can be grouped with #57 and #46 and #112

report #46 mentioned the waste gas issue but no partially fill, report 112 mention waste gas and blacklist

report #46 mention

> the order will therefore be stuck inside the contract.

the explanation is vague and not clear, I find it difficult to interpret the sentence as lock fund and it make sense to interpret it as "order cannot be filled".... if the auditor changed the term from "order" to "fund", I will strongly suggest issue 46 is a duplicate of this one, but in this case, a duplication with the newly created medium seems very reaonsable...

# Issue H-2: Bypass the blacklist restriction because the blacklist check is not done when minting or burning 

Source: https://github.com/sherlock-audit/2023-06-dinari-judging/issues/64 

## Found by 
ctf\_sec, dirk\_y, p-tsanev, toshii
## Summary

Bypass the blacklist restriction because the blacklist check is not done when minting or burning

## Vulnerability Detail

In the whitepaper:

> the protocol emphasis that they implement a blacklist feature for enforcing OFAC, AML and other account security requirements
A blacklisted will not able to send or receive tokens

the protocol want to use the whitelist feature to be compliant to not let the blacklisted address send or receive dSahres

For this reason, before token transfer, the protocol check if address from or address to is blacklisted and the blacklisted address can still create buy order or sell order

```solidity
   function _beforeTokenTransfer(address from, address to, uint256) internal virtual override {
        // Restrictions ignored for minting and burning
        // If transferRestrictor is not set, no restrictions are applied

        // @audit
        // why don't you not apply mint and burn in blacklist?
        if (from == address(0) || to == address(0) || address(transferRestrictor) == address(0)) {
            return;
        }

        // Check transfer restrictions
        transferRestrictor.requireNotRestricted(from, to);
    }
```

this is calling

```solidity
function requireNotRestricted(address from, address to) external view virtual {
	// Check if either account is restricted
	if (blacklist[from] || blacklist[to]) {
		revert AccountRestricted();
	}
	// Otherwise, do nothing
}
```

but as we can see, when the dShare token is burned or minted, the blacklist does not apply to address(to)

this allows the blacklisted receiver to bypass the blacklist restriction and still send and receive dShares and cash out their dShares

**because the minting dShares is not blacklisted**

a blacklisted user create a buy order with payment token and set the order receiver to a non-blacklisted address

then later when the [buy order is filled](https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/BuyOrderIssuer.sol#L189), the new dShares is transferred and minted to an not-blacklisted address

**because the burning dShares is not blacklisted**

before the user is blacklisted, a user can frontrun the blacklist transaction to create a sell order and [transfer the dShares into the OrderProcessor](https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L88)

then later when the sell order is filled, the dShares in [burnt from the SellOrderProcess](https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L115) escrow are burnt and the user can receive the payment token

## Impact

Bypass the blacklist restriction because the blacklist check is not done when minting or burning

## Code Snippet

https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L88

https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L115

## Tool used

Manual Review

## Recommendation

implement proper check when burning and minting of the dShares to not let user game the blacklist system, checking if the receiver of the dShares is blacklisted when minting, before filling sell order and burn the dShares, check if the requestor of the sell order is blacklisted

do not let blacklisted address create buy order and sell order




## Discussion

**jaketimothy**

Fixes for this should be considered in combination with #55 as it creates more opportunities for locking up orders.

# Issue M-1: In case of stock split and reverse split, the Dshare token holder will gain or loss his Dshare token value 

Source: https://github.com/sherlock-audit/2023-06-dinari-judging/issues/29 

## Found by 
ast3ros
## Summary

Stock split and reverse split may cause the token accounting to be inaccurate.

## Vulnerability Detail

Stock split and reverse split are very common in the stock market. There are many examples here: https://companiesmarketcap.com/amazon/stock-splits/

For instance, in a 2-for-1 stock split, a shareholder receives an additional share for each share held. However, the DShare token holder still holds only one DShare token after the split. If a DShare token holder owns 100 DShare tokens before the split, he will still own 100 DShare tokens after the split. However, he should own 200 DShare tokens after the split.

Currently, users can buy 1 DShare token at the current market price of the underlying share. https://sbt.dinari.com/tokens

This means that after the stock split, a new Dshare token holder can buy a Dshare at half the price of the previous Dshare token holder. This is unfair to the previous Dshare token holder. In other words, the original Dshare token holder will lose 50% of his Dshare token value after the stock split.

The same logic applies to stock reverse split.

## Impact

The Dshare token holder will gain or loss his Dshare token value after the stock split or reverse split.

## Code Snippet

https://github.com/sherlock-audit/2023-06-dinari/blob/50eb49260ab54e02748c2f6382fd95284d271f06/sbt-contracts/src/BridgedERC20.sol#L13

## Tool used

Manual Review

## Recommendation

The Operator should have a mechanism to mint or burn DShare tokens of holders when the underlying share is split or reverse split.



## Discussion

**jaketimothy**

While this would be catastrophic if unaddressed, our current - admittedly undocumented - approach is to halt and cancel all orders for assets at the end of the day the split is announced. Then deploy a new token with a burn->mint migration contract at the appropriate split ratio, add the new token to the issuers, and re-enable orders with the new token.

This accounts for various scenarios where the original tokens may be locked in other contracts without Dinari having to push airdrop transactions for every split - or maintaining burn privileges on all owners/holders.

**ctf-sec**

The amazon last stock split is 20 years ago. I think this report is out of scope

> The Operator should have a mechanism to mint or burn DShare tokens of holders when the underlying share is split or reverse split.

The operator is a privileged central party that can do that.

**Oot2k**

I still think this issue is valid. 
Stock splits happen fairly often, for example Tesla in 2020 and 2022.
I think this is a valid medium, because the offchain system was not mentioned anywhere and there still can be issues when deploying a token manually.
There wont be direct loss of funds, but protocols that integrate with dinari will have problems (Dex, lending etc.)

**ctf-sec**

While the stock and offline logic is out of scope,

> In 2-for-1 stock split, a shareholder receives an additional share for each share held

In the current implementation, this would require the admin [mint stock for user](https://github.com/sherlock-audit/2023-06-dinari/blob/50eb49260ab54e02748c2f6382fd95284d271f06/sbt-contracts/src/BridgedERC20.sol#L113)

In a stock reverse split,

In the current implementation, admin can't [burn stock for user](https://github.com/sherlock-audit/2023-06-dinari/blob/50eb49260ab54e02748c2f6382fd95284d271f06/sbt-contracts/src/BridgedERC20.sol#L120)

```solidity
  function burn(uint256 value) external virtual onlyRole(BURNER_ROLE) {
        _burn(msg.sender, value);
    }
```

if the implementation is 

```solidity
  function burn(address from, uint256 value) external virtual onlyRole(BURNER_ROLE) {
        _burn(from, value);
    }
```

I would agree this is a low severity and out of scope finding,

but since the admin can't actually burn for user,

this can be valid medium, severity is definitely not high :)

# Issue M-2: Escrow record not cleared on cancellation and order fill 

Source: https://github.com/sherlock-audit/2023-06-dinari-judging/issues/56 

## Found by 
Delvir0, bin2chen, bitsurfer, chainNue, ctf\_sec, dirk\_y, hals
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

# Issue M-3: Blocklisted recipient or address(0) force operator to waste gas when filling the sell order or filling the cancel order 

Source: https://github.com/sherlock-audit/2023-06-dinari-judging/issues/57 

## Found by 
0xMosh, Avci, auditsea, ctf\_sec, shogoki
## Summary

Blocklisted recipient or address(0) force operator to waste gas when filling the order

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

the operator are force to waste gas and the order filling transaction keep reverting because the recipient cannot receive the payment token in Sell Order

https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L171

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

In BuyOrder

User can send a tiny amount of payment and then call cancel order

the operator is expected to help user cancel order, but when he doing so, the gas is wasted because the transaction would keep reverting because the recipient address is blocklisted or address(0)

[here](https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/BuyOrderIssuer.sol#L214)

## Impact

Blocklisted recipient or address(0) force operator to waste gas when filling the sell order or filling the cancel order

## Code Snippet

https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/BuyOrderIssuer.sol#L214

https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L171

## Tool used

Manual Review

## Recommendation

We recommend the protocol validate the recipient address to not let user set a blocklisted address or address(0)

# Issue M-4: The percentage fee can vary during the life of a sell order 

Source: https://github.com/sherlock-audit/2023-06-dinari-judging/issues/58 

## Found by 
ctf\_sec
## Summary
Sell orders can be filled partially. Upon each partial fill, the percentage fee is calculated based on the current fee percentage set in the `OrderFees.sol` contract. Therefore, any change to the percentage fee rate retroactively applies to all active sell orders.

## Vulnerability Detail
Buy orders do not suffer from this issue, as they have the percentage fee taken out upon requesting the order. This ensures that future changes to the percentage fee rate do not affect active orders.

A user can create a sell order while the percentage fee is **1%**. If Dinari changes it so that the percentage fee is now **10%**, then it will apply to the already active order.

## Impact
- Users will be exposed to different percentage fees than expected when requesting the order

## Code Snippet
https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/SellOrderProcessor.sol#L92-L101

## Tool used
Manual Review

## Recommendation
The percentage fee rate should be stamped when the sell order is requested. Future changes to the current percentage fee should not affect existing orders.

# Issue M-5: Cancellation refunds should return tokens to order creator, not recipient 

Source: https://github.com/sherlock-audit/2023-06-dinari-judging/issues/61 

## Found by 
0x007, ArmedGoose, Kodyvim, ctf\_sec, dirk\_y, james\_wu, osmanozdemir1, shtesesamoubiq
## Summary
When an order is cancelled, the refund is sent to `order.recipient` instead of the order creator because it is the order creator (requestor) pay the payment token for buy order or pay the dShares for sell order

As is the standard in many L1/L2 bridges, cancelled deposits should be returned to the order creator instead of the recipient. In Dinari's current implementation, a refund acts as a transfer with a middle-man.

## Vulnerability Detail
Simply, the `_cancelOrderAccounting()` function returns the refund to the `order.recipient`:

```solidity
    function _cancelOrderAccounting(OrderRequest calldata orderRequest, bytes32 orderId, OrderState memory orderState)
        internal
        virtual
        override
    {
        ...

        uint256 refund = orderState.remainingOrder + feeState.remainingPercentageFees;

        ...

        if (refund + feeState.feesEarned == orderRequest.quantityIn) {
            _closeOrder(orderId, orderRequest.paymentToken, 0);
            // Refund full payment
            refund = orderRequest.quantityIn;
        } else {
            // Otherwise close order and transfer fees
            _closeOrder(orderId, orderRequest.paymentToken, feeState.feesEarned);
        }


        // Return escrow
        IERC20(orderRequest.paymentToken).safeTransfer(orderRequest.recipient, refund);
    }
```

Refunds should be returned to the order creator in cases where the input recipient was an incorrect address or simply the user changed their mind prior to the order being filled.

## Impact
- Potential for irreversible loss of funds
- Inability to truly cancel order

## Code Snippet
https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/issuer/BuyOrderIssuer.sol#L193-L215

## Tool used
Manual Review

## Recommendation
Return the funds to the order creator, not the recipient.

# Issue M-6: `orderId` needs to be unique 

Source: https://github.com/sherlock-audit/2023-06-dinari-judging/issues/106 

## Found by 
0xpinky, auditsea, seerether, tnquanghuy0512
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

