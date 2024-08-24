# First Flight #21: KittyFi - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01. [H-3] `meowintKittyCoin` or `purrgeBadPawsition` will always cause issue/fail for tokens without 18 decimals](#H-01)
    - [H-02. Users Can't Withdraw When All Collateral Is Supplied to Aave](#H-02)
    - [H-03. Incorrect Calculation of User Collateral Value in Euros on `Kitty.sol::getUserVaultMeowllateralInEuros`](#H-03)
    - [H-04. `KittyPool::purrgeBadPawsion` Should Not Call `KittyVault::executeWhiskdrawal` But Directly Transfer Collaterals](#H-04)
    - [H-05. The liquidate function: purrgeBadPawsition(KittyPool.sol) should take vaultCollateral  amount instead of vaultCollateral Value During the calculation. ](#H-05)
    - [H-06. [H-1] Wrong calculation in `getTotalMeowllateralInAave()` leads to wrong user shares estimation.](#H-06)
    - [H-07. Instead of earning 5 percent bonus, the liquidator gets his collateral stuck in the contract](#H-07)
- ## Medium Risk Findings
    - [M-01. KittyVault NOT checking For Stale Prices](#M-01)
    - [M-02. [M-1] - `getTotalMeowllateralInAave()` using two different Oracles can cause price mismatch!](#M-02)
    - [M-03. There is no incentive to liquidate small positions](#M-03)
- ## Low Risk Findings
    - [L-01. In `KittyPool` users collateral could potentially lock, leading to financial loss during the withdrew all the Collateral value form vault and tries to mint `kittyCoin` without deposit any Collateral.](#L-01)
    - [L-02. Collateral deposited won't earn additional yield when hits supply limitation in Aave pool](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #21

### Dates: Aug 1st, 2024 - Aug 8th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-kitty-fi)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 7
   - Medium: 3
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. [H-3] `meowintKittyCoin` or `purrgeBadPawsition` will always cause issue/fail for tokens without 18 decimals

_Submitted by [Owenzo](/profile/cls0la11f0000127hfng6wpzl), [nilay27](/profile/cltshhl2x0000cqre7q2bjkf4), [ayamercy](/team/clz09i1y50001hibkcpjzt4np). Selected submission by: [nilay27](/profile/cltshhl2x0000cqre7q2bjkf4)._      
            


### Description:

In case of tokens with different number of decimals than 18 (the decimals for KittyCoin), both `meowintKittyCoin` and `purrgeBadPawsition` will fail as they are dependent of `_hasEnoughMeowllateral`.

The issue is htat `_hasEnoughMeowllateral` checks fetches  `getUserMeowllateralInEuros(user)` which will return the value in terms of the number of decimals of the token.

Based on this value, the contract checks the `collateralRequiredInEuros` by upscaliing it with required collateral percent.

Lastly it compares `totalCollateralInEuros >= collateralRequiredInEuros;` which will fail in case of tokens with different number of decimals.

### Impact:

In case of tokens like USDC, WETH which have 6 and 8 decimals respectively, this will always fail thus causing users fund to be stuck in the contract.

Since users deposit collateral first using `depawsitMeowllateral` and then only they are allowed to mint tokens using `meowintKittyCoin()` which is defined as:

```solidity
function meowintKittyCoin(uint256 _ameownt) external {
    kittyCoinMeownted[msg.sender] += _ameownt;
    i_kittyCoin.mint(msg.sender, _ameownt);
    require(_hasEnoughMeowllateral(msg.sender), KittyPool__NotEnoughMeowllateralPurrrr());
}
```

This will always revert and the users deposited funds will be stuck!

Morever, this can lead to weird scenarios where despite having enough collateral, users can end up in having their positions purged.

```solidity
function purrgeBadPawsition(address _user) external returns (uint256 _totalAmountReceived) {
    // this condition might also turn to be true in case of enough collaterals in case
    // of usdc or weth
    require(!(_hasEnoughMeowllateral(_user)), KittyPool__UserIsPurrfect());
```

### Proof of Concept:

Add the following code in `KittyFiTest.t.sol`

```solidity
modifier userDepositsBTCCollateral(){
    deal(config.wbtc, user, 10e8); // wbtc has 8 decimals
    uint256 wbtcToDeposit = 5e8;

    // create btc vault:
    vm.prank(meowntainer);
    kittyPool.meownufactureKittyVault(config.wbtc, config.btcUsdPriceFeed);

    wbtcVault = KittyVault(kittyPool.getTokenToVault(config.wbtc));

    // user deposits 5 btc to vault as collateral
    vm.startPrank(user);
    IERC20(config.wbtc).approve(address(wbtcVault), wbtcToDeposit);
    kittyPool.depawsitMeowllateral(config.wbtc, wbtcToDeposit);
    vm.stopPrank();
    _;
}

function test_failureInMintingWithBTC() public userDepositsBTCCollateral{
    uint256 btcDeposited = 5e8;

    // check for BTC price in USD
    AggregatorV3Interface btcUsdPriceFeed = AggregatorV3Interface(config.btcUsdPriceFeed);
    (, int256 btcUSDPrice, , , ) = btcUsdPriceFeed.latestRoundData();
    console.log("btc price in oracle", uint256(btcUSDPrice));

    // fetch EURO in terms of USD
    AggregatorV3Interface euroPriceFeed = AggregatorV3Interface(config.euroPriceFeed);
    (, int256 euroPrice, , , ) = euroPriceFeed.latestRoundData();
    console.log("euroPrice", uint256(euroPrice));

    // expected value of KittyCoin to be minted
    uint256 expectedKittyCoinToBeMinted = btcDeposited.mulDiv(uint256(btcUSDPrice), uint256(euroPrice));
    console.log("expectedKittyCoinToBeMinted in EUR with 8 decimals", expectedKittyCoinToBeMinted);

    uint256 mintingArgument = expectedKittyCoinToBeMinted * 1e10; // as kittycoin has 18 decimals

    vm.startPrank(user);
    kittyPool.meowintKittyCoin(mintingArgument);
}

function test_failureInMintingWithBTC() public userDepositsBTCCollateral{
    uint256 btcDeposited = 5e8;

    // check for BTC price in USD
    AggregatorV3Interface btcUsdPriceFeed = AggregatorV3Interface(config.btcUsdPriceFeed);
    (, int256 btcUSDPrice, , , ) = btcUsdPriceFeed.latestRoundData();
    console.log("btc price in oracle", uint256(btcUSDPrice));

    // fetch EURO in terms of USD
    AggregatorV3Interface euroPriceFeed = AggregatorV3Interface(config.euroPriceFeed);
    (, int256 euroPrice, , , ) = euroPriceFeed.latestRoundData();
    console.log("euroPrice", uint256(euroPrice));

    // expected value of KittyCoin to be minted
    uint256 expectedKittyCoinToBeMinted = btcDeposited.mulDiv(uint256(btcUSDPrice), uint256(euroPrice));
    console.log("expectedKittyCoinToBeMinted in EUR with 8 decimals", expectedKittyCoinToBeMinted);

    uint256 mintingArgument = expectedKittyCoinToBeMinted * 1e10; // as kittycoin has 18 decimals

    vm.startPrank(user);
    vm.expectRevert();
    kittyPool.meowintKittyCoin(mintingArgument);
}
```

**Console Log**:

```Solidity
btc price in oracle 5739293200000
euroPrice 109264000
expectedKittyCoinToBeMinted in EUR with 8 decimals 26263422536242
```

### Recommended Mitigation:

1. Ensure that inside `_hasEnoughMeowllateral` the call to `getUserMeowllateralInEuros(user)` always returns fixed decimals and  then handle the number accordingly

## <a id='H-02'></a>H-02. Users Can't Withdraw When All Collateral Is Supplied to Aave

_Submitted by [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [evmninja](/profile/clxxng1pm0000mnz6avhempwf), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [maptool](/profile/clzgusneu0002ocq06zsl7fbf), [14xSachet](/profile/clz15oltz0006q1gtgf0ywir8). Selected submission by: [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31)._      
            


# \[H-01] Users Can't Withdraw When All Collateral Is Supplied to Aave

## Summary

Users should always be able to withdraw their funds without interruption, even in the face of maintainer actions. However, there's a critical issue with `KittyVault::executeWhiskdrawal` when the withdrawal request exceeds the available amount in the vault (`totalMeowllateralInVault`). Consequently, this function reverts due to underflow or insufficient balance for the transfer.

## Vulnerability Details

The `KittyPool::whiskdrawMeowllateral` function makes a call to `KittyVault::executeWhiskdrawal`, but when there's insufficient collateral available, this function reverts. This issue arises because the withdrawal amount requested is higher than the actual amount available in the vault.

```javascript
    function executeWhiskdrawal(
        address _user,
        uint256 _cattyNipToWithdraw
    ) external onlyPool {
        uint256 _ameownt = _cattyNipToWithdraw.mulDiv(
            getTotalMeowllateral(),
            totalCattyNip
        );
        userToCattyNip[_user] -= _cattyNipToWithdraw;
        totalCattyNip -= _cattyNipToWithdraw;
@>      totalMeowllateralInVault -= _ameownt;
@>      IERC20(i_token).safeTransfer(_user, _ameownt);
    }
```

## Impact

This flaw prevents users from withdrawing their funds when all collateral or more than the requested withdrawal amount is supplied to the Aave pool, disrupting normal operations and potentially causing financial loss.

## Proof of Concept

The following test case illustrates the issue:

```javascript
    function test_CantWithdrawWhenSupplied() public userDepositsCollateral {
        uint256 totalDepositedInVault = 5 ether;

        // meowntainer transfers collateral in eth vault to Aave to earn interest
        uint256 toSupply = 5 ether;

        vm.prank(meowntainer);
        wethVault.purrrCollateralToAave(toSupply);
        assertEq(
            wethVault.totalMeowllateralInVault(),
            totalDepositedInVault - toSupply
        );

        uint256 totalCollateralBase = wethVault.getTotalMeowllateralInAave();

        assert(totalCollateralBase > 0);

        vm.startPrank(user);
        uint256 toWithdraw = wethVault.userToCattyNip(user);
        vm.expectRevert(); //underflow because totalMeowllateralInVault is 0
        kittyPool.whiskdrawMeowllateral(weth, toWithdraw);
    }
```

## Tools Used

Manual Review

## Recommendations

* Implement logic in `executeWhiskdrawal` to handle cases where the requested withdrawal amount exceeds `totalMeowllateralInVault`. This can involve transferring additional collateral from the Aave pool if necessary.
* Grant the pool contract access to the `purrrCollateralFromAave` function alongside the maintainer, allowing for automated management of collateral supply and withdrawal.

```diff
    function executeWhiskdrawal(
        address _user,
        uint256 _cattyNipToWithdraw
    ) external onlyPool {
        uint256 _ameownt = _cattyNipToWithdraw.mulDiv(
            getTotalMeowllateral(),
            totalCattyNip
        );
+       if( _ameownt > totalMeowllateralInVault){
+           IKittyVault(tokenToVault[_token]).purrrCollateralFromAave(_ameownt - totalMeowllateralInVault);
+       }        
        userToCattyNip[_user] -= _cattyNipToWithdraw;
        totalCattyNip -= _cattyNipToWithdraw;
        totalMeowllateralInVault -= _ameownt;
        IERC20(i_token).safeTransfer(_user, _ameownt);
    }
```

## <a id='H-03'></a>H-03. Incorrect Calculation of User Collateral Value in Euros on `Kitty.sol::getUserVaultMeowllateralInEuros`

_Submitted by [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [esoetheric](/profile/clzkghdz90000t5c61llr10a0), [nilay27](/profile/cltshhl2x0000cqre7q2bjkf4), [0xsagetony](/profile/clnxw408y000gl6086iqksw4c). Selected submission by: [0xsagetony](/profile/clnxw408y000gl6086iqksw4c)._      
            


# Summary

The function `getUserVaultMeowllateralInEuros` is critical for determining the value of user collateral. However, there is an issue with the current calculation. Given the conversion rate of 1 EUR = 1.09 USD, the value in euros should be less than in USD. The current formula used:

```solidity
collateralAns.mulDiv(uint256(euroPriceFeedAns) * EXTRA_DECIMALS, PRECISION);
```

results in a value that is consistently higher than the original collateral amount (`collateralAns`). This discrepancy occurs because the formula incorrectly scales the conversion rate.

### Vulnerability Details

The calculation does not accurately reflect the conversion from USD to EUR. Given the current rate of 1 EUR = 1.09 USD, the value in euros should be lower than in USD. However, the formula provided leads to inflated results:

```solidity
collateralAns.mulDiv(uint256(euroPriceFeedAns) * EXTRA_DECIMALS, PRECISION);
```

This error results in the `getUserVaultMeowllateralInEuros` function returning a value that exceeds the USD equivalent of the collateral, contradicting the expected behaviour where EUR should be less than USD.

### Proof of Concept (POC)

1. Deposit 5 ether into the vault.
2. Mint 5 ether from the pool.
3. Observe that the output of `getUserVaultMeowllateralInEuros(user)` is greater than the USD value of 5 ether. Since 1 EUR = 1.09 USD, the EUR value should be lower than the USD value, but this is not reflected correctly in the current implementation.

### Impact&#x20;

The incorrect calculation of collateral value in euros has significant implications for the `KittyVault.sol` smart contract, particularly affecting the following areas:

1. **Collateral Valuation Accuracy:**

   The current formula inflates the euro value of collateral, resulting in a value that is higher than expected given the USD to EUR conversion rate. This misvaluation can lead to inaccurate assessments of collateral worth, potentially affecting liquidity and risk management within the system.
2. **User Funds and Liquidation Risks:**

   Users’ collateral might be undervalued or misrepresented, leading to potential issues with collateral requirements and liquidation thresholds. Users could face unintended liquidation risks or incorrect collateral requirements, which can erode trust in the system and affect user experience.

### Tool Used

Manual&#x20;

### Recommendation

To correct the calculation and ensure the proper conversion of collateral value from USD to EUR, update the formula as follows:

```solidity
collateralAns.mulDiv(
    PRECISION,
    uint256(euroPriceFeedAns) * EXTRA_DECIMALS
);
```

By adjusting the formula in this way, the calculation will accurately reflect the value of the collateral in euros, taking into account the correct conversion rate

## <a id='H-04'></a>H-04. `KittyPool::purrgeBadPawsion` Should Not Call `KittyVault::executeWhiskdrawal` But Directly Transfer Collaterals

_Submitted by [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [esoetheric](/profile/clzkghdz90000t5c61llr10a0), [bytesflow007](/profile/clywm16360006sr4igeys47em). Selected submission by: [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31)._      
            


# \[H-03] `KittyPool::purrgeBadPawsion` Should Not Call `KittyVault::executeWhiskdrawal` But Directly Transfer Collaterals

## Summary

The `purrgeBadPawsion` function in the `KittyPool` contract is designed to incentivize liquidators by calling the `executeWhiskdrawal` function on each vault involved in the liquidation process. However, this approach inadvertently reduces the liquidator's own collateral instead of incentivizing them, as `executeWhiskdrawal` operates on the assumption that the liquidator is withdrawing their own funds. This misalignment leads to two primary issues:

1. In the best-case scenario, the function will always revert due to the liquidator not having deposited any collateral with the protocol.
2. In the worst-case scenario, the liquidator loses money instead of gaining a reward, as they pay the kitty coins for liquidating and receive their own deposited collateral instead of a reward.

## Vulnerability Details

The core issue arises from the `purrgeBadPawsion` function calling `_vault.executeWhiskdrawal(msg.sender, toDistribute + extraReward);` with `msg.sender` as the user input, which incorrectly treats the liquidator as withdrawing their own collateral, thereby reducing their share instead of increasing it as a reward.

```javascript
    function purrgeBadPawsition(
        address _user
    ) external returns (uint256 _totalAmountReceived) {
        require(!(_hasEnoughMeowllateral(_user)), KittyPool__UserIsPurrfect());
        uint256 totalDebt = kittyCoinMeownted[_user];

        kittyCoinMeownted[_user] = 0;
@>      i_kittyCoin.burn(msg.sender, totalDebt);
        .
        .
        .
        for (uint256 i; i < vaults_length; ) {
            IKittyVault _vault = IKittyVault(vaults[i]);
            uint256 vaultCollateral = _vault.getUserVaultMeowllateralInEuros(
                _user
            );
            uint256 toDistribute = vaultCollateral.mulDiv(
                redeemPercent,
                PRECISION
            );
            uint256 extraCollateral = vaultCollateral - toDistribute;

            uint256 extraReward = toDistribute.mulDiv(
                REWARD_PERCENT,
                PRECISION
            );
            extraReward = Math.min(extraReward, extraCollateral);
            _totalAmountReceived += (toDistribute + extraReward);

@>          _vault.executeWhiskdrawal(msg.sender, toDistribute + extraReward);
            unchecked {
                ++i;
            }
        }
    }
```

The `executeWhiskdrawal` function is designed to allow users to withdraw their collateral, but when called with the intention of rewarding a liquidator, it fails to serve its intended purpose.

```javascript
    function executeWhiskdrawal(
        address _user,
        uint256 _cattyNipToWithdraw
    ) external onlyPool {
        //@audit-written high user cannot withdrawl if all collateral is supplied to aave atm
        uint256 _ameownt = _cattyNipToWithdraw.mulDiv(
            getTotalMeowllateral(),
            totalCattyNip
        );
@>      userToCattyNip[_user] -= _cattyNipToWithdraw;
@>      totalCattyNip -= _cattyNipToWithdraw;
@>      totalMeowllateralInVault -= _ameownt;

        IERC20(i_token).safeTransfer(_user, _ameownt);
    }
```

## Impact

This flaw directly impacts the financial outcome for liquidators, who are supposed to be rewarded for their actions but end up losing money instead.

## Tools Used

Manual Review

## Recommendations

To address this issue, a separate function specifically designed for liquidation rewards should be created. This new function would correctly increase the liquidator's share in the vaults, ensuring that they are genuinely incentivized for their role in the liquidation process.

## <a id='H-05'></a>H-05. The liquidate function: purrgeBadPawsition(KittyPool.sol) should take vaultCollateral  amount instead of vaultCollateral Value During the calculation. 

_Submitted by [bytesflow007](/profile/clywm16360006sr4igeys47em), [14xSachet](/profile/clz15oltz0006q1gtgf0ywir8). Selected submission by: [bytesflow007](/profile/clywm16360006sr4igeys47em)._      
            


## Summary

purrgeBadPawsition(KittyPool.sol) mistakenly use VaultMeowllateralInEuros instead of VaultMeowllateral amount, which leads to the wrong vaultCollateral for the following steps, then can't execuse purrgeBadPawsition successfully.

## Vulnerability Details

The liquidation logic firstly is to calculate the redeemPercent of the user's vaultCollateral based on the current collateral percent(<169)

Then calcuating how much collateral token to the liquidator, should based on the liquidated user's collateral amount, not the collateral euro value.

If based on the collateral euro value, the vaultCollateral value will greater than the actual value, Then calling the  executeWhiskdrawal function will beyond liquidator's cattyNip(share).

> ```Solidity
> /**
>      * @notice Liquidates the bad debt position of the user
>      * @param _user address of the user
>      */
>     function purrgeBadPawsition(address _user) external returns (uint256 _totalAmountReceived) {
>         require(!(_hasEnoughMeowllateral(_user)), KittyPool__UserIsPurrfect());
>         uint256 totalDebt = kittyCoinMeownted[_user];
>
>         kittyCoinMeownted[_user] = 0;
>         i_kittyCoin.burn(msg.sender, totalDebt);
>
>         uint256 userMeowllateralInEuros = getUserMeowllateralInEuros(_user);
>
>         uint256 redeemPercent;
>
>         if (totalDebt >= userMeowllateralInEuros) {
>             redeemPercent = PRECISION;
>         }
>         else {
>             redeemPercent = totalDebt.mulDiv(PRECISION, userMeowllateralInEuros);
>         }
>
>         uint256 vaults_length = vaults.length;
>
>         for (uint256 i; i < vaults_length; ) {
>             IKittyVault _vault = IKittyVault(vaults[i]);
>             // There should use _vault.getUserMeowllateral(), becasue for the following executeWhiskdrawal
>             // function, which take "The amount of shares corresponding to collateral to withdraw", not the actual Euro Value
>             uint256 vaultCollateral = _vault.getUserVaultMeowllateralInEuros(_user);
>             
>             uint256 toDistribute = vaultCollateral.mulDiv(redeemPercent, PRECISION);
>             uint256 extraCollateral = vaultCollateral - toDistribute;
>
>             uint256 extraReward = toDistribute.mulDiv(REWARD_PERCENT, PRECISION);
>             extraReward = Math.min(extraReward, extraCollateral);
>             _totalAmountReceived += (toDistribute + extraReward);
>
>             _vault.executeWhiskdrawal(msg.sender, toDistribute + extraReward);
>
>             unchecked {
>                 ++i;
>             }
>         }
>     }
> ```



## Impact

No one can execute purrgeBadPawsition successfully

The protocol can't guarantee the kittyCoin's price, such as when user's collateral value under the requirements. No ways to matain the totoal collateral value.

POC Test

```Solidity
function test_poc_purrgeBadPawsition() public {
        // For test, should create new kittyPool, and make the mockWethToken as the collotral
        kittyPool = new KittyPool(
            meowntainer,
            config.euroPriceFeed,
            config.aavePool
        );
        kittyCoin = KittyCoin(kittyPool.getKittyCoin());

        // mock wethVault
        ERC20Mock mockWethToken = new ERC20Mock();
        MockV3Aggregator wethPriceFeed = new MockV3Aggregator(8, 20000e8);

        // take mockWethToken as collotral
        vm.prank(meowntainer);
        kittyPool.meownufactureKittyVault(
            address(mockWethToken),
            address(wethPriceFeed)
        );

        KittyVault wethMockVault = KittyVault(
            kittyPool.getTokenToVault(address(mockWethToken))
        );

        // user1 depawsitMeowllateral 0.1 ether， and mint almostMaxKittyCoins
        address user1 = makeAddr("user1");
        AMOUNT = 1 ether;
        mockWethToken.mint(user1, AMOUNT);
        uint256 toDeposit = 1 ether;

        vm.startPrank(user1);
        mockWethToken.approve(address(wethMockVault), toDeposit);
        kittyPool.depawsitMeowllateral(address(mockWethToken), toDeposit);
        uint256 amountToMint = getAlmostMaxKittyTokenForMockwethVault(
            wethMockVault,
            user1
        );
        kittyPool.meowintKittyCoin(amountToMint);
        vm.stopPrank();
        // console.log("user minted kittyCoins", amountToMint);

        // liquidator get the same kittyCoins by depoisted the same colloateral as user1
        address liquidator = makeAddr("liquidator");
        mockWethToken.mint(liquidator, AMOUNT);
        vm.startPrank(liquidator);
        mockWethToken.approve(address(wethMockVault), toDeposit);
        kittyPool.depawsitMeowllateral(address(mockWethToken), toDeposit);
        kittyPool.meowintKittyCoin(amountToMint);
        vm.stopPrank();

        // when weth price = 2000 USD,  show user1 collotrall and debt(kittyCoins)
        console.log("When ETH price 2000");
        console.log("user1 collateral info........................start");
        uint256 user1VaultMeowllateralInEuros = wethMockVault
            .getUserVaultMeowllateralInEuros(user1);
        console.log("collllaterl Vaule in EUR", user1VaultMeowllateralInEuros);
        uint256 user1lDebtForKittyCoins = kittyCoin.balanceOf(user1);
        console.log("kittyCoins", user1lDebtForKittyCoins);
        console.log(
            "user1 collateral_percent",
            (user1VaultMeowllateralInEuros * 100) / user1lDebtForKittyCoins
        );
        console.log("user1 collateral info........................end");

        // make the price down, 2000=>1000
        wethPriceFeed.updateAnswer(12000e8);
        // wethPriceFeed.updateAnswer(10000e8);// this situation also can't run
        console.log("Change ETH price 2000=>1200");
        console.log(
            "Show user1's current collateral percent...................start"
        );
        user1VaultMeowllateralInEuros = wethMockVault
            .getUserVaultMeowllateralInEuros(user1);
        console.log("collllaterl Vaule in EUR", user1VaultMeowllateralInEuros);
        user1lDebtForKittyCoins = kittyCoin.balanceOf(user1);
        console.log("kittyCoins", user1lDebtForKittyCoins);
        console.log(
            "user1 collateral_percent",
            (user1VaultMeowllateralInEuros * 100) / user1lDebtForKittyCoins
        );
        console.log(
            "Show user1's current collateral percent...................end"
        );

        // show liquidator MeowllateralInfo
        console.log(
            "liquidator weth balance",
            mockWethToken.balanceOf(liquidator)
        );
        console.log(
            "liquidator kittyCoins balance",
            kittyCoin.balanceOf(liquidator)
        );

        // As user1's collateral percent < 169, liquidator can liquidate user1's collateral
        vm.startPrank(liquidator);
        kittyPool.purrgeBadPawsition(user1);
        vm.stopPrank();
        console.log("after liquuidation");
        console.log(
            "liquidator weth balance",
            mockWethToken.balanceOf(liquidator)
        );
        console.log(
            "liquidator kittyCoins balance",
            kittyCoin.balanceOf(liquidator)
        );
    }

    function getAlmostMaxKittyTokenForMockwethVault(
        KittyVault wethMockVault,
        address user
    ) internal view returns (uint256) {
        return
            (wethMockVault.getUserVaultMeowllateralInEuros(user) * 100) /
            169 -
            1e18;
    }
```

## Tools Used

Manual

## Recommendations

1. IKittyVault add getUserMeowllateral intereface.
2. Apply `_vault.getUserMeowllateral(_user); `instead of \_vault.getUserVaultMeowllateralInEuros(\_user).

   &#x20;

```Solidity
interface IKittyVault {
    function getUserVaultMeowllateralInEuros(
        address _user
    ) external view returns (uint256);
    function executeDepawsit(address _user, uint256 _ameownt) external;
    function executeWhiskdrawal(address _user, uint256 _ameownt) external;
    // add this getUserMeowllateral.
    function getUserMeowllateral(address _user) external view returns (uint256);
}




function purrgeBadPawsition(address _user) external returns (uint256 _totalAmountReceived) {
        require(!(_hasEnoughMeowllateral(_user)), KittyPool__UserIsPurrfect());
        uint256 totalDebt = kittyCoinMeownted[_user];

        kittyCoinMeownted[_user] = 0;
        i_kittyCoin.burn(msg.sender, totalDebt);

        uint256 userMeowllateralInEuros = getUserMeowllateralInEuros(_user);

        uint256 redeemPercent;

        if (totalDebt >= userMeowllateralInEuros) {
            redeemPercent = PRECISION;
        }
        else {
            redeemPercent = totalDebt.mulDiv(PRECISION, userMeowllateralInEuros);
        }

        uint256 vaults_length = vaults.length;

        for (uint256 i; i < vaults_length; ) {
            IKittyVault _vault = IKittyVault(vaults[i]);
            // Apply  getUserMeowllateral instead of getUserVaultMeowllateralInEuros
            //uint256 vaultCollateral = _vault.getUserVaultMeowllateralInEuros(_user);
            uint256 vaultCollateral = _vault.getUserMeowllateral(_user);
          
            uint256 toDistribute = vaultCollateral.mulDiv(redeemPercent, PRECISION);
            uint256 extraCollateral = vaultCollateral - toDistribute;

            uint256 extraReward = toDistribute.mulDiv(REWARD_PERCENT, PRECISION);
            extraReward = Math.min(extraReward, extraCollateral);
            _totalAmountReceived += (toDistribute + extraReward);

            _vault.executeWhiskdrawal(msg.sender, toDistribute + extraReward);

            unchecked {
                ++i;
            }
        }
    }
```


## <a id='H-06'></a>H-06. [H-1] Wrong calculation in `getTotalMeowllateralInAave()` leads to wrong user shares estimation.

_Submitted by [soupy](/profile/clxtrdlee0002mh58dhdplsq9), [Brene](/profile/clxrieq96000c4boavunj2s9e), [nilay27](/profile/cltshhl2x0000cqre7q2bjkf4). Selected submission by: [nilay27](/profile/cltshhl2x0000cqre7q2bjkf4)._      
            


### Description:

The `getTotalMeowllateralInAave` will always return values in 8 decimals, which is wrong.

As `getTotalMeowllateral()` is defined as:

```solidity
   function getTotalMeowllateral() public view returns (uint256) {
        // totalMeowllateralInVault -> token decimals, getTotalMeowllateralInAave - > 8 decimals
        return totalMeowllateralInVault + getTotalMeowllateralInAave();
    }
```

As `totalMeowllateralInVault` has token decimals where as  `getTotalMeowllateralInAave` has 8 decimals, this will lead to incorrect values in case of tokens which do not have 8 decimals.

* `totalCollateralBase` returned by `i_aavePool.getUserAccountData()` will always be in BASE\_CURRENCY (i.e USD) with 8 decimals.
* `collateralToUsdPrice` returned via `i_priceFeed.latestRoundData()` also has 8 decimals.
* `PRECISION` has 18 decimals, [ref](https://github.com/Cyfrin/2024-08-kitty-fi/blob/main/src/KittyVault.sol#L33)
* `EXTRA_DECIMALS` has 10 decimal, [ref](https://github.com/Cyfrin/2024-08-kitty-fi/blob/main/src/KittyVault.sol#L32)

Thus the returned value via `totalCollateralBase.mulDiv(PRECISION, uint256(collateralToUsdPrice) * EXTRA_DECIMALS)` will be:
(`totalCollateralBase` \* `PRECISION`)/(`collateralToUsdPrice` \* `EXTRA_DECIMALS`)

i.e in terms of decimals:
`(8 decimals * 18 decimals)/(8 decimals * 10 decimals)` \~> **8 decimals**

### Impact:

Completely wrong value of `getTotalMeowllateral()` thus `getUserMeowllateral()` will also return wrong values as:

```solidity
function getUserMeowllateral(address _user) public view returns (uint256) {
    uint256 totalMeowllateralOfVault = getTotalMeowllateral(); // 
    return userToCattyNip[_user].mulDiv(totalMeowllateralOfVault, totalCattyNip);
}
```

The user shares are always wrong and the amount of KittyCoins that can be minted or burnt will be wrong!

**Proof of Concept:**

```solidity
function getTotalMeowllateralInAave() public view returns (uint256) {
        (uint256 totalCollateralBase, , , , , ) = i_aavePool.getUserAccountData(address(this));
        (, int256 collateralToUsdPrice, , , ) = i_priceFeed.latestRoundData();

        //@audit High -> WRONG CALCULATION
        /**
         * will fail for any token which does not have 8 decimals.
         * 
         * totalCollateralBase -> value of collateral in USD with 8 Decimals (amt * Xe8)
         * PRECISION -> 1e18
         * collateralToUsdPrice -> value of collateral in USD with 8 Decimals (Xe8)
         * EXTRA_DECIMALS -> 1e10
         * Hence,
         * returned value -> (amt * Xe8 * 1e18)/(Xe8 * 1e10) -> (amt * 1e18)/(amt * 1e10)
         * Returned value will only have 8 decimal base
         */ 
        return totalCollateralBase.mulDiv(PRECISION, uint256(collateralToUsdPrice) * EXTRA_DECIMALS);
    }
```

**TEST**
Add the following test to `KittyFiTest.t.sol`

```solidity
function test_IncorrectUserMeowllateral() public userDepositsCollateral{
        uint256 totalDepositedInVault = 5 ether;
        uint256 expectedTotalMeowllateral = totalDepositedInVault; // the amount deposited into vault by user

        // move some value of users meowllateral to Aave for getTotalMeowllateralInAave
        uint256 toSupply = 1 ether;
        vm.prank(meowntainer);
        wethVault.purrrCollateralToAave(toSupply);
        assertEq(wethVault.totalMeowllateralInVault(), totalDepositedInVault - toSupply);

        uint256 totalMeowlateralInVault = wethVault.totalMeowllateralInVault();
        uint256 totalMeowlateralInAave = wethVault.getTotalMeowllateralInAave();
        uint256 actualTotalMeawllateral = wethVault.getTotalMeowllateral();
        console.log("totalMeowlateralInVault", totalMeowlateralInVault);
        console.log("totalMeowlateralInAave", totalMeowlateralInAave);
        console.log("actualTotalMeawllateral", actualTotalMeawllateral);
        assertEq(actualTotalMeawllateral, (totalMeowlateralInVault + totalMeowlateralInAave));
        assertNotEq(actualTotalMeawllateral, expectedTotalMeowllateral);
    }
```

run the test via command:

```Solidity
forge test --via-ir --fork-url https://mainnet.infura.io/v3/<your_token> -vvv --mt test_IncorrectUserMeowllateral 
```

**Console Log**

```Solidity
  totalMeowlateralInVault 4000000000000000000
  totalMeowlateralInAave 100000000
  actualTotalMeawllateral 4000000000100000000 // this is 4e18 + 1e8 rather than 5e18
```

### Recommended Mitigation:

Change `getTotalMeowllateral` to use token decimals instead

```solidity
function getTotalMeowllateralInAave() public view returns (uint256) {
    (uint256 totalCollateralBase, , , , , ) = i_aavePool.getUserAccountData(address(this));

    (, int256 collateralToUsdPrice, , , ) = i_priceFeed.latestRoundData();
    - return totalCollateralBase.mulDiv(PRECISION, uint256(collateralToUsdPrice) * EXTRA_DECIMALS);
    + uint256 token_decimals = ERC20(i_token).decimals()
    + return totalCollateralBase.mulDiv(10 ** token_decimals, uint256(collateralToUsdPrice));
}
```

## <a id='H-07'></a>H-07. Instead of earning 5 percent bonus, the liquidator gets his collateral stuck in the contract

_Submitted by [14xSachet](/profile/clz15oltz0006q1gtgf0ywir8)._      
            


## Summary

In the current liquidation process, a liquidator needs to mint the required amount of KittyCoins in order to  burn on behalf of a user. However, to do this, the liquidator must deposit 169 percent more of the value of KittyCoins they intend to mint as collateral. After liquidation, the liquidator receives a 5% bonus on top of the value of KittyCoins burned, but there is an issue with the rest of the collateral. The liquidator transferred almost double the amount required, so they should be able to redeem the excess collateral (total collateral - value burned in KittyCoins). However, attempting to redeem this collateral would break the liquidator's health factor. 

## Vulnerability Details

Consider the scenario where User A' is undercollateralized and their total debt is 10 KittyCoins. User B decides to liquidate User A and deposits around 17 EUR worth of collateral (since the protocol requires 169% collateralization) and mints 10 KittyCoins. The process unfolds as follows:

* B liquidates A.
* After liquidation A's debt is cleared and his KittyCoin balance at protocol is 0 but he still has those 10 KittyCoins in his wallet.
* B receives 10.5 EUR worth of collateral (10 KittyCoins worth of collateral + 5% bonus) back in their wallet and is left with 0 KittyCoins in their wallet.
* B still has 17 EUR worth of collateral deposited in the protocol but their KittyCoinBalance is still 10 KittyCoins in the mappings.
* B can not redeem his remaining  collateral because if he tries to, his health factor will break as the protocol is 169% collateralized, so after spending 17 EUR he is left with 10.5 EUR.

**Consequently, instead of earning a 5% bonus, the liquidator loses 40% of their collateral.**

## Impact

Liquidators may avoid liquidating users with broken health factors, disrupting the integrity of the protocol, which relies on the assumption that users will be liquidated when their health factor breaks. Eventually protocol will become undercollateralized, breaking the system.

## Tools Used

VSCode

## Recommendations

1. Make the collateralization rate and the liquidation bonus arithmetically incentivised so as to allow re-entrancy for a flash loan type of atomic mint within the protocol.
2. Allow an alternative stable coin to be used for repayment should KittyCoin not be available.


# Medium Risk Findings

## <a id='M-01'></a>M-01. KittyVault NOT checking For Stale Prices

_Submitted by [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [0xsalami](/profile/clt01df3800035pfctlu2xr0m), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [evmninja](/profile/clxxng1pm0000mnz6avhempwf), [bytesflow007](/profile/clywm16360006sr4igeys47em), [Brene](/profile/clxrieq96000c4boavunj2s9e), [BryanConquer](/profile/cllw1qa7j0000jw08yme07cs2), [n3smaro](/profile/clyw8gc2200009pkxbzh2omnk), [Caesar](/profile/cly4c6opu00006ixlxrmy6rc2), [nilay27](/profile/cltshhl2x0000cqre7q2bjkf4), [Owenzo](/profile/cls0la11f0000127hfng6wpzl), [0xsagetony](/profile/clnxw408y000gl6086iqksw4c). Selected submission by: [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7)._      
            


## Summary

the `KittyVault.sol` smart contract doesn’t check whether that price data sent by chainlink pricefeeds is stale.

## Vulnerability Details

If `Chainlink pricefeed` returns  pricing data that is stale,`KittyVault::getTotalMeowllateralInAave` and `KittyVault::getUserVaultMeowllateralInEuros` will execute with prices that don’t reflect the current pricing of the asset, and this result in a potential loss of funds for the user and/or the protocol.

## Impact

* Loss of funds for the protocol if a wrong high price is used to check for the assets pricing as this may lead to not liquidating `users` that may be *liquidatable*
* Loss of funds for protocol users is  wrong low price is used in  `KittyVault::getUserVaultMeowllateralInEuros`

## Tools Used

* manual review
* chainlink documentation

## Recommendations

check the `updatedAt` parameter returned from `latestRoundData()` and compare it to a staleness threshold(heartbeat) specified in the chainlink docs.
e.g for `BTC/USD`, the heartbeat is `3600s` i.e.*1 hour*, hence do the check like this...

```diff
function getUserVaultMeowllateralInEuros(address _user) external view returns (uint256) {
-        (, int256 collateralToUsdPrice, , , ) = i_priceFeed.latestRoundData();
+        (, int256 collateralToUsdPrice, ,uint256 updatedAt , ) = i_priceFeed.latestRoundData();
+        if (updatedAt < block.timestamp - 3600) {
+               revert("stale price feed");
+       }
        (, int256 euroPriceFeedAns, , ,) = i_euroPriceFeed.latestRoundData();
        uint256 collateralAns = getUserMeowllateral(_user).mulDiv(uint256(collateralToUsdPrice) * EXTRA_DECIMALS, PRECISION);
        return collateralAns.mulDiv(uint256(euroPriceFeedAns) * EXTRA_DECIMALS, PRECISION);
    }
```

```diff
 function getTotalMeowllateralInAave() public view returns (uint256) {
        (uint256 totalCollateralBase, , , , , ) = i_aavePool.getUserAccountData(address(this));
-        (, int256 collateralToUsdPrice, , , ) = i_priceFeed.latestRoundData();
+        (, int256 collateralToUsdPrice, ,uint256 updatedAt , ) = i_priceFeed.latestRoundData();
+        if (updatedAt < block.timestamp - 3600) {
+               revert("stale price feed");
+       }
        return totalCollateralBase.mulDiv(PRECISION, uint256(collateralToUsdPrice) * EXTRA_DECIMALS);
    }
```

## <a id='M-02'></a>M-02. [M-1] - `getTotalMeowllateralInAave()` using two different Oracles can cause price mismatch!

_Submitted by [nilay27](/profile/cltshhl2x0000cqre7q2bjkf4)._      
            


**Description:**
`i_aavePool.getUserAccountData(address(this))` will return the `totalCollateralBase` in the BASE\_CURRENCY of AAVEV3 (i.e USD with 8 Decimals). However in AAVEV3, AAVE can turn on a single oracle use on any E-mode category. When it is used, the `IAaveOracle` (the oracle used internally by `AavePool`) will return a different price from `AggregatorV3Interface`.

```solidity
function getTotalMeowllateralInAave() public view returns (uint256) {
        (uint256 totalCollateralBase, , , , , ) = i_aavePool.getUserAccountData(address(this));
        //@audit medium: `totalCollateralBase` can have different price feed than `i_priceFeed.latestRoundData()`
        (, int256 collateralToUsdPrice, , , ) = i_priceFeed.latestRoundData();
        return totalCollateralBase.mulDiv(PRECISION, uint256(collateralToUsdPrice) * EXTRA_DECIMALS);
    }
```

**Impact:**
It will lead to incorrect calculation in `getTotalMeowllateralInAave` as the pricing used to `totalCollateralBase` in Aave will be different to `collateralToUsdPrice` returned by `i_priceFeed`, thus the `getTotalMeowllateralInAave` will have slight mismatch in calculation.

**Proof of Concept:**
Add the following mainnet addresses in `HelperConfig.s.sol`

```solidity
function getMainnetConfig() internal pure returns (NetworkConfig memory) {
        return NetworkConfig ({
            aavePool: 0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2,
            euroPriceFeed: 0xb49f677943BC038e9857d61E7d053CaA2C1734C1,
            ethUsdPriceFeed: 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419,
            btcUsdPriceFeed: 0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c, //@audit-info : wrong feed address
            usdcUsdPriceFeed: 0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6,
            weth: 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2,
            wbtc: 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599,
            usdc: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
        });
    }
```

Add the following test in KittyFiTest.t.sol

```solidity
modifier userDepositsBTCCollateral(){
        deal(config.wbtc, user, 10e8); // wbtc has 8 decimals
        uint256 wbtcToDeposit = 5e8;

        // create btc vault:
        vm.prank(meowntainer);
        kittyPool.meownufactureKittyVault(config.wbtc, config.btcUsdPriceFeed);

        wbtcVault = KittyVault(kittyPool.getTokenToVault(config.wbtc));

        // user deposits 5 btc to vault as collateral
        vm.startPrank(user);
        IERC20(config.wbtc).approve(address(wbtcVault), wbtcToDeposit);
        kittyPool.depawsitMeowllateral(config.wbtc, wbtcToDeposit);
        vm.stopPrank();
        _;
    }

    function test_btcPriceDifference() public userDepositsBTCCollateral{
        // supply only 1 btc to aave, so that `totalCollateralBaseBTC` is essentially price of BTC
        uint256 wbtcToSupply = 1e8;
        // move collateral to aave
        vm.prank(meowntainer);
        wbtcVault.purrrCollateralToAave(wbtcToSupply);
        assertEq(wbtcVault.totalMeowllateralInVault(), 4e8);

        // fetch collateralValue according to 
        (uint256 totalCollateralBaseBTC, , , , , ) = IAavePool(config.aavePool).getUserAccountData(address(wbtcVault));
        console.log("totalCollateralBaseBTC", totalCollateralBaseBTC); // 12000000000000 i.e 1.2e13
        
        //fetch the price of BTC from Chainlink Oracle
        AggregatorV3Interface btcUsdPriceFeed = AggregatorV3Interface(config.btcUsdPriceFeed);
        (, int256 btcUSDPrice, , , ) = btcUsdPriceFeed.latestRoundData();
        console.log("btc price in oracle", uint256(btcUSDPrice));
    }
```

Run the test using the command:

```Solidity
forge test --via-ir --fork-url https://mainnet.infura.io/v3/<your_infura_key> -vvv --mt test_btcPriceDifference
```

**Log**

```Solidity
  totalCollateralBaseBTC 5720896508833
  btc price in oracle 5719859727059
```

Value of 1 BTC in aave is 5720896508833 i.e 57208.96508833 USD
Value of 1 BTC in chainlink is 5719859727059 i.e 57198.59727059 USD

Difference between price: \~10USD per BTC

**Recommended Mitigation:**
Use `IAaveOracle` to fetch `collateralToUsdPrice`

## <a id='M-03'></a>M-03. There is no incentive to liquidate small positions

_Submitted by [14xSachet](/profile/clz15oltz0006q1gtgf0ywir8)._      
            


## Summary

Due to increase in gas costs, there is no incentive for the liquidators to liquidate accounts having minimal values, lets say 5 EUR.

## Vulnerability Details

Liquidators liquidate users for the profit they can make. If there is no profit to be made than there will be no one to call the liquidate function. Lets say there is an account having 6 EUR worth of collateral and 5 KittyCoins minted. This user is undercollateralized and must be liquidated in order to ensure that the protocol remains overcollateralized. Because the value of the account is so low, after gas costs, liquidators will not make a profit liquidating this user. 

In the end these low value accounts will never get liquidated, leaving the protocol with bad debt and can even cause the protocol to be undercollateralized with enough small value accounts being underwater.

## Impact

A large number of dust account could bring bad debts to the protocol

## Tools Used

Manual Review

## Recommendations

Try adding a minimum minting threshold. This will eliminate the accounts with minimal values, protecting the protocol from being in debt.


# Low Risk Findings

## <a id='L-01'></a>L-01. In `KittyPool` users collateral could potentially lock, leading to financial loss during the withdrew all the Collateral value form vault and tries to mint `kittyCoin` without deposit any Collateral.

_Submitted by [BadalSharma](/profile/clqr9x7ok0000xj22sa1owbxb), [ayamercy](/team/clz09i1y50001hibkcpjzt4np). Selected submission by: [BadalSharma](/profile/clqr9x7ok0000xj22sa1owbxb)._      
            


**Description:** The `KittyPool` contract is assigned the role of allowing the user to withdraw their collateral. The KittyPool contract routes the call to the respective vault for deposit and withdrawal collateral which is created for every collateral token used in the protocol.

```javascript
   function whiskdrawMeowllateral(address _token, uint256 _ameownt) external tokenExists(_token) {
        IKittyVault(tokenToVault[_token]).executeWhiskdrawal(msg.sender, _ameownt);
        require(_hasEnoughMeowllateral(msg.sender), KittyPool__NotEnoughMeowllateralPurrrr());
    }
```

However, users get a `panic error` when withdrawing all the Collateral value from the vault and When users try to mint `kittyCoin` without depositing any Collateral.

`ERROR:`

```bash
Failing tests:
Encountered 1 failing test in test/KittyFiPoc.t.sol:KittyFiPoc
[FAIL. Reason: panic: division or modulo by zero (0x12)] test_CanUserWithdrewAllCollateral() (gas: 248302)
```

It's happening because of one of his inner functions.

```javascript
function getUserMeowllateral(address _user) public view returns (uint256) {
        uint256 totalMeowllateralOfVault = getTotalMeowllateral();
        return userToCattyNip[_user].mulDiv(totalMeowllateralOfVault, totalCattyNip);
    }
```

**Impact:** A panic with the reason "division or modulo by zero" (error code 0x12) indicates that the contract code attempted to divide by zero, which is a critical runtime error in Solidity. This type of error can break the protocol and cause it to behave unexpectedly or fail entirely.

**Proof Of Concept:**

<details>

<summary> Proof of Concept (foundry test)</summary>

1. This is happening because a user tries to withdraw all his collateral and the contract has 0 balance to prevent this we need to add some collateral in the contract beforehand or check recommendations.

```javascript
  function test_CanUserWithdrewAllCollateral() public {
        uint256 depositAmount = 5 ether;
        uint256 withdrewAmount = 5 ether;

        vm.startPrank(user);
         // Deposit the collateral
         vm.startPrank(user);
         IERC20(weth).approve(address(wethVault), depositAmount);
         kittyPool.depawsitMeowllateral(weth, depositAmount);

        // note: unComment me if you do not want to get an error
        // address user2 = makeAddr("user2");
        // deal(weth, user2, AMOUNT);

        // vm.startPrank(user2);
        // IERC20(weth).approve(address(wethVault), depositAmount);
        // kittyPool.depawsitMeowllateral(weth, depositAmount);
        // assertEq(wethVault.getUserMeowllateral(user), depositAmount);
        // vm.stopPrank();

         // User want to withdrew all amount but unable to withdrew 
        kittyPool.whiskdrawMeowllateral(weth, withdrewAmount);
        vm.stopPrank();
    }
```

1. This causes a panic error when the contract has 0 collateral and a `user` tries to mint `kittyCoin` without depositing any collateral.

```javascript
function test_UserFundStuctInContract() public {
        uint256 amountToMint = 20e18;
        address attacker = makeAddr("attacker");

        //An attacker tries to mint kittyCoin without depositing any collateral 
        // and also KittyPool does'nt have any collateral
        vm.startPrank(attacker);
        vm.expectRevert( "total Meowllateral is zero");
        kittyPool.meowintKittyCoin(amountToMint);
        vm.stopPrank();
    }
```

</details>

**Tools Used:**

Manual Review
Foundry

**Recommendations:**

We need to adjust the logic in `KittyVault::getUserMeowllateral`

```diff
 function getUserMeowllateral(address _user) public view returns (uint256) {
        uint256 totalMeowllateralOfVault = getTotalMeowllateral();
+       require(totalMeowllateralOfVault > 0, "total Meowllateral is zero");
        return userToCattyNip[_user].mulDiv(totalMeowllateralOfVault, totalCattyNip);
    }
```

## <a id='L-02'></a>L-02. Collateral deposited won't earn additional yield when hits supply limitation in Aave pool

_Submitted by [soupy](/profile/clxtrdlee0002mh58dhdplsq9)._      
            


## Summary

Additional collateral deposited in KittyPool will just serve as backing the KittyCoin but won't get extra yield when the supply limitation is hit in Aave pool causing user to mistakenly think their deposits earn them additional interest.

## Vulnerability Details

User will be able to earn yield from their deposits only when meowntainer perform `KittyValut:purrrCollateralToAave` function. However, there is a supply limitation which is not clearly known to the end user. If the total supply handled in Aepool is exceeded, meowntainer will no longer be able to perform the `KittyValut:purrrCollateralToAave` function, resulting any extra collateral deposited to KittyPool that has yet to transfer to Aave pool won't be earning any yield.

Proof of Concept:
In `test/KittyFiTest.t.sol`, add `usdc` related setup and run test on `test_audit_purrrCollateralToAave_supplyLimitation()`

```javascript

    function setUp() external {
        HelperConfig helperConfig = new HelperConfig();
        config = helperConfig.getNetworkConfig();
        weth = config.weth;
        usdc = config.usdc;
        deal(weth, user, AMOUNT);
        deal(usdc, user, AMOUNT);

        kittyPool = new KittyPool(meowntainer, config.euroPriceFeed, config.aavePool);

        vm.startPrank(meowntainer);
        kittyPool.meownufactureKittyVault(config.weth, config.ethUsdPriceFeed);
        kittyPool.meownufactureKittyVault(config.usdc, config.ethUsdPriceFeed);
        vm.stopPrank();

        kittyCoin = KittyCoin(kittyPool.getKittyCoin());
        wethVault = KittyVault(kittyPool.getTokenToVault(config.weth));
        usdcVault = KittyVault(kittyPool.getTokenToVault(config.usdc));
    }

    function test_audit_purrrCollateralToAave_supplyLimitation() public {
        uint256 toDeposit = 5 ether;

        vm.startPrank(user);
        IERC20(usdc).approve(address(usdcVault), toDeposit);
        kittyPool.depawsitMeowllateral(usdc, toDeposit);
        vm.stopPrank();

        // first xfer of usdc from usdcVault to aavePool
        vm.startPrank(meowntainer);
        uint256 xferAmount = 1e14;
        usdcVault.purrrCollateralToAave(xferAmount);
        assert(usdcVault.getTotalMeowllateralInAave() > 0);

        // second transfer of bigger amount from usdcVault to aavePool
        uint256 xferBiggerAmount = 1e16;
        vm.expectRevert();
        usdcVault.purrrCollateralToAave(xferBiggerAmount);

        vm.stopPrank();
    }
```

The test will pass with `forge test --via-ir --fork-url $SEPOLIA_RPC_URL --match-test test_audit_purrrCollateralToAave_supplyLimitation -vvvv `

In the console, the followings will be displayed:

```javascript

Ran 1 test for test/KittyFiTest.t.sol:KittyFiTest
[PASS] test_audit_purrrCollateralToAave_supplyLimitation() (gas: 479253)

Traces:
  [479253] KittyFiTest::test_audit_purrrCollateralToAave_supplyLimitation()
    ├─ [0] VM::startPrank(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D])
    │   └─ ← [Return] 
    ├─ [24619] TestnetERC20::approve(KittyVault: [0xE84c9bd544D87642DD01E7415f2a54fc08061FB0], 5000000000000000000 [5e18])
    │   ├─ emit Approval(owner: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], spender: KittyVault: [0xE84c9bd544D87642DD01E7415f2a54fc08061FB0], value: 5000000000000000000 [5e18])
    │   └─ ← [Return] true

.......


    │   │   │   │   ├─ [3176] InitializableImmutableAdminUpgradeabilityProxy::getSupplyData() [staticcall]
    │   │   │   │   │   ├─ [2615] StableDebtToken::getSupplyData() [delegatecall]
    │   │   │   │   │   │   └─ ← [Return] 3246908447782 [3.246e12], 3249282345040 [3.249e12], 55884533678440988670555868 [5.588e25], 1722520872 [1.722e9]
    │   │   │   │   │   └─ ← [Return] 3246908447782 [3.246e12], 3249282345040 [3.249e12], 55884533678440988670555868 [5.588e25], 1722520872 [1.722e9]
    │   │   │   │   ├─ [924] InitializableImmutableAdminUpgradeabilityProxy::scaledTotalSupply() [staticcall]
    │   │   │   │   │   ├─ [375] AToken::scaledTotalSupply() [delegatecall]
    │   │   │   │   │   │   └─ ← [Return] 116924875644443 [1.169e14]
    │   │   │   │   │   └─ ← [Return] 116924875644443 [1.169e14]
    │   │   │   │   └─ ← [Revert] revert: 51
    │   │   │   └─ ← [Revert] revert: 51
    │   │   └─ ← [Revert] revert: 51
    │   └─ ← [Revert] revert: 51
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Return] 
```

In the first transfer of `1e14 usdc`, the process will go through and successfully move collateral to Aavepool. However, the second transfer of `1e16 usdc` will `revert with error code 51` which is related to `ReserveLogic - Liquidity index overflows` [(doc)](https://docs.aave.com/developers/v/2.0/guides/troubleshooting-errors#reference-guide). In the Aave protocol github repo related to `ValidationLogic.sol`[(ref)](https://github.com/aave/aave-v3-core/blob/feb3f20885c73025f40cc272b59e7eacfaa02fe4/contracts/protocol/libraries/logic/ValidationLogic.sol#L71-L77), there is a maximum supply for a given type of collateral which cannot be exceeded.

## Impact

Any additional collateral deposited to KittyPool which exceeds the Aaeve supply limit, can't be moved to Aave to earn yield and just stay in the KittyVault serving solely as a backing to KittyCoin, deviating from the protocol statement to enable user to earn yield from their deposits.

## Tools Used

Manual review with forge test

## Recommendations

To implement control check on the collateral supply limit to Aave pool and additional functions enable both meowntainer and user to know the state of the current supply.





    