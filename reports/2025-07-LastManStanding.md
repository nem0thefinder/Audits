# Findings Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](https://github.com/nem0thefinder/Audits/new/main/reports#h-01-legitimate-winners-can-be-dethroned-after-grace-period)|Legitimate Winners Can Be Dethroned After Grace Period |HIGH|
|[H-02](https://github.com/nem0thefinder/Audits/new/main/reports#h-02-claim-fee-calculation-based-on-minimum-required-instead-of-actual-payment)|Claim Fee Calculation Based on Minimum Required Instead of Actual Payment|HIGH|
|[M-01](https://github.com/nem0thefinder/Audits/new/main/reports#m-01-critical-comparison-operator-error-in-claimthrone)|Critical comparison operator Error in `claimThrone` |MEDIUM|
|[M-02](https://github.com/nem0thefinder/Audits/new/main/reports#m-02-previous-king-payout-mechanism-completely-missing)|Previous King Payout Mechanism Completely Missing|MEDIUM|

## 
## [H-01] Legitimate Winners Can Be Dethroned After Grace Period 
### Summary

The `claimThrone()` function allows new throne claims even after the `gracePeriod` has expired, enabling attackers to steal victories from legitimate winners. Players who have rightfully won the game (by surviving the grace period) can still be dethroned by subsequent claimants, completely breaking the core game mechanics and allowing theft of winnings.

## Description

The vulnerability exists because `claimThrone()` only checks the `gameNotEnded` modifier but fails to validate whether the `gracePeriod` has expired for the current king. This creates a critical window between `gracePeriod` expiration and declaring winner where legitimate winners can be robbed of their victory.

**Game Rules (as intended):**

1. Player becomes king by calling `claimThrone()`
2. If no one dethrones them within the `gracePeriod`, they win
3. Winner gets the entire pot when `declareWinner()` is called

**Current Bug:**

* Even after `gracePeriod` expires, new players can still call `claimThrone()`

* This steals the victory from the player who legitimately survived the grace period

* The attacker becomes the new king and can win the pot instead

```Solidity
function claimThrone() external payable gameNotEnded nonReentrant {
    // @Missing validation: should check if grace period expired
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
    // @rest of the logic
}
```

## Proof Of Concept

```solidity
function test_legitimate_winner_can_be_dethroned() public {
    // STEP 1: Player1 becomes king legitimately
    vm.startPrank(player1);
    game.claimThrone{value: INITIAL_CLAIM_FEE}();
    vm.stopPrank();
    
    assertEq(game.currentKing(), player1);
    uint256 winningTimestamp = block.timestamp;
    
    // STEP 2: Grace period expires - Player1 is now the legitimate winner
    vm.warp(winningTimestamp + game.gracePeriod() + 1 seconds);
    
    // At this point, player1 has WON the game according to the rules
    // No one should be able to dethrone them anymore
    
    // STEP 3: Player2 exploits the vulnerability and steals the victory
    vm.startPrank(player2);
    uint256 newFee = INITIAL_CLAIM_FEE + (INITIAL_CLAIM_FEE * game.feeIncreasePercentage()) / 100;
    
    // This should REVERT but currently SUCCEEDS
    game.claimThrone{value: newFee}();
    vm.stopPrank();
    
    // STEP 4: Player2 has stolen the crown from the legitimate winner
    assertEq(game.currentKing(), player2, "Legitimate winner was dethroned!");
    
    // STEP 5: Player2 now waits for their grace period and wins
    vm.warp(block.timestamp + game.gracePeriod() + 1 seconds);
    
    uint256 potAmount = game.pot();
    game.declareWinner();
    
    // STEP 6: Player2 gets the rewards that Player1 rightfully earned
    assertEq(game.pendingWinnings(player2), potAmount, "Attacker won illegitimately");
    assertEq(game.pendingWinnings(player1), 0, "Legitimate winner got nothing");
    
    // Player1 lost their rightful victory and all associated rewards
}
```

## Impact

* **Theft of Legitimate Winnings**: Players who rightfully won by surviving the grace period lose their victory and pot rewards

* **Complete Rule Breakdown**: The fundamental "grace period = win condition" rule is completely broken

## Mitigation

* Add grace period validation to prevent claiming throne after grace Period expiration:

```diff
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
    
    // CRITICAL FIX: Prevent throne claims after grace period expires
+    if (currentKing != address(0)) {
+       require(
+            block.timestamp <= lastClaimTime + gracePeriod,
+            "Game: Grace period has expired. Current king has won."
+        );
+    }
    
    // Rest of logic
}
```
## [H-02] Claim Fee Calculation Based on Minimum Required Instead of Actual Payment
### Summary

The `claimThrone()` function calculates the next claim fee based on the minimum required fee rather than the actual amount sent by the player. This creates an economic vulnerability where players can invest large amounts  but the next challenger only needs to pay a small increase based on the original minimum fee, making strategic overpayment worthless and breaking the core "Last Man Standing" game mechanics.

## Description

The vulnerability exists in the fee calculation logic within [`claimThrone()`](https://github.com/CodeHawks-Contests/2025-07-last-man-standing/blob/47d9d19a78acb52270269f4bff1568b87eb81a96/src/Game.sol#L215)

```solidity
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    
    uint256 sentAmount = msg.value; // Could be much higher than claimFee
    
    // ... processing logic uses sentAmount ...
    
    // BUG: Next fee calculated from minimum required, not actual payment
    claimFee = claimFee + (claimFee * feeIncreasePercentage) / 100;
    //        ^^^^^^^^   ^^^^^^^^ - Uses old minimum, ignores sentAmount
}
```

**The Problem:**
[M-02]()|Previous King Payout Mechanism Completely Missing
1. Player pays significantly more than required (strategic overpayment)
2. All excess payment goes to pot/fees but provides no strategic protection
3. Next challenger only pays a small increase based on the original minimum
4. Large investments become economically irrational and strategically worthless

**Expected Behavior (Last Man Standing Games):**
The next claim fee should be based on what was actually paid, making overpayment strategically valuable by creating higher barriers for challengers.

**Current Broken Behavior:**
Overpayments are wasted, providing no protection against dethroning despite players investing significantly more.

## Proof Of Concept

```solidity
function test_claimFee_should_be_increased_by_percentage_from_sentAmount() public {
        // setting initial fee to 1ETH in constructor for simplicity
                // uint256 public constant INITIAL_CLAIM_FEE = 1 ether;
        // Initial state: claimFee = 1 ETH
        uint256 initialClaimFee = game.claimFee();
        assertEq(initialClaimFee, 1 ether);

        // STEP 1: Player1 makes strategic investment - pays 5 ETH instead of 1 ETH
        // Player1 expects this will make it much harder for challengers
        vm.startPrank(player1);
        game.claimThrone{value: 5 ether}(); // 5x overpayment for protection
        vm.stopPrank();

        assertEq(game.currentKing(), player1);

        // STEP 2: Check what the next challenger must pay
        uint256 actualNextFee = game.claimFee();

        // BROKEN: Fee only increased by percentage of original 1 ETH
        uint256 expectedBrokenFee = initialClaimFee + (initialClaimFee * game.feeIncreasePercentage()) / 100;
        // With 10% increase: 1 + (1 * 10)/100 = 1.1 ETH

        assertEq(actualNextFee, expectedBrokenFee); // Currently 1.1 ETH
        console2.log("Player1 invested: 5 ETH");
        console2.log("Next challenger only needs: ", actualNextFee); // Only 1.1 ETH!

        // STEP 3: Player2 easily dethrones Player1 for minimal cost
        vm.startPrank(player2);
        game.claimThrone{value: actualNextFee}(); // Just 1.1 ETH vs Player1's 5 ETH
        vm.stopPrank();

        assertEq(game.currentKing(), player2);

        // RESULT: Player1 lost 5 ETH investment, Player2 only paid 1.1 ETH
        // Player1's 4.9 ETH overpayment provided zero strategic benefit
    }

```

## Impact

* **Economic Irrationality**: Large investments provide minimal protection, making strategic overpayment worthless

* **Broken Game Mechanics**: "Last Man Standing" escalation doesn't work as intended

* **User Financial Loss**: Players waste significant ETH on overpayments that provide no benefit

## Mitigation

* Recommended Fix: Base Fee Calculation on Actual Payment

  * In this implementation \`feeIncreasePercentage\` will act as min increase amount above current king payed amount&#x20;

* Another Fix:Don't allow excess payments

```diff
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
    
+    uint256 sentAmount = msg.value;
    uint256 previousKingPayout = 0;
    uint256 currentPlatformFee = 0;
    uint256 amountToPot = 0;
    
    // ... existing platform fee and pot logic ...
    
    // Update game state
    currentKing = msg.sender;
    lastClaimTime = block.timestamp;
    playerClaimCount[msg.sender] = playerClaimCount[msg.sender] + 1;
    totalClaims = totalClaims + 1;
    
    // FIXED: Calculate next fee based on actual payment, not minimum required
+    claimFee = sentAmount + (sentAmount * feeIncreasePercentage) / 100;
    
    emit ThroneClaimed(msg.sender, sentAmount, claimFee, pot, block.timestamp);
}
```
## [M-01] Critical comparison operator Error in `claimThrone`
### Summary

A critical vulnerability exists in the `claimThrone()` function due to an incorrect comparison operator. The function uses `==` instead of `!=` when checking if the sender is already the current king, preventing anyone from claiming the throne and effectively breaking the core game functionality.

## Description

The vulnerability is located in  [`claimThrone()`](https://github.com/CodeHawks-Contests/2025-07-last-man-standing/blob/47d9d19a78acb52270269f4bff1568b87eb81a96/src/Game.sol#L188) function:

```solidity
require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
```

This logic is inverted. The function should prevent the current king from re-claiming the throne, but instead it prevents anyone who is NOT the current king from claiming it.&#x20;

### Current Behavior:&#x20;

* If `msg.sender` is the current king → requirement passes → function continues

* If `msg.sender` is NOT the current king → requirement fails → transaction reverts

### Expected Behavior:

* If `msg.sender` is the current king → requirement fails → transaction reverts with appropriate message

* If `msg.sender` is NOT the current king → requirement passes → function continues

## Impact&#x20;

* **Complete Game Breakdown**: The core functionality of the game is completely broken. No player can claim the throne except potentially the initial king which is `address(0)`.

* **Financial Loss**: Players who attempt to claim the throne will have their transactions fail, wasting gas fees.

* **Contract Becomes Unusable**: The primary purpose of the contract (throne claiming mechanism) is non-functional.

## Proof of Concept

```Solidity
function test_noOne_can_claimThrone() public {
    vm.startPrank(player1);
    assert(game.currentKing() != player1);  // Confirms player1 is not the current king
    vm.expectRevert();                      // Expects the transaction to revert
    game.claimThrone{value: INITIAL_CLAIM_FEE}();  // Attempt to claim throne fails
    vm.stopPrank();
}
```

## Recommended Mitigation

```diff

- require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");

+ require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
```

## [M-02] Previous King Payout Mechanism Completely Missing
### Summary

The `claimThrone()` function fails to implement the [Documented](https://github.com/CodeHawks-Contests/2025-07-last-man-standing?tab=readme-ov-file#2-king-current-king) game mechanic where the previous king should "receive a small payout from the next player's `claimFee`". The `previousKingPayout` variable is hardcoded to 0 and never updated, resulting in previous kings receiving no compensation when `dethroned`, despite this being a core documented feature.

## Description

According to the game [documentation](https://github.com/CodeHawks-Contests/2025-07-last-man-standing?tab=readme-ov-file#2-king-current-king), the current king has the power to "receive a small payout from the next player's `claimFee` (if applicable)". However, the implementation in `claimThrone()` contains a critical flaw:

1. **Variable Initialization**: [`previousKingPayout`](https://github.com/CodeHawks-Contests/2025-07-last-man-standing/blob/47d9d19a78acb52270269f4bff1568b87eb81a96/src/Game.sol#L191) is initialized to 0 and never modified
2. **No Payout Logic**: There is no code to calculate or transfer any amount to the previous king
3. **Misleading Code**: The presence of `previousKingPayout` variable suggests this feature was planned but never implemented
4. **Redundant Check**: [The defensiveCheck](https://github.com/CodeHawks-Contests/2025-07-last-man-standing/blob/47d9d19a78acb52270269f4bff1568b87eb81a96/src/Game.sol#L199C8-L201C10) becomes meaningless since `previousKingPayout` is always 0

```Solidity

  function claimThrone() external payable gameNotEnded nonReentrant {
    // @Cropped
     uint256 previousKingPayout = 0;
     if (currentPlatformFee > (sentAmount - previousKingPayout)) {
            currentPlatformFee = sentAmount - previousKingPayout;
        }
    // @restOfLogic

}
```

The current flow simply:

* Takes the new player's `claimFee`

* [Deducts platform fee](https://github.com/CodeHawks-Contests/2025-07-last-man-standing/blob/47d9d19a78acb52270269f4bff1568b87eb81a96/src/Game.sol#L205)

* [Adds remainder to pot](https://github.com/CodeHawks-Contests/2025-07-last-man-standing/blob/47d9d19a78acb52270269f4bff1568b87eb81a96/src/Game.sol#L206)

* [Updates the king without compensating the previous one](https://github.com/CodeHawks-Contests/2025-07-last-man-standing/blob/47d9d19a78acb52270269f4bff1568b87eb81a96/src/Game.sol#L209)

```Solidity
 function claimThrone() external payable gameNotEnded nonReentrant {
// @Cropped

     // 1.Calculate platform fee
        currentPlatformFee = (sentAmount * platformFeePercentage) / 100;
    // 2.deduct it from amount going to pot 
        amountToPot = sentAmount - currentPlatformFee;
   // 3.Updating pot
        pot = pot + amountToPot;
   // @RestOfLogic
}
```

## Impact

* **Broken Game Mechanics**: Core documented functionality is completely missing

* **Economic Incentive Failure**: No immediate reward for becoming king reduces player engagement

## Proof of Concept

```Solidity
function test_prevKing_wont_Receives_payout_from_the_next_player() public {
    // Player1 becomes king
    vm.startPrank(player1);
    assertEq(player1.balance, 10 ether);
    uint expectedBalance = player1.balance - INITIAL_CLAIM_FEE;
    game.claimThrone{value: INITIAL_CLAIM_FEE}();
    assertEq(player1.balance, expectedBalance);
    vm.stopPrank();

    assertEq(game.currentKing(), player1);

    // Player2 claims throne (should trigger payout to Player1)
    vm.startPrank(player2);
    uint newClaimFee = INITIAL_CLAIM_FEE + (INITIAL_CLAIM_FEE * game.feeIncreasePercentage()) / 100;
    game.claimThrone{value: newClaimFee}();
    vm.stopPrank();
    
    assertEq(game.currentKing(), player2);
    
    // BUG: Player1's balance unchanged - should have received payout
    assertEq(player1.balance, expectedBalance);
}
```

## Mitigation

* you can rewrite `calimThrone` as following

```diff
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");

    uint256 sentAmount = msg.value;
    uint256 previousKingPayout = 0;
    uint256 currentPlatformFee = 0;
    uint256 amountToPot = 0;

    // Calculate and pay previous king (if exists)
+     if (currentKing != address(0)) {
        // Example: 5% of claim fee goes to previous king
+         previousKingPayout = (sentAmount * previousKingPayoutPercentage) / 100;
        
        // Transfer to previous king
       // This is safe since we're using nonReentrant Modifier
+         if (previousKingPayout > 0) {
+            (bool success, ) = currentKing.call{value: previousKingPayout}("");
+             require(success, "Game: Transfer to previous king failed");
        }
    }
  // Defensive check to ensure platformFee doesn't exceed available amount after previousKingPayout
        if (currentPlatformFee > (sentAmount - previousKingPayout)) {
            currentPlatformFee = sentAmount - previousKingPayout;
        }
        platformFeesBalance = platformFeesBalance + currentPlatformFee;

        // Remaining amount goes to the pot
        amountToPot = sentAmount - currentPlatformFee;
        pot = pot + amountToPot;

        // Update game state
        currentKing = msg.sender;
        lastClaimTime = block.timestamp;
        playerClaimCount[msg.sender] = playerClaimCount[msg.sender] + 1;
        totalClaims = totalClaims + 1;

        // Increase the claim fee for the next player
        claimFee = claimFee + (claimFee * feeIncreasePercentage) / 100;

        emit ThroneClaimed(
            msg.sender,
            sentAmount,
            claimFee,
            pot,
            block.timestamp
        );
}
```
