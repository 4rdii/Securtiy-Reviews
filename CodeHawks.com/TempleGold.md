# TempleGold - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01. Incompatibility with Multisig Wallets in `TempleGold::send` Function](#H-01)
    - [H-02. Future stakers are paid with rewards that have been accrued from the past due to miscalculation in userRewardPerTokenPaid and _perTokenReward.](#H-02)
- ## Medium Risk Findings
    - [M-01. Not upadting `_totalAuctionTokenAllocation` when removing last auction config at cooldown leads to wrong accounting of `_totalAuctionTokenAllocation` and permanent lock of auction tokens](#M-01)
    - [M-02. Changes to vesting period is not handled inside `_getVestingRate` ](#M-02)
- ## Low Risk Findings
    - [L-01. Incosistent message generation in TempleTeleporter.quote() and TempleTeleporter.teleport() results in inaccurate required fee calculation by TempleTeleporter.quote()](#L-01)
    - [L-02. Lack of Comprehensive Pausability for Critical Functions](#L-02)
    - [L-03. Auction tokens cannot be recovered for the first ever spice auction](#L-03)
    - [L-04. TempleGold tokens cannot be recovered when a `DaiGoldAuction` ends with 0 bids](#L-04)
    - [L-05. Malicious user can prevent  `rewardData.perodfinish` from ending by calling `TempleGoldStaking::distributeRewards()` before the end of the reward duration when no starter is set.](#L-05)
    - [L-06. Incorrect templeGold minting due to unresolved accumulation in `TempleGold::setVestingFactor`](#L-06)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: TempleDAO

### Dates: Jul 4th, 2024 - Jul 11th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-templegold)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 2
   - Low: 6


# High Risk Findings

## <a id='H-01'></a>H-01. Incompatibility with Multisig Wallets in `TempleGold::send` Function

_Submitted by [matej](/profile/clqf9x2s3000r83pidasj858e), [n08ita](/profile/cly5hxrwf000poucy2n0ul2bx), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Bauchibred](/profile/clk9ibj6p0002mh08c603lr2j), [ptsanev](/profile/clk41ds6d0056la0868j7rf0l), [Beyond Audit](/team/cly7pmb9l000b48ek3tx2x490), [WildSniper](/profile/cls31c3z900004sonbd37v0ry), [ZdravkoHr](/profile/clkmey53n0018l008gwzuqcmu), [SBSecurity](/team/clkuz8xt7001vl608nphmevro), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [nisedo](/profile/clk3saar60000l608gsamuvnw), [y4y](/profile/cllq879u70000mo08o0n110vi), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [ke1caM](/profile/clk46fjfm0014la08xl7mwtis), [x18a6](/profile/cly76bl8k0002qr8yn2cbuft1), [0xspryon](/profile/clo19fw280000mf08c4yazene), [Fortis Audits](/team/cly2eas2s000gpgez427qfcw9), [zaevlad](/profile/clk4cjkez0004mo0871jg7ktq), [yotov721](/profile/clwf98b9t0000y257ug1muf8m), [y0ng0p3](/profile/clk4ouhib000al808roszkn4e), [KiteWeb3](/profile/clk9pzw3j000smh08313lj91l), [0xaman](/profile/cln8ldajr0000jy085cig6cz4), [Trident Audits](/team/clyfyc6wf000pr98ax0arts8w), [pep7siup](/profile/clktaa8x50014mi08472cywse), [Pelz](/profile/clokuwofs000yih08n1oqrf6d), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [adriro](/profile/cllkxto1y001yjx08kh87bqge), [n0kto](/profile/clm0jkw6w0000jv08gaj4hof4), [1337web3](/profile/clw6sn30t0000mruqsj84nk6p), [0xhals](/profile/clkfub7qh0002l508bt5xdugv), [turvec](/profile/clkghm50k0008l70869gpsyyw), [Joshuajee](/profile/clp8tjvhb0000r61pfr2owzuy), [0xsandy](/profile/clk43kus5009imb0830ko7dxy), [billoBaggeBilleyan](/profile/clp81a9bn0000iaa0sgjkbtai), [0xDemon](/profile/clqm9tqvx00045lj2c6xi9gvv), [0xlucky](/profile/cllvmhg1i0008md080shk9pzx). Selected submission by: [Fortis Audits](/team/cly2eas2s000gpgez427qfcw9)._      
            


## Summary:

The `send` function in `TempleGold` smart contract is designed to facilitate cross-chain token transfers using LayerZero. However, it contains a restrictive condition that disallows transfers if the sender's address does not match the recipient's address. This creates a significant issue for users utilizing multisig wallets, as these wallets often have different addresses across different chains, preventing them from transferring their funds cross-chain.

## Vulnerability Detail:

The vulnerability lies in the address validation check: `if (msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }`. This condition ensures that the sender and the recipient addresses are identical, which is not the case for multisig wallets operating across different chains such as Ethereum and Arbitrum.

## Code Snippet:

```javascript
function send(
        SendParam calldata _sendParam,
        MessagingFee calldata _fee,
        address _refundAddress
    ) external payable virtual override(IOFT, OFTCore) returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt) {
        if (_sendParam.composeMsg.length > 0) { revert CannotCompose(); }
        /// cast bytes32 to address
        address _to = _sendParam.to.bytes32ToAddress();
        /// @dev user can cross-chain transfer to self
@>      if (msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }

        // @dev Applies the token transfers regarding this send() operation.
        (uint256 amountSentLD, uint256 amountReceivedLD) = _debit(
            msg.sender,
            _sendParam.amountLD,
            _sendParam.minAmountLD,
            _sendParam.dstEid
        );

        // @dev Builds the options and OFT message to quote in the endpoint.
        (bytes memory message, bytes memory options) = _buildMsgAndOptions(_sendParam, amountReceivedLD);

        // @dev Sends the message to the LayerZero endpoint and returns the LayerZero msg receipt.
        msgReceipt = _lzSend(_sendParam.dstEid, message, options, _fee, _refundAddress);
        // @dev Formulate the OFT receipt.
        oftReceipt = OFTReceipt(amountSentLD, amountReceivedLD);

        emit OFTSent(msgReceipt.guid, _sendParam.dstEid, msg.sender, amountSentLD, amountReceivedLD);
    }
```

## Impact:

This vulnerability prevents users of multisig wallets from performing cross-chain transfers of their tokens. The condition `if (msg.sender != _to)` fails for multisig wallet users due to differing addresses across chains, which:

* Restricts the usability of the contract for multisig wallet users.
* Limits the flexibility and accessibility of cross-chain token transfers.
* Potentially deters users from adopting the contract due to this inflexibility.

## Proof Of Concept:

1. User A, who owns a multisig wallet, attempts to transfer his temple tokens from Ethereum to Arbitrum using the `send` function.
2. The `send` function checks if the sender's address matches the recipient's address.
3. The condition `if (msg.sender != _to)` fails due to the differing addresses of the multisig wallet on Ethereum and Arbitrum.
4. The transaction reverts, preventing User A from completing the cross-chain transfer.

**Proof Of Code:**

Place the following code in the `TempleGoldLayerZero.t.sol` contract:

```javascript
    function test_FortisAudits_MultiSigWallet_LossFunds() public {
        address multisig_MainNet = makeAddr("multisig-mainnet"); // user's address
        address multisig_Arb = makeAddr("multisig-arb"); // user's same address which is undeployed in arbitrum
        vm.deal(multisig_MainNet, 100 ether);
        vm.deal(multisig_Arb, 100 ether);
        aTempleGold.mint(multisig_MainNet, 100 ether);
        uint256 tokensToSend = 1 ether;
        bytes memory options = OptionsBuilder.newOptions().addExecutorLzReceiveOption(200_000, 0);
        SendParam memory sendParam =
            SendParam(bEid, addressToBytes32(multisig_Arb), tokensToSend, tokensToSend, options, "", "");

        MessagingFee memory fee1 = aTempleGold.quoteSend(sendParam, false);

        // If a msg.sender uses a mulitisig wallet in the destination chain, thus funds cannot be
        // transferred and DOS the user
        vm.startPrank(multisig_MainNet);
        vm.expectRevert(abi.encodeWithSelector(ITempleGold.NonTransferrable.selector, multisig_MainNet, multisig_Arb));
        aTempleGold.send{ value: fee1.nativeFee }(sendParam, fee1, multisig_MainNet);
        verifyPackets(bEid, addressToBytes32(address(bTempleGold)));
        vm.stopPrank();
    }
```

## Tools Used:

Manual code review

Foundry

## Recommendations:

1. **Remove or Modify Address Check**: Consider modifying or removing the restrictive address check to accommodate multisig wallet users. For example:

   ```javascript
   if (msg.sender != _to) {
     // Additional validation to check if msg.sender is a multisig wallet or other criteria
     // revert ITempleGold.NonTransferrable(msg.sender, _to);
   }
   ```

2. **Implement Whitelisting**: Implement a whitelisting mechanism for known multisig wallet addresses across chains to bypass the restrictive check.

3. **User Validation**: Introduce a more sophisticated user validation process that allows for different addresses on different chains but ensures the integrity of the cross-chain transfer.

By addressing this vulnerability, the contract will become more inclusive and practical for a broader range of users, particularly those using multisig wallets.

## <a id='H-02'></a>H-02. Future stakers are paid with rewards that have been accrued from the past due to miscalculation in userRewardPerTokenPaid and _perTokenReward.

_Submitted by [mt030d](/profile/cly360v0o000014p4y4rxyufy), [0x3b](/profile/clk3yiyaq002imf088cd3644k), [hyperhorse](/profile/clyd0a8es000ezbebc71je4yn), [touthang](/profile/clk9bd0tr0000mi084i1py0rx), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [adriro](/profile/cllkxto1y001yjx08kh87bqge), [jesjupyter](/profile/clxvvtui0000310nimqwnkuvi), [ZukaNoPro](/profile/clvhorw4z000049xga7gskpfr). Selected submission by: [touthang](/profile/clk9bd0tr0000mi084i1py0rx)._      
            


## Summary

Furure stakers are always vested rewards using the total `rewardPerToken` without deducting how much `rewardPerToken` has already been accumulated before they stake.

Any new staker will be rewarded with `rewardData.rewardPerTokenStored` that have been accumulated since creation of contract. That means a user who hasn't been staking from the beginning will be rewarded the same as a user who has been staking sing the beginning. Or in other words the contract assumes that every user staked when the `rewardData.rewardPerTokenStored` is 0.

This is exactly the same problem mentioned here by [rareskills](https://www.rareskills.io/post/staking-algorithm#:~:text=What%20if%20someone,at%20block%2020.).

## Vulnerability Details

`stake` calls `updateReward` modifier and we can see that every new staker's `userRewardPerTokenPaid` will be 0 since vesting rate is currently 0;

```Solidity
    modifier updateReward(address _account, uint256 _index) {
        {
            // stack too deep
            rewardData.rewardPerTokenStored = uint216(_rewardPerToken());
            rewardData.lastUpdateTime = uint40(_lastTimeRewardApplicable(rewardData.periodFinish));
            if (_account != address(0)) {
                StakeInfo memory _stakeInfo = _stakeInfos[_account][_index];
                uint256 vestingRate = _getVestingRate(_stakeInfo);
                claimableRewards[_account][_index] = _earned(_stakeInfo, _account, _index);
                userRewardPerTokenPaid[_account][_index] = vestingRate * uint256(rewardData.rewardPerTokenStored) / 1e18;
            }
        }
        _;
    }
```

```Solidity
    function _getVestingRate(StakeInfo memory _stakeInfo) internal view returns (uint256 vestingRate) {
        if (_stakeInfo.stakeTime == 0) {
            return 0;
        }
        if (block.timestamp > _stakeInfo.fullyVestedAt) {
            vestingRate = 1e18;
        } else {
            vestingRate = (block.timestamp - _stakeInfo.stakeTime) * 1e18 / vestingPeriod;
        }
    }
```

Suppose,

* after 1 year the accumulated `rewardPerTokenStored` is `1000e18`, which means every token staked since the beginning has earned 1000 tokens each.
* a new user Bob stakes 100 TEMPLE tokens.
* assuming the vesting period is 16 weeks, Bob claim his rewards after 16 weeks
* now his reward calculation is;

`_perTokenReward = _rewardPerToken();`

we can see that \_`_perTokenReward` for the new staker is the total  \_`_rewarsPerToken` which has been accumulating since the beginning.

```Solidity
        return  
            (_stakeInfo.amount * (_perTokenReward - userRewardPerTokenPaid[_account][_index])) / 1e18 +
            claimableRewards[_account][_index];
    }
```

* `(100 TEMPLE * (1000e18 - 0)) / 1e18 + 0`
* `= 100,000` TGLD rewards for staking 16 weeks.

This is assuming no new rewards are added, where bob should've gotten 0 TGLD as rewards. Bob just earned rewards equivalent to staking 1 year + 16 weeks(for new rewards).

The issue here is that the contract does not account for the already accumulated `rewardPerTokenStored` from the past when calculating the new staker's `userRewardPerTokenPaid` and calculates it using the total `rewardPerToken` :

&#x20;` userRewardPerTokenPaid[_account][_index] = vestingRate * uint256(rewardData.rewardPerTokenStored) / 1e18;`.

## Impact

Loss of reward tokens and unfair reward calculation for initial stakers or DOS(not enough reward tokens).
It also opens an attack path where a user can steal unclaimed rewards even when there are no new rewards added i.e. a user who enters after no new rewards are distributed will still get rewards from the past.

## Tools Used

manual

## Recommendations

Introduce a new variable;

```solidity
    //@audit a new variable to keep track of the already accumulated rewardPerToken before the user enters the protocol
    mapping(address account => mapping(uint256 index => uint256 amount)) public userRewardDebt; 
```

note that we cannot use the variable `userRewardPerTokenPaid` to track the already accumulated rewardPerToken because of the vestingRate, that would introduce another problem.

```diff
    function stakeFor(address _for, uint256 _amount) public whenNotPaused {
        if (_amount == 0) revert CommonEventsAndErrors.ExpectedNonZero();

        stakingToken.safeTransferFrom(msg.sender, address(this), _amount);
        uint256 _lastIndex = _accountLastStakeIndex[_for];
        _accountLastStakeIndex[_for] = ++_lastIndex; 

+       userRewardDebt[_for][_lastIndex] = _rewardPerToken(); //@audit we need to make sure we do it here before _applyStake, because it calls updateReward 

        _applyStake(_for, _amount, _lastIndex); 
        _moveDelegates(address(0), delegates[_for], _amount);
    }
```

```diff
    modifier updateReward(address _account, uint256 _index) {
        {
            // stack too deep
            rewardData.rewardPerTokenStored = uint216(_rewardPerToken());
            rewardData.lastUpdateTime = uint40(_lastTimeRewardApplicable(rewardData.periodFinish));
            if (_account != address(0)) {
                StakeInfo memory _stakeInfo = _stakeInfos[_account][_index];
                uint256 vestingRate = _getVestingRate(_stakeInfo);
                claimableRewards[_account][_index] = _earned(_stakeInfo, _account, _index);
-               userRewardPerTokenPaid[_account][_index] = vestingRate * uint256(rewardData.rewardPerTokenStored) / 1e18;
+               //@audit vest only the newly accrued rewardPerToken after the user enters 
+               userRewardPerTokenPaid[_account][_index] = vestingRate * (uint256(rewardData.rewardPerTokenStored) - userRewardDebt[_account][_index] ) / 1e18;
                // this will still be zero when new staker enters, but since we deduct the userRewardDebt before calculating it, we will only reward with the newly accrued rewardPerToken
            }
        }
        _;
    }
```

```diff
    function _earned(
        StakeInfo memory _stakeInfo,
        address _account,
        uint256 _index
    ) internal view returns (uint256) {
        uint256 vestingRate = _getVestingRate(_stakeInfo);
        if (vestingRate == 0) {
            return 0;
        }
        uint256 _perTokenReward;
        if (vestingRate == 1e18) { 
-            _perTokenReward = _rewardPerToken();
+            _perTokenReward = _rewardPerToken() - userRewardDebt[_account][_index];
        } else { 
-           _perTokenReward = _rewardPerToken() * vestingRate / 1e18;
+           _perTokenReward = (_rewardPerToken() - userRewardDebt[_account][_index]) * vestingRate / 1e18;
        }
        
        return
            (_stakeInfo.amount * (_perTokenReward - userRewardPerTokenPaid[_account][_index])) / 1e18 +
            claimableRewards[_account][_index];
    }
```

This ensures that `_perTokenReward` for the staker is only the newly accumulated `rewardPerToken`, post staking.


# Medium Risk Findings

## <a id='M-01'></a>M-01. Not upadting `_totalAuctionTokenAllocation` when removing last auction config at cooldown leads to wrong accounting of `_totalAuctionTokenAllocation` and permanent lock of auction tokens

_Submitted by [0x3b](/profile/clk3yiyaq002imf088cd3644k), [ZdravkoHr](/profile/clkmey53n0018l008gwzuqcmu), [touthang](/profile/clk9bd0tr0000mi084i1py0rx), [cryptomoon](/profile/clo8a861c001emc092vef10q1), [y4y](/profile/cllq879u70000mo08o0n110vi), [0xbrivan2](/profile/clp1dquol0000kupd1wulucuy), [Cryptara](/profile/clyd2whwx000an25ixhmg3hhl), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [kildren](/profile/clxmmb9g7000010aw9we896mw), [0xCiphky](/profile/clkrx20ym0000ju088s337njb), [GEEK](/profile/clk48fvzr003wla08us7vyyv4), [marsWhiteHacker](/profile/cloxak63u0000l7088jqhgbbt), [nisedo](/profile/clk3saar60000l608gsamuvnw), [stanchev](/profile/cly6vo6yg0000bkngvgt6njut). Selected submission by: [0xbrivan2](/profile/clp1dquol0000kupd1wulucuy)._      
            


## Vulnerability Details

[removeAuctionConfig](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L107-L133) is used to remove auction config set for last epoch. If the auction was started but the cooldown has not passed yet, both auction config and epoch info are deleted and the `_currentEpochId` is decremented:

```solidity
function removeAuctionConfig() external override onlyDAOExecutor {
    /// only delete latest epoch if auction is not started
    uint256 id = _currentEpochId;
        
    EpochInfo storage info = epochs[id];
    // ...

    bool configSetButAuctionStartNotCalled = auctionConfigs[id+1].duration > 0;
    if (!configSetButAuctionStartNotCalled) {
        /// @dev unlikely because this is a DAO execution, but avoid deleting old ended auctions
        if (info.hasEnded()) { revert AuctionEnded(); }
        /// auction was started but cooldown has not passed yet
        delete auctionConfigs[id];
        delete epochs[id];
        _currentEpochId = id - 1;
        // @audit NOT UPDATING `_totalAuctionTokenAllocation`
        emit AuctionConfigRemoved(id, id);
    } 
    // ...
}
```

The issue is that `_totalAuctionTokenAllocation` is not updated in this situation and this will lead to the following consequences for the next epochs:

1. `_totalAuctionTokenAllocation` will always contain extra auction tokens for all subsequent epochs.
2. Those extra auction tokens can not be recovered.

Consider the following example (that is demonstrated by PoC below) to better explain the consequences:

* For simplicity, we set the auction config for the first epoch with `100 ether` auction tokens sent to the Spice contract
* The auction is started, and based on the [startAuction logic](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L158-L173), the following states are updated as follows:
  * `info.totalAuctionTokenAmount` = 100 ether:
  ```solidity
      uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[auctionToken]; // @audit  = 0
      uint256 balance = IERC20(auctionToken).balanceOf(address(this)); // @audit 100 ether
      uint256 epochAuctionTokenAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[auctionToken]); // @audit 100 ether
      // ...
      info.totalAuctionTokenAmount = epochAuctionTokenAmount; // @audit 100 ether
  ```
  * `_totalAuctionTokenAllocation[auctionToken]` = 100 ether:
  ```solidity
      _totalAuctionTokenAllocation[auctionToken] = totalAuctionTokenAllocation + epochAuctionTokenAmount;
  ```
* Now, for some reason, the DaoExecutor removes the auction config for the last epoch (the first epoch in our example). Because the epoch is still at cooldown, the following block of [removeAuctionConfig](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L119-L127) will be executed:
  ```solidity
      delete auctionConfigs[id];
      delete epochs[id];
      _currentEpochId = id - 1;
  ```
  **Notice that `_totalAuctionTokenAllocation` is NOT updated.**
* Now, the DaoExecutor wants to set up a new auction (with `100 ether` auction token amount) and starts it, so he calls [setAuctionConfig](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L84-L104) and calls [startAuction](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L138-L176). **Normally, we should not feed the Spice contract with `100 ether` because the contract already holds it from the previous deleted auction**. However, because the `_totalAuctionTokenAllocation` still holds `100 ether`, the `epochAuctionTokenAmount` will be evaluated to zero, and thus the transaction will revert:
  ```solidity
      uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[auctionToken]; // @audit 100 ether
      uint256 balance = IERC20(auctionToken).balanceOf(address(this)); // @audit 100 ether
      uint256 epochAuctionTokenAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[auctionToken]); // @audit = 0

      if (epochAuctionTokenAmount < config.minimumDistributedAuctionToken) { revert NotEnoughAuctionTokens(); } // @audit tx will revert
  ```
  So, the contract should be fed with an extra 100 ether if we want to start a new auction with 100 ether tokens.
* Now, we fed the Spice contract with the extra 100 ether that we were obliged to, the `_totalAuctionTokenAllocation` will be updated wrongly:
  ```solidity
      uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[auctionToken]; // @audit 100 ether
      uint256 balance = IERC20(auctionToken).balanceOf(address(this)); // @audit 200 ether
      uint256 epochAuctionTokenAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[auctionToken]); // @audit 100 ether
      // ...

      info.totalAuctionTokenAmount = epochAuctionTokenAmount; // @audit 100 ether
      // Keep track of total allocation auction tokens per epoch
      _totalAuctionTokenAllocation[auctionToken] = totalAuctionTokenAllocation + epochAuctionTokenAmount; // @audit 200 ether !!!!!
  ```
  We can see that the `_totalAuctionTokenAllocation[auctionToken]` is updated to `200 ether` while it should have been updated to `100 ether` because there is only ONE auction running with 100 ether tokens.
* Even if the DaoExecutor attempts to recover those extra 100 ether tokens stuck in the contract after the auction ends, he will not be able to do so. For simplicity, let's consider no one has claimed his tokens yet:
  ```solidity
  function recoverToken(
      address token,
      address to,
      uint256 amount
  ) external override onlyDAOExecutor {
      // ...

      uint256 epochId = _currentEpochId;
      EpochInfo storage info = epochs[epochId];
      // ....
      uint256 totalAuctionTokenAllocation = _totalAuctionTokenAllocation[token]; // @audit 200 ether
      uint256 balance = IERC20(token).balanceOf(address(this)); // @audit 200 ether
      uint256 maxRecoverAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[token]); // @audit = 0
      
      if (amount > maxRecoverAmount) { revert CommonEventsAndErrors.InvalidParam(); } // @audit tx will revert

  }
  ```

### PoC

The following PoC demonstrates the example illustrated above. Copy/Paste the following code into `test/forge/templegold/SpiceAuction.t.sol`:

```solidity
contract SpiceAuctionAccessTest is SpiceAuctionTestBase {

    error NotEnoughAuctionTokens();
    error InvalidParam();

    // ...
function testPoC() public {
    // templeGold is the auction token, with 100 ether amount balance for spice contract
    _startAuction(true, true);
    address auctionToken = spice.getAuctionTokenForCurrentEpoch();
    vm.startPrank(daoExecutor);
    spice.removeAuctionConfig(); // removing the latest epoch config before it starts
        

    uint256 auctionTokenBalance = IERC20(auctionToken).balanceOf(address(spice));
    assertEq(100 ether, auctionTokenBalance);
    // Now, we should start the next epoch, BUT WE DO NOT NEED TO SEND AUCTION TOKENS TO THE SPICE CONTRACT
    // Because the contract already has 100 ether balance.
    // But, if we attempt to start the auction without feeding the contract -> it will revert 

    ISpiceAuction.SpiceAuctionConfig memory config = _getAuctionConfig();
    spice.setAuctionConfig(config);
    if (config.starter != address(0)) { vm.startPrank(config.starter); }
    vm.warp(block.timestamp + config.waitPeriod);
    vm.expectRevert(abi.encodeWithSelector(NotEnoughAuctionTokens.selector)); // Revert message
    spice.startAuction();

    // So, we are obliged to feed to the contract with other auction tokens
    auctionToken = config.isTempleGoldAuctionToken ? address(templeGold) : spice.spiceToken();
    dealAdditional(IERC20(auctionToken), address(spice), 100 ether);

    // Now, the contract holds 200 ether of auction token (extra 100 ether were obliged to be added)
    auctionTokenBalance = IERC20(auctionToken).balanceOf(address(spice));
    assertEq(200 ether, auctionTokenBalance);

    uint256 epoch = spice.currentEpoch();
    IAuctionBase.EpochInfo memory epochInfo = spice.getEpochInfo(epoch);
    uint64 startTime = uint64(block.timestamp + config.startCooldown);
    uint64 endTime = uint64(startTime+config.duration);
    vm.expectEmit(address(spice));
    emit AuctionStarted(epoch+1, alice, startTime, endTime, 100 ether); // Notice that only 100 ether were counted for the next epoch
    spice.startAuction(); 
    vm.startPrank(daoExecutor);

    // attempting to recover those extra 100 ether after the auction ends.
    vm.warp(endTime);
    vm.expectRevert(abi.encodeWithSelector(InvalidParam.selector));
    spice.recoverToken(auctionToken, alice, 100 ether);

    // 100 ether of auction token are locked permanently in the contract
}
}
```

```bash
forge test --mt testPoC
[PASS] testPoC() (gas: 734490)
```

## Impact

If the last auction config at cooldown is removed, two serious impacts will occur:

1. `_totalAuctionTokenAllocation` will always contain extra auction tokens for all subsequent epochs.
2. Those extra auction tokens can not be recovered.

## Tools Used

Manual Review

## Recommendations

Consider updating `_totalAuctionTokenAllocation` when removing the config of the last started auction at the cooldown period. Below is a suggestion for an updated code of `SpiceAuction::removeAuctionConfig`:

```diff
function removeAuctionConfig() external override onlyDAOExecutor {
    /// only delete latest epoch if auction is not started
    uint256 id = _currentEpochId;
        
    EpochInfo storage info = epochs[id];
    // _currentEpochId = 0
    if (info.startTime == 0) { revert InvalidConfigOperation(); }
    // Cannot reset an ongoing auction
    if (info.isActive()) { revert InvalidConfigOperation(); }
    /// @dev could be that `auctionStart` is triggered but there's cooldown, which is not reached (so can delete epochInfo for _currentEpochId)
    // or `auctionStart` is not triggered but `auctionConfig` is set (where _currentEpochId is not updated yet)
    bool configSetButAuctionStartNotCalled = auctionConfigs[id+1].duration > 0;
    if (!configSetButAuctionStartNotCalled) {
        /// @dev unlikely because this is a DAO execution, but avoid deleting old ended auctions
        if (info.hasEnded()) { revert AuctionEnded(); }
        /// auction was started but cooldown has not passed yet
+        SpiceAuctionConfig storage config = auctionConfigs[id];
+        (,address auctionToken) = _getBidAndAuctionTokens(config);
+        _totalAuctionTokenAllocation[auctionToken] -= info.totalAuctionTokenAmount;
        delete auctionConfigs[id];
        delete epochs[id];
        _currentEpochId = id - 1;
        emit AuctionConfigRemoved(id, id);
    } else {
        // `auctionStart` is not triggered but `auctionConfig` is set
        id += 1;
        delete auctionConfigs[id];
        emit AuctionConfigRemoved(id, 0);
    }
}
```

## <a id='M-02'></a>M-02. Changes to vesting period is not handled inside `_getVestingRate` 

_Submitted by [gajiknownnothing](/profile/cluxdb9cf0000a5hu1yjybej7), [mrudenko](/profile/cllm9cr0f0008mi0811ow750m), [0x3b](/profile/clk3yiyaq002imf088cd3644k), [hyperhorse](/profile/clyd0a8es000ezbebc71je4yn), [adriro](/profile/cllkxto1y001yjx08kh87bqge), [1337web3](/profile/clw6sn30t0000mruqsj84nk6p), [0xhals](/profile/clkfub7qh0002l508bt5xdugv), [0xCiphky](/profile/clkrx20ym0000ju088s337njb), [jsmi](/profile/clxtywrn20006v8m6ioxq33r6), [cryptomoon](/profile/clo8a861c001emc092vef10q1). Selected submission by: [hyperhorse](/profile/clyd0a8es000ezbebc71je4yn)._      
            


## Summary

Changes to vesting period is not handled inside `_getVestingRate`

## Vulnerability Detail

The vesting period end time of a stake is fixated at the time of staking itself while the vesting period of the contract can change after an ongoing draw

[link](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/TempleGoldStaking.sol#L495-L498)

```solidity
    function _applyStake(address _for, uint256 _amount, uint256 _index) internal updateReward(_for, _index) {
        totalSupply += _amount;
        _balances[_for] += _amount;
=>      _stakeInfos[_for][_index] = StakeInfo(uint64(block.timestamp), uint64(block.timestamp + vestingPeriod), _amount);
```

In case the vesting period is reduced, this allows for scenario's where the calculated `vestingRate` of a stake can be greater than 1e18 ie. the ideal maximum value for `vestingRate` leading to inflated returns for the staker initially and a potential timeframe where the reward claiming could revert due to underflow caused by the previosuly stored inflated `userRewardPerTokenPaid`

[link](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/TempleGoldStaking.sol#L484)

```solidity
    function _getVestingRate(StakeInfo memory _stakeInfo) internal view returns (uint256 vestingRate) {
        if (_stakeInfo.stakeTime == 0) {
            return 0;
        }
        if (block.timestamp > _stakeInfo.fullyVestedAt) {
            vestingRate = 1e18;
        } else {
            vestingRate = (block.timestamp - _stakeInfo.stakeTime) * 1e18 / vestingPeriod;
        }
    }
```

In case the vesting period is increased, there will be a sudden drop in the `vestingRates` of the stakers whose vesting time has not ended and this will cause the reward claiming to revert until the vesting period fully finishes

### Example

Draw period == initial vesting period == 16weeks
User stakes at 15th week
For the stake, fullyVestedAt = 15week + 16weeks == 31 weeks
After draw ends, vesting period is reduced to 8 weeks
Now after 9 weeks the vesting rate of the stake will be,
(10 weeks/8 weeks) \* 1e18 > 1e18

### POC

Apply the follwing diff and run `forge test --mt testAudit_vesting_period_influence_inflated_reward`
The function `getVestingRate` is added to the `TempleGoldStaking` contract in order to access the vesting rate publicly

```diff
diff --git a/protocol/contracts/templegold/TempleGoldStaking.sol b/protocol/contracts/templegold/TempleGoldStaking.sol
index b2d95ae..4397f26 100644
--- a/protocol/contracts/templegold/TempleGoldStaking.sol
+++ b/protocol/contracts/templegold/TempleGoldStaking.sol
@@ -460,6 +460,9 @@ contract TempleGoldStaking is ITempleGoldStaking, TempleElevatedAccess, Pausable
         }
     }
 
+    function getVestingRate(StakeInfo memory _stakeInfo) public view returns (uint256 vestingRate) {
+        return _getVestingRate(_stakeInfo);
+    }
     function _earned(
         StakeInfo memory _stakeInfo,
         address _account,
diff --git a/protocol/test/forge/templegold/TempleGoldStaking.t.sol b/protocol/test/forge/templegold/TempleGoldStaking.t.sol
index 705e3d4..4330691 100644
--- a/protocol/test/forge/templegold/TempleGoldStaking.t.sol
+++ b/protocol/test/forge/templegold/TempleGoldStaking.t.sol
@@ -1111,6 +1111,52 @@ contract TempleGoldStakingTest is TempleGoldStakingTestBase {
         assertEq(staking.earned(alice, 2), 0);
     }
 
+    function testAudit_vesting_period_influence_inflated_reward() public {
+        uint32 _vestingPeriod = 16 weeks;
+        {
+            skip(3 days);
+            _setVestingPeriod(_vestingPeriod);
+            _setRewardDuration(_vestingPeriod);
+            _setVestingFactor(templeGold);
+        }
+
+        uint256 stakeAmount = 100 ether;
+        uint256 goldRewardsAmount;
+        ITempleGoldStaking.Reward memory rewardDataOne;
+        {
+            // initial distribution and draw
+            vm.startPrank(alice);
+            deal(address(templeToken), alice, 1000 ether, true);
+            deal(address(templeToken), bob, 1000 ether, true);
+            _approve(address(templeToken), address(staking), type(uint).max);
+            staking.stake(stakeAmount);
+            assertEq(staking.earned(alice, 1), 0);
+            staking.distributeRewards();
+        }
+
+        {
+            // 2nd stake, 1 week left for the draw end
+            skip(15 weeks);
+            staking.stake(stakeAmount);
+        }
+
+        {
+            // current draw finished
+            skip(2 weeks);
+            // new vesting period is set to half
+            _setVestingPeriod(_vestingPeriod / 2);
+            // new draw started
+            staking.distributeRewards();
+            // after 8 weeks, the vestingRate will return > 1e18 instead of 1e18 since still attached will the old vest end period
+            skip(9 weeks);
+            ITempleGoldStaking.StakeInfo memory _stakeInfo = staking.getAccountStakeInfo(alice, 2);
+            uint vestingRate = staking.getVestingRate(_stakeInfo);
+            // 1.375000000000000000 
+            // console.log("vestingRate",vestingRate);
+            assert(vestingRate > 1e18);
+        }
+    }
+
     function test_getReward_tgldStaking_multiple_stakes_multiple_rewards_distribution() public {
         uint32 _vestingPeriod = 16 weeks;
         {

```

## Impact

Incorrect reward allocation in case of vesting period changes causing loss of rewards for deserving stakers

## Tool used

Manual Review

## Recommendation

Limit the maximum to the 1e18 inside the `_getVestingRate` function


# Low Risk Findings

## <a id='L-01'></a>L-01. Incosistent message generation in TempleTeleporter.quote() and TempleTeleporter.teleport() results in inaccurate required fee calculation by TempleTeleporter.quote()

_Submitted by [mt030d](/profile/cly360v0o000014p4y4rxyufy), [ZdravkoHr](/profile/clkmey53n0018l008gwzuqcmu), [qpzm](/profile/cllu8b144000gjs08aolwd6rr), [0xPkhatri4](/profile/clybv63k100003ug6xetxgtd3), [0x3b](/profile/clk3yiyaq002imf088cd3644k), [hyperhorse](/profile/clyd0a8es000ezbebc71je4yn), [nnez](/profile/cly5nzz7a0004g0882z8nvedb), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [0xhuy0512](/profile/clk80hto50004m908dr18akv7), [pep7siup](/profile/clktaa8x50014mi08472cywse), [adeolu](/profile/cly76tbdq0000yyyrvl162rik), [boredpukar](/profile/clkarpfyo000cl90845grept2), [datamcrypto](/profile/clqzz1thz0006w5ld3aw1aigx), [unRekt](/team/clygqij4y001bgabeovyj6hg2), [0xsandy](/profile/clk43kus5009imb0830ko7dxy), [0xspryon](/profile/clo19fw280000mf08c4yazene), [ZukaNoPro](/profile/clvhorw4z000049xga7gskpfr). Selected submission by: [adeolu](/profile/cly76tbdq0000yyyrvl162rik)._      
            


## Summary

`TempleTeleporter.quote()` will return a fee value lower than will be required for sending a message via `TempleTeleporter.teleport()`. This is because the way both functions construct the layerzero message/payload is different. `TempleTeleporter.quote()` uses `abi.encodePacked(_to, _amount)` while `TempleTeleporter.teleport()` uses `abi.encodePacked(to.addressToBytes32(), amount)`.

`TempleTeleporter.quote()` is a function used for getting the fee amount needed for sending a layerZero message via `TempleTeleporter.teleport()`. `TempleTeleporter.quote()` does not calculate properly/underestimates the fee amount because of this difference mentioned above.

## Vulnerability Details

The conversion of the recipient address `to` into bytes, as implemented in [TempleTeleporter.teleport()](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/TempleTeleporter.sol#L52), results in a higher messaging fee for a sucessfull teleport() call. This occurs because the message or payload generated in `TempleTeleporter.teleport()` is longer due to the address conversion to bytes before encoding. In contrast, TempleTeleporter.quote() does not convert the address to bytes before encoding, resulting in a shorter message and thus a lower message fee is calculated. Because of thie `quote()` will always return an underestimated value required for a sucessfull `teleport()` call.

<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/TempleTeleporter.sol#L43-L52>

```solidity
    function teleport(
        uint32 dstEid,
        address to,
        uint256 amount,
        bytes calldata options
    ) external payable override returns(MessagingReceipt memory receipt) {
        if (amount == 0) { revert CommonEventsAndErrors.ExpectedNonZero(); }
        if (to == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
        // Encodes the message before invoking _lzSend.
        bytes memory _payload = abi.encodePacked(to.addressToBytes32(), amount);
```

<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/TempleTeleporter.sol#L87C1-L93C81>

```solidity
    function quote(
        uint32 _dstEid,
        address _to,
        uint256 _amount,
        bytes memory _options
    ) external view returns (MessagingFee memory fee) {
        return _quote(_dstEid, abi.encodePacked(_to, _amount), _options, false);
```

#

The layerzero docs advise that the arguments passed into the `quote()` function identically match the parameters used in the `lzSend()` function to call `_lzSend()` because If parameters mismatch, you  can run into errors as your `msg.value` will not match the actual gas quote. --> <https://docs.layerzero.network/v2/developers/evm/oapp/overview#estimating-gas-fees>



## Proof Of Concept

The script below shows the differences fee estimation via` LZendpoint.quote()` which is eventually called by the OApp's internal `_quote()` function. It compares the fees between messages created with address as address before encoding and address converted to bytes before encoding.

* paste code below in new file created in `./protocol/script/` folder
* run code with `forge script --chain mainnet script/Counter.s.sol:QuoteTestScript --rpc-url **YOUR_RPC_URL** --broadcast -vv`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from 'forge-std/Script.sol';
import {MessagingParams, MessagingFee, MessagingReceipt, ILayerZeroEndpointV2} from '@layerzerolabs/lz-evm-protocol-v2/contracts/interfaces/ILayerZeroEndpointV2.sol';
import {OAppOptionsType3} from '@layerzerolabs/lz-evm-oapp-v2/contracts/oapp/libs/OAppOptionsType3.sol';
import {OptionsBuilder} from '@layerzerolabs/lz-evm-oapp-v2/contracts/oapp/libs/OptionsBuilder.sol';

contract QuoteTestScript is Script {
  using OptionsBuilder for bytes;

  function setUp() public {}

  function run() public {
    vm.startBroadcast();
    teleport_quote();

    vm.stopBroadcast();
  }

  function _addressToBytes32(address _addr) internal pure returns (bytes32) {
    return bytes32(uint256(uint160(_addr)));
  }

  function teleport_quote() public {
    //aim is to check if fee value returned when message is abi.encodePacked(_to, _amount) as seen in templeTeleporter.quote() is same for when message
    // is abi.encodePacked(to.addressToBytes32(), amount) which is used in templeTeleporter.teleport()

    //address of layerZero endpoint on ethereum mainnet
    address endpoint = 0x1a44076050125825900e736c501f859c50fE728c;
    uint amount = 100;
    address to = makeAddr("to");
    uint32 dstEid = 30102; //LZ destination endpoint ID

    bytes memory options = OptionsBuilder
      .newOptions()
      .addExecutorLzReceiveOption(200000, 0); //build options

    //build two different message params, one with address as bytes in message, the other with plain address in message
    MessagingParams memory messageParamsWithAddressAsBytes = MessagingParams({
      dstEid: dstEid, //endpoint id for avalanche
      message: abi.encodePacked(_addressToBytes32(to), amount),
      options: options,
      payInLzToken: false,
      receiver: _addressToBytes32(makeAddr("receiver"))
    });

    MessagingParams memory messageParamsWithAddressAsAddress = MessagingParams({
      dstEid: dstEid, //endpoint id for avalanche
      message: abi.encodePacked(to, amount),
      options: options,
      payInLzToken: false,
      receiver: _addressToBytes32(makeAddr("receiver"))
    });

    address sender = makeAddr("sender");

    //call endpoint with the two different message params,
    //only diff between the two is the address as address in one and address as bytes in the other
    MessagingFee memory fee1 = ILayerZeroEndpointV2(endpoint).quote(
      messageParamsWithAddressAsBytes,
      sender
    );
    MessagingFee memory fee2 = ILayerZeroEndpointV2(endpoint).quote(
      messageParamsWithAddressAsAddress,
      sender
    );

    require(fee1.nativeFee != fee2.nativeFee, "same value returned");

    console.log("fee when address is converted to bytes: ", fee1.nativeFee);
    console.log("fee when address is not converted to bytes: ", fee2.nativeFee);
    console.log("fee charged when address is not converted to bytes is lesser");
    console.log(
      "the templeTeleporter.quote() fcn will return a smaller fee value than required by templeTeleporter.teleport() "
    );

    require(fee1.nativeFee > fee2.nativeFee);
  }
}
```

## Impact 

`TempleTeleporter.quote()`does not return the correct fee amount require for a layerZero message via `TempleTeleporter.teleport()`.

## Tools Used

manual review , foundry

## Recommendations

ensure consistency between both functions, use same methods for generating the layerzero payload/message in both functions.

## <a id='L-02'></a>L-02. Lack of Comprehensive Pausability for Critical Functions

_Submitted by [arnie](/profile/clk4gbnc30000mh088nl2a5i4), [tpiliposian](/profile/clnwsmii60000jq08rqe1buzt), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [Triad of Trust](/team/clygb6qvg000114i7yujtn68m), [maushishreal](/profile/cly7qmvhe0000121icxq7juzd), [Nave765](/profile/clu2razgi0009adkkckhfloax), [ZukaNoPro](/profile/clvhorw4z000049xga7gskpfr). Selected submission by: [tpiliposian](/profile/clnwsmii60000jq08rqe1buzt)._      
            


## Summary

The `TempleGoldStaking.sol` contract features a `whenNotPaused` modifier that is currently only applied to the `stakeFor()` function. This limited application of the pausing functionality could leave critical operations exposed during emergencies, potentially jeopardizing the safety of staked assets and reward distributions.

## Vulnerability Details

The `whenNotPaused` modifier is used only in the `stakeFor()` function, which allows users to stake tokens:

```solidity
    function stakeFor(address _for, uint256 _amount) public whenNotPaused {
        if (_amount == 0) revert CommonEventsAndErrors.ExpectedNonZero();
        
        // pull tokens and apply stake
        stakingToken.safeTransferFrom(msg.sender, address(this), _amount);
        uint256 _lastIndex = _accountLastStakeIndex[_for];
        _accountLastStakeIndex[_for] = ++_lastIndex;
        _applyStake(_for, _amount, _lastIndex);
        _moveDelegates(address(0), delegates[_for], _amount);
    }
```

`withdraw()` and `withdrawAll()` functions that allow users to withdraw staked tokens and claim rewards are not protected by the pausing mechanism.

`getReward()`function facilitates the claiming of rewards and is not covered by the pausing control.

`distributeRewards()`and`distributeGold()` reward distribution functions are also not pausable.

## Impact

The lack of comprehensive pause functionality exposes the contract to potential issues if the contract needs to be paused for maintenance or in response to an attack. By not restricting all non-migration functions during a pause, users can still interact with the contract in ways that may not be intended during a paused state, i.e. if an emergency occurs (e.g., a security vulnerability is discovered), the contract cannot be fully paused to protect funds and prevent unauthorized transactions. This could lead to:

Unauthorized withdrawals and claims of rewards during a security breach.

Potential loss of staked tokens and rewards if a vulnerability is exploited before a fix can be applied.

Increased risk to user assets, as pausing is a common safeguard to mitigate damage during incidents.

## Tools Used

Manual review.

## Recommendations

Apply `whenNotPaused` modifier to mentioned critical functions.

## <a id='L-03'></a>L-03. Auction tokens cannot be recovered for the first ever spice auction

_Submitted by [Meeve](/profile/cly4oayy40000n9c7efbaa18j), [cryptomoon](/profile/clo8a861c001emc092vef10q1), [0xbrivan2](/profile/clp1dquol0000kupd1wulucuy), [Fortis Audits](/team/cly2eas2s000gpgez427qfcw9), [touthang](/profile/clk9bd0tr0000mi084i1py0rx), [GEEK](/profile/clk48fvzr003wla08us7vyyv4), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [Trident Audits](/team/clyfyc6wf000pr98ax0arts8w), [kildren](/profile/clxmmb9g7000010aw9we896mw), [Triad of Trust](/team/clygb6qvg000114i7yujtn68m), [recursiveEth](/profile/cln15dz4e0000l208u4l7dx6k), [0xcontrol](/profile/clygxkhm10008f564k5hqdhy7), [binary](/profile/cly46keh4000087tn5a4feozf). Selected submission by: [Trident Audits](/team/clyfyc6wf000pr98ax0arts8w)._      
            


## Summary

The `SpiceAuction::recoverToken()` function is used to recover the auction tokens (either temple gold or a spice token) for an auction whose config is set, but hasn't started yet. However, for auction id 1 (and epoch id 0, since auction hasn't started yet), tokens can never be recovered if the dao wanted to do so.

> **NOTE**
> Auction id isn't a state variable. It is used to better represent the latest auction number. Auction config is set for the `_currentEpochId` + 1 index, and the `_currentEpochId` variable lags behind the auction id by 1 until the auction starts. Then the `_currentEpochId` is incremented, and epoch info is set.

## Vulnerability Details

Consider the following code segment from `SpiceAuction::recoverToken()` function,

```Solidity
    function recoverToken(address token, address to, uint256 amount) external override onlyDAOExecutor {
        // ...

        uint256 epochId = _currentEpochId;
@>      EpochInfo storage info = epochs[epochId];
        /// @dev use `removeAuctionConfig` for case where `auctionStart` is called and cooldown is still pending
@>      if (info.startTime == 0) revert InvalidConfigOperation();
        if (!info.hasEnded() && auctionConfigs[epochId + 1].duration == 0) revert RemoveAuctionConfig();

        // ...
    }
```

For the first spice auction which hasn't started yet (auction id 1 and epoch id 0), the epoch info hasn't been set (it is set once the `SpiceAuction::startAuction()` function is called). Thus, [L#249](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L249) accesses a non-existent epoch info (all fields in the struct hold value 0) and the function reverts at [L#251](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L251).

Additionally, auction tokens cannot be recovered during the cooldown period. This will revert at [L#252](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L252).

## Impact

The dao executor won't be able to recover tokens for the first ever spice auction. This is essentially a DOS type of vulnerability.

## Proof Of Concept

Add the following test to `SpiceAuctionTest` contract in `./protocol/test/forge/templegold/SpiceAuction.t.sol`,

```solidity
    function testRecoverTokenFailsForFirstAuction() public {
        ISpiceAuction.SpiceAuctionConfig memory config = _getAuctionConfig();
        vm.startPrank(daoExecutor);
        spice.setAuctionConfig(config);
        vm.stopPrank();

        address auctionToken = config.isTempleGoldAuctionToken ? address(templeGold) : spice.spiceToken();
        dealAdditional(IERC20(auctionToken), address(spice), 100 ether);

        vm.prank(daoExecutor);
        vm.expectRevert(ISpiceAuction.InvalidConfigOperation.selector);
        spice.recoverToken(auctionToken, daoExecutor, 100 ether);
    }
```

The test passes, with the following logs,

```shell
Ran 1 test for test/SpiceAuction.t.sol:SpiceAuctionTest
[PASS] testRecoverTokenFailsForFirstAuction() (gas: 396290)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.06s (2.92ms CPU time)

Ran 1 test suite in 2.08s (2.06s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual review and Foundry for writing POC and tests.

## Recommended Mitigation

Make the following changes to `SpiceAuction::recoverToken()` functions,

```diff
    function recoverToken(address token, address to, uint256 amount) external override onlyDAOExecutor {
        // ...

        uint256 epochId = _currentEpochId;
        EpochInfo storage info = epochs[epochId];
        /// @dev use `removeAuctionConfig` for case where `auctionStart` is called and cooldown is still pending
-       if (info.startTime == 0) { revert InvalidConfigOperation(); }
-       if (!info.hasEnded() && auctionConfigs[epochId+1].duration == 0) { revert RemoveAuctionConfig(); }
+       if (epochId != 0) {
+           if (info.startTime == 0) revert InvalidConfigOperation();
+           if (!info.hasEnded() && auctionConfigs[epochId + 1].duration == 0) revert RemoveAuctionConfig();
+       }

        // ...
    }
```

Now let's unit test the changes,

```solidity
    function testRecoverTokenForFirstAuction() public {
        ISpiceAuction.SpiceAuctionConfig memory config = _getAuctionConfig();
        vm.startPrank(daoExecutor);
        spice.setAuctionConfig(config);
        vm.stopPrank();

        address auctionToken = config.isTempleGoldAuctionToken ? address(templeGold) : spice.spiceToken();
        dealAdditional(IERC20(auctionToken), address(spice), 100 ether);

        vm.prank(daoExecutor);
        spice.recoverToken(auctionToken, daoExecutor, 100 ether);

        assertEq(IERC20(auctionToken).balanceOf(daoExecutor), 100 ether);
    }
```

The test passes with the following logs,

```shell
Ran 1 test for test/SpiceAuction.t.sol:SpiceAuctionTest
[PASS] testRecoverTokenForFirstAuction() (gas: 424640)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.59s (3.46ms CPU time)

Ran 1 test suite in 1.61s (1.59s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## <a id='L-04'></a>L-04. TempleGold tokens cannot be recovered when a `DaiGoldAuction` ends with 0 bids

_Submitted by [0x3b](/profile/clk3yiyaq002imf088cd3644k), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [62theories](/profile/cly85wthq0000vq4looxvii2u), [cryptomoon](/profile/clo8a861c001emc092vef10q1), [0xbrivan2](/profile/clp1dquol0000kupd1wulucuy), [ke1caM](/profile/clk46fjfm0014la08xl7mwtis), [nnez](/profile/cly5nzz7a0004g0882z8nvedb), [nisedo](/profile/clk3saar60000l608gsamuvnw), [merlinboii](/profile/clnxnj1ow000ll008rx7zrb8h), [jacopod](/profile/cllj5m9dx0000mh08b60qr0zn), [matej](/profile/clqf9x2s3000r83pidasj858e), [adriro](/profile/cllkxto1y001yjx08kh87bqge), [0xCiphky](/profile/clkrx20ym0000ju088s337njb), [Aravn](/profile/cly5eu9km0005k6tyikt9diwz), [billoBaggeBilleyan](/profile/clp81a9bn0000iaa0sgjkbtai), [Trident Audits](/team/clyfyc6wf000pr98ax0arts8w), [Tripathi](/profile/clk3xe9tk0024l808xjc9wkg4), [dinkras](/profile/cly6zi83d000611y4odwhusk4), [Myrault](/profile/cls49aoq3000cw28oit3j6zj3). Selected submission by: [Trident Audits](/team/clyfyc6wf000pr98ax0arts8w)._      
            


## Summary

When an auction ends with 0 bids, the `totalAuctionTokenAmount` value of TGLD tokens remain locked forever as `DaiGoldAuction::recoverToken()` function can only be used when the auction is still in its cooldown period.

## Vulnerability Details

`DaiGoldAuction::recoverToken` provides a mechanism to recover tokens that have been distributed and allotted to a specific auction only **BEFORE** the auction starts, that is during the cooldown period as evident by the code below,

```Solidity
function recoverToken(
    address token,
    address to,
    uint256 amount
) external override onlyElevatedAccess {
    // code

    // auction started but cooldown pending
    uint256 epochId = _currentEpochId;
    EpochInfo storage info = epochs[epochId];
    if (info.startTime == 0) { revert InvalidOperation(); }
@>  if (info.isActive()) { revert AuctionActive(); }
@>  if (info.hasEnded()) { revert AuctionEnded(); }

    // code
}
```

When users bid in an auction, they are entitled to their claims, even if there's just one bid, that bidder deserves the entire pot. Users are also allowed to claim tokens from a previous auction as they please, hence we need not worry about tokens remaining stuck in the contract.

However this is not the case when an auction ends with no bids, as in such a situation we would want to recover those tokens as there are no claims. This is prevented by the fact that tokens cannot be recovered after an auction ends.

## Impact

`totalAuctionTokenAmount` value worth of tokens allotted for the auction would remain permanently stuck inside the contract. If this scenario repeats multiple times, a significant amount of tokens would be cut out from circulation.

## Proof of Code

1. Add the following test to the existing `DaiGoldAuction.t.sol` file,

```Solidity
function test_CannotRecover_AuctionTokens_WhenZeroBids() public{
    // set the config and startAuction
    IDaiGoldAuction.AuctionConfig memory config = _getAuctionConfig();
    vm.startPrank(executor);
    daiGoldAuction.setAuctionConfig(config);
    skip(1 days); // give enough time for TGLD emissions
    daiGoldAuction.startAuction();

    /*
        THE AUCTION ENDS WITHOUT ANY BIDS
    */

    uint256 currentEpoch = daiGoldAuction.currentEpoch();
    IAuctionBase.EpochInfo memory info1 = daiGoldAuction.getEpochInfo(currentEpoch);
    vm.warp(info1.endTime);
    uint256 recoveryAmount1 = info1.totalAuctionTokenAmount;

    // unable to recover tokens
    vm.expectRevert(abi.encodeWithSelector(IAuctionBase.AuctionEnded.selector));
    daiGoldAuction.recoverToken(address(templeGold), alice, recoveryAmount1);

    // tokens not included in the next auction either
    assertEq(daiGoldAuction.nextAuctionGoldAmount(), 0);

    // next auction starts and is in cooldown period
    vm.warp(info1.endTime + config.auctionsTimeDiff);
    daiGoldAuction.startAuction();
    currentEpoch = daiGoldAuction.currentEpoch();
    IAuctionBase.EpochInfo memory info2 = daiGoldAuction.getEpochInfo(currentEpoch);
    uint256 recoveryAmount2 = info2.totalAuctionTokenAmount;

    // let's try to recover tokens from previous auction along with current auction
    uint256 totalRecoveryAmount = recoveryAmount1 + recoveryAmount2;
    vm.expectRevert(abi.encodeWithSelector(CommonEventsAndErrors.InvalidAmount.selector, address(templeGold), totalRecoveryAmount));
    daiGoldAuction.recoverToken(address(templeGold), alice, totalRecoveryAmount);

    // assuming that distribution hasn't happened in the meanwhile
    assertEq(templeGold.balanceOf(address(daiGoldAuction)), totalRecoveryAmount);
}
```

1. Also update the vesting period params in `DaiGoldAuction.t.sol::_configureTempleGold` as follows,

```diff
function _configureTempleGold() private {
    ITempleGold.DistributionParams memory params;
    params.escrow = 60 ether;
    params.gnosis = 10 ether;
    params.staking = 30 ether;
    templeGold.setDistributionParams(params);
    ITempleGold.VestingFactor memory factor;
-   factor.numerator = 2 ether;
+   factor.numerator = 1;
-   factor.denominator = 1000 ether;
+   factor.denominator = 162 weeks; // 3 years
    templeGold.setVestingFactor(factor);
    templeGold.setStaking(address(goldStaking));
    // whitelist
    templeGold.authorizeContract(address(daiGoldAuction), true);
    templeGold.authorizeContract(address(goldStaking), true);
    templeGold.authorizeContract(teamGnosis, true);
}
```

1. Run `forge test --mt test_CannotRecover_AuctionTokens_WhenZeroBids -vv`

The test passes successfully with the following logs,

```shell
Ran 1 test for test/forge/templegold/DaiGoldAuction.t.sol:DaiGoldAuctionTest
[PASS] test_CannotRecover_AuctionTokens_WhenZeroBids() (gas: 405414)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 475.59ms (2.81ms CPU time)

Ran 1 test suite in 483.13ms (475.59ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendations

Preferable mitigation would be to add another function to `DaiGoldAuction` for specifically recovering tokens when an auction ends with 0 bids as shown below:

```diff
+ /**
+  * @notice Recover auction tokens for epoch with zero bids
+  * @param epochId Epoch Id
+  * @param to Recipient
+  */
+ function recoverAuctionTokenForZeroBidAuction(uint256 epochId, address to) external onlyElevatedAccess {
+     if (to == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
+     // has to be valid epoch
+     if (epochId > _currentEpochId) { revert InvalidEpoch(); }
+     // epoch has to be ended
+     EpochInfo storage info = epochs[epochId];
+     if (!info.hasEnded()) { revert AuctionActive(); }
+     // bid token amount for epoch has to be 0
+     if (info.totalBidTokenAmount > 0) { revert InvalidOperation(); }

+     uint256 amount = info.totalAuctionTokenAmount;
+     emit CommonEventsAndErrors.TokenRecovered(to, address(templeGold), amount);
+     templeGold.safeTransfer(to, amount);
+ }
```

Above change can be verified against the following uint test,

```Solidity
function test_MitigationSuccessful() public {
    // set the config and startAuction
    IDaiGoldAuction.AuctionConfig memory config = _getAuctionConfig();
    vm.startPrank(executor);
    daiGoldAuction.setAuctionConfig(config);
    skip(1 days); // give enough time for TGLD emissions
    daiGoldAuction.startAuction();

    /*
        THE AUCTION ENDS WITHOUT ANY BIDS
    */

    uint256 currentEpoch = daiGoldAuction.currentEpoch();
    IAuctionBase.EpochInfo memory info = daiGoldAuction.getEpochInfo(currentEpoch);
    vm.warp(info.endTime);
    uint256 recoveryAmount = info.totalAuctionTokenAmount;

    vm.expectEmit(address(daiGoldAuction));
    emit TokenRecovered(alice, address(templeGold), recoveryAmount);
    daiGoldAuction.recoverAuctionTokenForZeroBidAuction(currentEpoch, alice);

    // assuming alice has no prior token balance
    assertEq(templeGold.balanceOf(alice), recoveryAmount);
}
```

The test passes successfully with the following logs,

```shell
Ran 1 test for test/forge/templegold/DaiGoldAuction.t.sol:DaiGoldAuctionTest
[PASS] test_MitigationSuccessful() (gas: 329745)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 455.58ms (3.28ms CPU time)

Ran 1 test suite in 458.30ms (455.58ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Another way would be to update `recoverToken` to also account for 0 bid auctions but it is better to have another function for the sake of Separation of Concerns.

## Tools Used

Manual Review and Foundry for POC.

## <a id='L-05'></a>L-05. Malicious user can prevent  `rewardData.perodfinish` from ending by calling `TempleGoldStaking::distributeRewards()` before the end of the reward duration when no starter is set.

_Submitted by [nisedo](/profile/clk3saar60000l608gsamuvnw), [Shalafat](/profile/clvz9swar0000ck6v8rf2uxh1), [SBSecurity](/team/clkuz8xt7001vl608nphmevro), [jesjupyter](/profile/clxvvtui0000310nimqwnkuvi), [dinkras](/profile/cly6zi83d000611y4odwhusk4). Selected submission by: [Shalafat](/profile/clvz9swar0000ck6v8rf2uxh1)._      
            


## Summary

Calling `TempleGoldStaking::distributeRewards()` upadates the `rewwardData.periodFinish` paramater which specificies when reward distribution is supposed to stop however when a starter is not set, a malicious user can call this function towards the end of a reward duration which will renew the `rewardData.periodFinish` paramater making the reward period go beyond the intended time.

## Vulnarebility Details

In the `TempleGoldStaking` contract it is possible for rewardDistribution to be started by anyone when a starter is not set.

```Solidity
* @dev If starter is address zero, anyone can call `distributeRewards` to apply and
   * distribute rewards for next reward duration
```

A malitious user a can recall this function towards the end of a reward duration which will renew the `rewwardData.periodFinish` when `TempleGoldStaking::_notifyReward` is called.

```Solidity
 function _notifyReward(uint256 amount) private {
    if (block.timestamp >= rewardData.periodFinish) {
      rewardData.rewardRate = uint216(amount / rewardDuration);
      // collect dust
      nextRewardAmount = amount - (rewardData.rewardRate * rewardDuration);
    } else {
      uint256 remaining = uint256(rewardData.periodFinish) - block.timestamp;
      uint256 leftover = remaining * rewardData.rewardRate;
      rewardData.rewardRate = uint216((amount + leftover) / rewardDuration);
      // collect dust
      nextRewardAmount =
        (amount + leftover) -
        (rewardData.rewardRate * rewardDuration);
    }
   rewardData.lastUpdateTime = uint40(block.timestamp);
   @@>>  rewardData.periodFinish = uint40(block.timestamp + rewardDuration);
  }
```

The `rewardData.periodFinish` paramater is renewed when the above function is called making the reward period go beyond the intended time.

 ## POC
  - We have user A and B
  User A is the reward distribution starter but has not been set in the contract.

 -  User A starts the Reward Distribution for a duration that has been set.
 
 - Towards the end of this reward distribution duration, User B maliciously calls `TempleGoldStaking::distributeRewards()` just to prevent the reward distribution from stopping.

  - This would go against user A's intentions as he may have wanted to add a reward duration and vesting period to be added. Which is now impossible due to the renewed `rewardData.periodfinish`. Even more gold will be distributrd than intended for this specific reward duration.


## Impact

This makes the reward distribution to go beyond the intended time and makes it impossible to set a new reward duration and vesting period.

##Tools
Manual Review

## Recommendation

Make it impossible for anyone to start Rewards distribution by ensuring only a single starter is supposed to do so. 



* <https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/TempleGoldStaking.sol#L515>

## <a id='L-06'></a>L-06. Incorrect templeGold minting due to unresolved accumulation in `TempleGold::setVestingFactor`

_Submitted by [ZdravkoHr](/profile/clkmey53n0018l008gwzuqcmu), [jacopod](/profile/cllj5m9dx0000mh08b60qr0zn), [jsmi](/profile/clxtywrn20006v8m6ioxq33r6), [cryptomoon](/profile/clo8a861c001emc092vef10q1), [pep7siup](/profile/clktaa8x50014mi08472cywse), [kildren](/profile/clxmmb9g7000010aw9we896mw). Selected submission by: [kildren](/profile/clxmmb9g7000010aw9we896mw)._      
            


### Relevant GitHub Links

<https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/TempleGold.sol#L130>
<https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/TempleGold.sol#L144>

## Summary

`templeGold` is minted at a rate strictly following the `VestingFactor`. However, when the operators call `TempleGold::setVestingFactor` to modify the `VestingFactor`, the tokens accumulated based on the previous factor are not resolved. This miscommunication results in an incorrect calculation of mint volumes, as the accumulated time is multiplied by the new `VestingFactor`, making the emission of `templeGold` unpredictable.

## Vulnerability Details

At [Line 130](https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/TempleGold.sol#L130), the `VestingFactor` is updated directly without resolving the tokens to be minted based on the accumulated time and the previous `VestingFactor`. This causes the accumulated time to be incorrectly calculated with the new `VestingFactor`, leading to discrepancies between expected and actual minting amounts.

```solidity
    function setVestingFactor(VestingFactor calldata _factor) external override onlyOwner {
        if (_factor.numerator == 0 || _factor.denominator == 0) { revert CommonEventsAndErrors.ExpectedNonZero(); }
        if (_factor.numerator > _factor.denominator) { revert CommonEventsAndErrors.InvalidParam(); }
@>        vestingFactor = _factor;
        /// @dev initialize
        if (lastMintTimestamp == 0) { lastMintTimestamp = uint32(block.timestamp); }
        emit VestingFactorSet(_factor.numerator, _factor.denominator);
    }
```

### POC

We will compare the actual minting quantity of the protocol with the expected minting quantity:

1. On the first day, minting follows the normal vesting factor rate.
2. On the second day, the vesting factor is doubled.&#x20;
3. Expected behavior is that the first day follows the old rate and the second day follows the doubled rate, making the total equivalent to three days at the normal rate.&#x20;
4. However, the protocol will incorrectly assume both days are minted at the doubled rate.

Place the following test into `TempleGold.t.sol::TempleGoldTest`:

```Solidity
    function test_setVestingFactorUnresolved_tgld() public {
        vm.startPrank(executor);
        ITempleGold.VestingFactor memory _factor = _getVestingFactor();
        // The _getVestingFactor() sets the rate too fast, distributing everything within ten time units.
        _factor.numerator = 1 ether;
        _factor.denominator = 100_000_000_000 ether;

        templeGold.setVestingFactor(_factor);
        // pass one day
        uint256 mintTime = block.timestamp + 1 days;
        vm.warp(mintTime);
        uint256 one_day_mint_amount = templeGold.getMintAmount();
        emit log_named_uint("mintAmount: ", templeGold.getMintAmount());

        _factor.numerator *= 2;
        templeGold.setVestingFactor(_factor);
        // pass one day again
        mintTime += 1 days;
        vm.warp(mintTime);
        uint256 final_mint_amount = templeGold.getMintAmount();

        // expected:
        // first day:  1 day * _factor
        // second day: 1 day * 2 * _factor
        // final : first day + second day = 3 * one_day_mint_amount
        assertGt(final_mint_amount, 3 * one_day_mint_amount);
        // actual:
        // final : 2 day * 2 * _factor  = 4 * one_day_mint_amount
        assertEq(final_mint_amount, 4 * one_day_mint_amount);
    }
```

## Impact

This issue can lead to significant fluctuations in the issuance rate of `templeGold`, causing unpredictable minting patterns that can destabilize the protocol and undermine user confidence.

## Tools Used

Manual Review.

## Recommendations

**Resolve Pending Mints Before Updating `VestingFactor`**: Call `mint()` to resolve any tokens accumulated under the previous vesting factor before updating to a new `VestingFactor`. This ensures that the calculation for the new vesting period starts accurately.

```diff
    function setVestingFactor(VestingFactor calldata _factor) external override onlyOwner {
        if (_factor.numerator == 0 || _factor.denominator == 0) { revert CommonEventsAndErrors.ExpectedNonZero(); }
        if (_factor.numerator > _factor.denominator) { revert CommonEventsAndErrors.InvalidParam(); }
+      if (lastMintTimestamp == 0) { lastMintTimestamp = uint32(block.timestamp); }
+      else { mint() }
        vestingFactor = _factor;
        /// @dev initialize
 -      if (lastMintTimestamp == 0) { lastMintTimestamp = uint32(block.timestamp); }
        emit VestingFactorSet(_factor.numerator, _factor.denominator);
    }
```

(Optional) Restrict VestingFactor Changes: Implement constraints on how significantly the VestingFactor can be changed relative to the previous factor and impose time-based restrictions on how frequently it can be updated. These measures will enhance the predictability and stability of templeGold emission.

By resolving any accumulated tokens before setting the new VestingFactor and by restricting the frequency and extent of VestingFactor changes, the protocol can maintain a stable and predictable minting process, thereby preserving the stability and user trust of the templeGold system.





    