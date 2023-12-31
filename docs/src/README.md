## Lyra Hedge Call Demo

### Installation
```bash
$ git clone https://github.com/voith/lyra-hedge-call.git
$ cd lyra-hedge-call
$ git submodule update --init --recursive
$ yarn
$ curl -L https://foundry.paradigm.xyz | bash
$ foundryup
```

### Testing
```bash
$ FOUNDRY_PROFILE="fork" forge test -vvv --fork-url <OPTIMISM_RPC_URL>
```
**Note**: *replace `<OPTIMISM_RPC_URL>` with an actual node url for Optimism mainnet.*

*Some notes on testing*
- Foundry evm is not fully compliant with optimism's EVM. `vm.rollFork` doesn't sync block changes after calling it more than once inside the test function.
   However, `vm.rollFork` works fine inside the `setUp` function.
   Because of this, each test had to be moved to a separate Test contract. This way every test can have its own `setUp` function.
- The fork tests roll to blocks that have already been finalised. Lyra positions that the tests open get overriden by a new owner when the test rolls to a new block.
  For this reason, the tests re-open new positions to mock options that were opened at a previous block.

### Architecture

![architecture](./static/LyraSnxHedgeStrategy-Architecture.jpeg)

*Account Architecture* 
- The account contract is designed to be upgradeable using the beacon proxy pattern. Here's an example of the [beacon proxy pattern](https://gist.github.com/voith/2b4f15ee19cb041a8521b300a176801b).
- Whenever the User makes a call to the `AccountProxy` contract, the `AccountProxy` requests the `AccountFactory` contract for the implementation contract address and then delegates the call further to the implementation contract via `DELEGATECALL`.
- The Account Contract inherits from the `LyraSNXHedgeStrategy` contract that has the core logic for the `buyHedgedCall` function.

*LyraSNXHedgeStrategy Architecture*
- The `LyraSNXHedgeStrategy` contract inherits from `LyraOptionsAdapter` and `SynthetixPerpsAdapter`
- The `LyraOptionsAdapter` contract has the logic for buying a long call option from lyra and to calculate the delta of any given strike.
- The `SynthetixPerpsAdapter` contract has the logic for transferring margin and for submitting orders for buying/selling perps on synthethix perps v2.
- `LyraSNXHedgeStrategy` also contains a function that allows users to re hedge their positions. 
- Orders submitted by `SynthetixPerpsAdapter` are offchain delayed orders. These need to be executed by a keeper/user.

*User Flow*
- A user creates a new account by calling the `newAccount()` function on the `AccountFactory` contract and gets the address of an `AccountProxy` instance.
- The `AccountFactory` also has a `getOwnerAccountAddress` function that returns the `AccountProxy` address for the owner.
- Whenever the User makes a call to the `AccountProxy` contract, the `AccountProxy` requests the `AccountFactory` contract for the implementation contract address and then delegates the call further to the implementation contract via `DELEGATECALL`.
- The user needs to transfer `USDC` and `SUSD` to the `AccountProxy` contract before calling `buyHedgedCall`. These tokens are used to buy options and perps.
- The user then optimistically calls `buyHedgeCall`. If all goes well then an option is bought on lyra and an order is submitted for shorting perps on synthetix perps v2.
- A keeper or the same user must execute the offcahin order submitted by `buyHedgedCall`.
- Over time if the call delta changes, then the user can re hedge their positions by calling the `reHedge()` function on the `AccountProxy` contract.

### Example
Here's an example to show the usage of `buyHedgedCall` using foundry scripts.
Please note that this example is not tested, and it's meant for demonstration purposes only. If you're interested in a working example then take a look at the tests. 

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Script.sol";
import {Account as Account_} from "contracts/Account.sol";
import {AccountFactory} from "contracts/AccountFactory.sol";
import {IERC20} from "openzeppelin-contracts-4.4.1/token/ERC20/IERC20.sol";

contract BuyHedgedCallExample is Script {
    Account userAccount;
    AccountFactory accountFactory;
    IERC20 usdc = IERC20(0x7F5c764cBc14f9669B88837ca1490cCa17c31607);
    IERC20 susd = IERC20(0x8c6f28f2F1A3C87F0f938b96d27520d9751ec8d9);
    uint256 userPrivateKey;
    address user;

    function setUp() public {
        userPrivateKey = vm.envUint("PrivateKey");
        user = vm.addr(userPrivateKey);
        accountFactory = AccountFactory(vm.envAddress("AccountFactoryAddress"));
    }

    function run() public {
        vm.startBroadcast(userPrivateKey);
        // create new Account for user. This will revert if account already exists for the user. 
        address userAccountAddress = accountFactory.newAccount();
        userAccount = Account(userAccountAddress);
        
        // transfer USDC and sUSD to userAccount. This will be using for buying the Option and perps.
        usdc.transfer(userAccountAddress, 1000e6);
        susd.transfer(userAccountAddress, 5000e18);
        
        // buy call on lyra with strikeID 181 and size 2. This will also hedge the delta by shorting perps on snx.
        userAccount.buyHedgedCall(181, 2e18);
        // similarly we can call userAccount.reHedge() if an option was previously bought through `buyHedgedCall`.
    }
}
```
**Note**: The order for shorting perps submitted by `buyHedgedCall` needs to executed externally by a keeper/user.
Here's an [example](https://github.com/voith/lyra-hedge-call/blob/f873497d985505e623005b128f0ef7e378dfeab4/test/ForkTestReHedgeWhenDeltaDecrease.t.sol#L58-L61) that shows how to execute the order.

### Design consideration and Limitations
1. <ins>Beacon Proxy Account model</ins>: The easiest way to implement `buyHedgedCall` would have been to buy the option and perps through a smart contract and then transfer
   the positions to the `msg.sender`. However, `SNXPerpsV2` does not issue any token after buying perps. Also, the perps positon cannot be transferred. 
   Hence, an account was created to hold a users perps and options. Holding the option and the perps in a single account also simplifies the re-hedging logic.
   Also, any new logic that needs to be added can be done by upgrading the implementation. The beacon proxy model was chosen because it allows upgrading the `Account` contract for all users by simply updating the implementation in the `AccountFactory`.  
2. <ins>Holding `SUSD` and `USDC` balance in `AccountProxy`</ins>: The `buyHedgedCall` function assumes that the `AccountProxy` will have sufficient balance to buy options and perps.
   Since this project is limited to demonstrating `buyHedgedCall`, a quoter contract to calculate the exact amount of tokens is considered out of scope.
3. <ins>ReHedging logic is not very efficient</ins>: 
   - SNX Perps V2 maintains only one position for any address(positons are added and subtracted).
   - A user that buys multiple strikes through `buyHedgedCall` will have only one net perps position on SNX.
   - Whenever the call delta changes for any strike then the perps position should be recalculated for all the active strikes.
   - For this reason, we need to calculate the net call delta for all the active strikes. 
     This is not efficient as the code iterates through a loop, and it can run out of gas if there are too many open positions.
   - Also, the contract doesn't maintain a list of active positions as a position could get liquidated. Instead, the `getOwnerPositions` function is used to fetch the current active positions.
   - A simple fix could be to limit the number of positions that a user can open. This will ensure that the reHedge function doesn't run out of gas.
4. <ins>Tokens stuck in the account contract</ins>: It might seem like the option tokens and the perps are stuck forever inside the `AccountProxy` contract. 
    However, since the contracts are upgradeable, a new implementation can be added that has logic for closing the open positions and for transferring the PnL.

### Contracts
- [Account](docs/src/contracts/Account.sol/contract.Account.md)
- [AccountFactory](docs/src/contracts/AccountFactory.sol/contract.AccountFactory.md)
- [AccountProxy](docs/src/contracts/AccountProxy.sol/contract.AccountProxy.md)
- [LyraOptionsAdapter](docs/src/contracts/LyraOptionsAdapter.sol/contract.LyraOptionsAdapter.md)
- [LyraSNXHedgeStrategy](docs/src/contracts/LyraSNXHedgeStrategy.sol/contract.LyraSNXHedgeStrategy.md)
- [SynthetixPerpsAdapter](docs/src/contracts/SynthetixPerpsAdapter.sol/abstract.SynthetixPerpsAdapter.md)

To read the contract documentation through a browser, run: 
```bash
$ forge doc --serve
```
