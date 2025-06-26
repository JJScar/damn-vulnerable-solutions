# The Rewarder

**As always, try to solve the challenge yourself before reading on. Review the contracts, experiment, debug, and test your assumptions!**

## Challenge Description:
A contract is distributing rewards of Damn Valuable Tokens (DVT) and WETH.

To claim rewards, users must prove they’re included in the chosen set of beneficiaries. Don’t worry about gas though. The contract has been optimized and allows claiming multiple tokens in the same transaction.

Alice has claimed her rewards already. You can claim yours too! But you’ve realized there’s a critical vulnerability in the contract.

Save as much funds as you can from the distributor. Transfer all recovered assets to the designated recovery account.

The contract here is meant to be an optimized Merkle-based airdrop claim system, allowing multiple rewards to be claimed in a single transaction, even across multiple tokens. But buried within that optimization is a subtle batching bug that allows for **repeated claims of the same batch** under specific conditions — and that’s where our exploit lies.

---

## 🔍 Vulnerability Overview

The vulnerability here is a **flawed claim batching mechanism** due to incorrect handling of bitmaps per token and batch. Specifically:

### 🧨 *Bitmap Clobbering in Batching Logic*

Let’s break it down:

- The contract allows users to claim multiple reward batches for one or more tokens in a single call to `claimRewards(...)`.
- For gas efficiency, it batches together multiple claims **per token** in a loop and only calls `_setClaimed(...)` once per token.
- Claim status is stored using a **bitmap**: each bit in a 256-bit word marks a batch number as claimed.
- However, **it assumes all claims in the batch have the same `wordPosition`**, and only writes that one word.
- If you submit **multiple claims across different bit positions (but same word)** — e.g. batch 0, 1, 2, ... — it works.
- But if you submit **the same claim multiple times** within the batch — it **still passes**, because the bit position is only set once.

That’s the bug. You can repeatedly include the same valid claim, and as long as `_setClaimed()` sees that bitmap word as unclaimed, the logic passes and rewards are transferred — **even for duplicate claims**.

---

## 🧪 Demonstrating the Exploit

We target the player’s valid Merkle proof from the existing distribution (`batchNumber = 0`) and duplicate it **hundreds of times** in one batched call.

Here’s the key idea:
- The `claimRewards()` function doesn’t track individual claim identities — it only tracks the **bitmask per word**.
- If you repeat the same claim enough times in a single batch (all under `batchNumber = 0`), the contract will happily transfer the tokens repeatedly.
- This allows us to drain nearly the entire DVT and WETH balances by spamming the same claim entry again and again.

---

<details>
<summary>💥 Exploit Snippet</summary>

```solidity
Claim ; // hundreds of repeated claims

for (uint256 i = 0; i < claims.length; i++) {
    if (i < 867) {
        claims[i] = Claim({
            batchNumber: 0,
            amount: dvt_amount_to_claim,
            tokenIndex: 0,
            proof: merkle.getProof(dvtLeaves, dvt_player_index)
        });
    } else {
        claims[i] = Claim({
            batchNumber: 0,
            amount: weth_amount_to_claim,
            tokenIndex: 1,
            proof: merkle.getProof(wethLeaves, weth_player_index)
        });
    }
}
distributor.claimRewards(claims, tokensToClaim);
```
</details>

Afterward, we simply transfer all stolen funds to the recovery account:

```solidity
dvt.transfer(recovery, dvt.balanceOf(player));
weth.transfer(recovery, weth.balanceOf(player));
```

And that’s it — the exploit passes the `_isSolved()` check.

---

## 🔒 Recommendations
✅ Fix the batching logic to handle all word positions.

A correct implementation would:
- Track and store which `wordPositions` were affected during the loop
- Call `_setClaimed()` once per `(token, wordPosition)` pair with the relevant bitsSet
- Or simply call `_setClaimed()` on each iteration, but that sacrifices some gas efficiency

✅ Add replay protection per batch per user per token

Introduce a mapping like:
```solidity
mapping(address => mapping(IERC20 => mapping(uint256 => bool))) claimed;
```
…or similar, to explicitly mark individual (token, batch) claims.

✅ Consider using an external audited MerkleClaim module (like OpenZeppelin’s)

This prevents having to write and manage Merkle + bitmap logic manually, which is error-prone and nuanced.

---

## Final Thoughts
This was a classic example of an optimization gone wrong. By trying to minimize gas usage through bitmaps and batching, the developer introduced a subtle and powerful vulnerability — one that allows a valid claim to be copied and reused within a single batch call.

Well done if you discovered this — and if not, now you’ve got another tool in your mental model of how state-handling bugs can be abused in Web3 smart contracts.
