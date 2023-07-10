ctf_sec

high

# TransferRestrictor#restrict can be front-run and user can transfer their dShare out to avoid being blacklisted

## Summary

TransferRestrictor#restrict can be front-run and user can transfer their dShare out

## Vulnerability Detail

In the whitepaper:

> the protocol emphasis that they implement a blacklist feature for enforcing OFAC, AML and other account security requirements
A blacklisted will not able to send or receive tokens

For this reason, before token transfer, the protocol check if address from or address to is blacklisted

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

And the admin owner are supposed to call to [restrict](https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/TransferRestrictor.sol#L37) a user transfer or send dShares

```solidity
    /// @notice Restrict `account` from sending or receiving tokens
    /// @dev Does not check if `account` is restricted
    /// Can only be called by `owner`
    function restrict(address account) external onlyOwner {
        blacklist[account] = true;
        emit Restricted(account);
    }
```

However, a user can montior the mempool and if he detect the restrict transaction,

he can frontrun the restrict and transfer his dShares to another address to avoid beging blocklisted

## Impact

Bypass the blacklist restriction by transferring the dShares out

## Code Snippet

https://github.com/sherlock-audit/2023-06-dinari/blob/4851cb7ebc86a7bc26b8d0d399a7dd7f9520f393/sbt-contracts/src/TransferRestrictor.sol#L37

## Tool used

Manual Review

## Recommendation

We recommend the protocol use private relay service such as flashbot to avoid frontrunning 






