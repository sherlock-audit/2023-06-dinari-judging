ctf_sec

high

# Bypass the blacklist restriction because the blacklist check is not done when minting or burning

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

