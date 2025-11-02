# Findings Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[M-01](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-OpenEdenUSDO.md#m-01-using-the-same-heartbeat-for-multiple-price-feeds-causing-dos)|Using the same heartbeat for multiple price feeds, causing DOS|MEDIUM|
|[M-02](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-OpenEdenUSDO.md#m-02-redemption-cancellation-results-in-shares-loss-due-to-bonus-multiplier-mismatch)|Redemption Cancellation Results in shares Loss Due to Bonus Multiplier Mismatch|MEDIUM|
|[L-01](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-OpenEdenUSDO.md#l-01-inconsistent-use-of-the-storage-__gap-variable) |Inconsistent Use of the storage __gap Variable|LOW|
|[L-02](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-OpenEdenUSDO.md#l-02-assetregistery_getfreshprice-uses-deprecated-answerinround)| `AssetRegistery::_getFreshPrice` Uses deprecated `answerInRound`|LOW|
|[L-03](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-OpenEdenUSDO.md#l-03-incorrect-share-calculation-in-minting-process-due-to-future-multiplier-usage)| Incorrect Share Calculation in Minting Process Due to Future Multiplier Usage|LOW|


## [M-01] Using the same heartbeat for multiple price feeds, causing DOS
### Summary

The protocol uses a single global `maxStalePeriod` value (default 2 days) for all price feed staleness checks across all assets and chains. This creates a critical vulnerability as different asset price feeds on different chains have vastly different heartbeat intervals. This mismatch leads to either frequent DoS conditions (when heartbeat > staleness period) or insufficient staleness validation (when heartbeat < staleness period).

## Description

The vulnerability exists in the asset registry contract's price feed validation logic. The contract implements:

1.  **Global Staleness Period**: A single `maxStalePeriod` variable (initialized to 2 days) applied universally
2.  **Centralized Configuration**: The `setMaxStalePeriod()` function can only set one value for all assets
3.  **Rigid Validation**: The `_getFreshPrice()` function checks all price feeds against this single threshold

**Problematic Code Flow:**
```solidity

    AssetRegistery.sol
    // Single staleness period for all assets/chains
    maxStalePeriod = 2 days;
    
    // Applied to all price feed checks
    if (block.timestamp - updatedAt > maxStalePeriod) {
        revert AssetRegistryStalePriceData(updatedAt, block.timestamp, maxStalePeriod);
    }
```
**Real-World Heartbeat Variations:**

Chainlink price feeds have significantly different update frequencies depending on the asset and chain for example:

Chainlink price feeds have significantly different update frequencies depending on the asset and chain:

-   **USDC/USD on Ethereum**: ~82800s heartbeat
-   **USDC/USD on Base**: ~86400s heartbeat
-   **ETH/USD on Ethereum**: ~3600s heartbeat
-   **ETH/USD on Base**: ~1200s heartbeat
-   **BTC/USD on Ethereum**: ~3600s heartbeat
-   **BTC/USD on Base**: ~1200 heartbeat

**The protocol's claim of EVM-chain compatibility is undermined by this design, as deploying with a 2-day staleness period would:**

-   Allow critically stale prices on fast-updating feeds
-   Cause constant reverts on slow-updating feeds during normal operation
## Impact

1.  **Denial of Service (DoS)**
    -   Transactions will systematically revert when using assets with heartbeats longer than `maxStalePeriod`
2.  **Stale Price Acceptance**
    -   Feeds with shorter heartbeats could return dangerously outdated prices

## Mitigation

-   Update `AssetConfig` struct to store `maxStalePeriod` Per asset.

```diff

    struct AssetConfig {
        address asset;
        bool isSupported;
        address priceFeed;
    +   uint256 maxStalePeriod; 
    }
    
```

## Proof of Concept

### ChainLink PriceFeeds Docs

-   [Ethereum](https://hackenproof.com/redirect?url=https://docs.chain.link/data-feeds/price-feeds/addresses?page=1&testnetPage=1&network=ethereum)
-   [Base](https://hackenproof.com/redirect?url=https://docs.chain.link/data-feeds/price-feeds/addresses?page=1&testnetPage=1&network=base)
-   [BNB-Chain](https://hackenproof.com/redirect?url=https://docs.chain.link/data-feeds/price-feeds/addresses?page=1&testnetPage=1&network=bnb-chain)

## [M-02] Redemption Cancellation Results in shares Loss Due to Bonus Multiplier Mismatch
### Summary

When redemption requests are cancelled by the maintainer, users receive fewer shares than they originally burned due to the reminting process using the current bonus multiplier instead of the original multiplier from when shares were burned. This results in permanent shares loss for users.

### Description

The vulnerability exists in the interaction between the `redeemRequest()` and `cancel()` functions in the redemption queue system.

**Flow of the Issue:**

1.  **User Requests Redemption:** When a user calls `redeemRequest()`, their USD0 shares are burned:

```solidity 

    USD0ExpressV2.sol:
    function redeemRequest(address to, uint256 amt) external whenNotPausedRedeem {
    
     // @Cropped
        // Burn USDO from the user
      @>  _usdo.burn(from, amt);
      _redemptionInfo[to] += amt;
    
     // @Cropped
    }
```
2.  **Multiplier Increases:** The bonus multiplier increases over time according to the protocol's mechanics or Admin updated the multipler:

```solidity 

    USD0ExpressV2.sol:
        function addBonusMultiplier() external onlyRole(MULTIPLIER_ROLE) {
            if (_lastUpdateTS != 0) {
                if (block.timestamp < _lastUpdateTS + _timeBuffer) revert USDOExpressTooEarly(block.timestamp);
            }
    
       @>     _usdo.addBonusMultiplier(_increment);
            _lastUpdateTS = block.timestamp;
        }
        
```

```solidity

    USD0.sol:
        function updateBonusMultiplier(uint256 _bonusMultiplier) external onlyRole(DEFAULT_ADMIN_ROLE) {
            _updateBonusMultiplier(_bonusMultiplier);
        }
```

3.  **Maintainer Cancels Redemption:** When the maintainer cancels the redemption, tokens are reminted using the **current** higher multiplier:

```solidity 

    USD0ExpressV2.sol:
    function cancel(uint256 _len) external onlyRole(MAINTAINER_ROLE) {
     // @Cropped
    
            // Mint USDO back to the user
          
    @>        _safeMintInternal(sender, usdoAmt);
     // @Cropped
     }
```     

4.  **Tokens Minted with Wrong Multiplier:** The `_safeMintInternal()` function eventually calls `_mint()`, which converts the amount to shares using the current multiplier:

```solidity 

    USD0.sol:
    function convertToShares(uint256 amount) public view returns (uint256) {
     @>   return (amount * _BASE) / bonusMultiplier; // current multiplier
    }

```
```solidity 

    USD0.sol
    function _mint(address to, uint256 amount) private {
       // @Cropped
       
        (bool isAllowed, uint256 newTotalSupply, uint256 shares) = @>> checkNewTotalSupply(amount);
        if (!isAllowed) {
            revert USDOExceedsTotalSupplyCap(newTotalSupply, _totalSupplyCap);
        }
    
    @>    _totalShares += shares;
    
        unchecked {
    @>        _shares[to] += shares;
        }
    
        _afterTokenTransfer(address(0), to, amount);
    }
```
**Root Cause:** The redemption queue only stores the token amount but not the bonus multiplier at the time of burning. When reminting occurs, the current (higher) multiplier is used, resulting in fewer shares and thus fewer effective tokens for the user.

## Impact

1.  **Permanent shares Loss:** Users lose shares proportional to the multiplier increase between burn and remint
2.  **No User Control:** Users have no control over:
    -   Whether their redemption will be cancelled
    -   How long it takes before cancellation
    -   The multiplier increase or change during this period
3.  **Breaks User Trust:** Users requesting legitimate redemptions are penalized for protocol operational decisions

## Mitigation

1.  store multiplier in the `redeemRequest`
2.  refactor or create another `mint` function which accept minting with specific multiplier

## Proof of Concept
### Paste the following test in `USD0Express.ts::'Queue Redemption System'`

```js 
 it('POC: User loses tokens when redemption is cancelled due to bonus multiplier increase', async function () {
      console.log('\n============MyTestLogsStartHere============');
     await usdoExpress.connect(maintainer).updateAPY(2000); // 20.00%
       
      await usdo.mint(whitelistedUser.address,ethers.utils.parseUnits('500', 18));
      const initialUserBalance = await usdo.balanceOf(whitelistedUser.address);
      console.log('Initial user USDO balance:', ethers.utils.formatUnits(initialUserBalance, 18));
      const initialSharesBalance = await usdo.sharesOf(whitelistedUser.address);
      console.log('Initial user shares balance:', ethers.utils.formatUnits(initialSharesBalance, 18));
      const initialMultiplier = await usdo.bonusMultiplier();
      console.log('Initial bonus multiplier:', initialMultiplier.toString());

      // User requests redemption - tokens are burned
      const redeemAmount = ethers.utils.parseUnits('1000', 18);
      await usdoExpress.connect(whitelistedUser).redeemRequest(whitelistedUser.address, redeemAmount);

      const balanceAfterRedeem = await usdo.balanceOf(whitelistedUser.address);
      console.log('User balance after redemption request:', ethers.utils.formatUnits(balanceAfterRedeem, 18));
      expect(balanceAfterRedeem).to.equal(0); // All tokens burned

      // Scenario 1: Normal daily multiplier increments (simulating time passing)
      console.log('\n--- Scenario 1: Normal Daily Increments (20% APY) ---');

      // Simulate 10 days of multiplier increases (assuming 20% APY )
      // With time buffer, operator calls addBonusMultiplier once per day
      for (let i = 0; i <= 10; i++) {
        await usdoExpress.connect(operator).addBonusMultiplier();
        // Advance time by 1 day to pass the time buffer check
        await ethers.provider.send('evm_increaseTime', [86400]); // 1 day
        await ethers.provider.send('evm_mine', []);
      }

      const multiplierAfter10Days = await usdo.bonusMultiplier();
      console.log('Multiplier after 10 days:', multiplierAfter10Days.toString());

      const multiplierIncrease10Days = multiplierAfter10Days.sub(initialMultiplier);
      const percentageIncrease10Days = multiplierIncrease10Days.mul(10000).div(initialMultiplier);
      console.log('Multiplier increase after 10 days:', ethers.utils.formatUnits(percentageIncrease10Days, 2), '%');

      // Maintainer cancels the redemption - tokens are reminted with NEW multiplier
      await usdoExpress.connect(maintainer).cancel(1);

      const UserBalanceAfter = await usdo.balanceOf(whitelistedUser.address);
      console.log(' USDO balance After:', ethers.utils.formatUnits(UserBalanceAfter, 18));
      const sharesBalanceAfter = await usdo.sharesOf(whitelistedUser.address);
      console.log('User shares after redemption request:', ethers.utils.formatUnits(sharesBalanceAfter, 18));
      
      
      const shareloss10Days = initialSharesBalance.sub(sharesBalanceAfter);
      const lossPercentage10Days = shareloss10Days.mul(10000).div(initialSharesBalance);
      console.log('share loss after 10 days:', ethers.utils.formatUnits(shareloss10Days, 18));
      console.log('Loss percentage:', ethers.utils.formatUnits(lossPercentage10Days, 2), '%');

      // User has lost shares due to multiplier increase
      expect(sharesBalanceAfter).to.be.lt(initialSharesBalance);
      console.log('\n--- Scenario 2: Admin Directly Updates Multiplier ---');
      const initialUserBalance2 = await usdo.balanceOf(whitelistedUser.address);
      console.log('User balance before second redemption request:', ethers.utils.formatUnits(initialUserBalance2, 18));
      const initialSharesBalance2 = await usdo.sharesOf(whitelistedUser.address);
      console.log('User shares balance before second redemption request:', ethers.utils.formatUnits(initialUserBalance2, 18));
      // Reset for scenario 2: Burn remaining tokens and request redemption again
      await usdoExpress.connect(whitelistedUser).redeemRequest(whitelistedUser.address, UserBalanceAfter);

      // Scenario 2: Admin directly updates multiplier (operational change)
     

      const currentMultiplier = await usdo.bonusMultiplier();
      console.log('Current multiplier before admin update:', currentMultiplier.toString());

      // Admin updates multiplier significantly (e.g., due to high APY update or direct manipulation)
      // Increase by another 5% (simulating APY change or direct admin action)
      const newMultiplier = currentMultiplier.mul(105).div(100); // 5% increase
      await usdo.connect(owner).updateBonusMultiplier(newMultiplier);

      const multiplierAfterAdminUpdate = await usdo.bonusMultiplier();
      console.log('Multiplier after admin update:', multiplierAfterAdminUpdate.toString());

      const totalMultiplierIncrease = multiplierAfterAdminUpdate.sub(initialMultiplier);
      const totalPercentageIncrease = totalMultiplierIncrease.mul(10000).div(initialMultiplier);
      console.log('Total multiplier increase from initial:', ethers.utils.formatUnits(totalPercentageIncrease, 2), '%');
  
      // Maintainer cancels this redemption too
      await usdoExpress.connect(maintainer).cancel(1);

      const finalBalance = await usdo.balanceOf(whitelistedUser.address);
      console.log('User balance after second cancellation:', ethers.utils.formatUnits(finalBalance, 18));
      const finalSharesBalance = await usdo.sharesOf(whitelistedUser.address);
      console.log('User shares after second cancellation:', ethers.utils.formatUnits(finalSharesBalance, 18));

      const totalsharesLoss = initialSharesBalance2.sub(finalSharesBalance);
      const totalLossPercentage = totalsharesLoss.mul(10000).div(initialSharesBalance2);
 
      console.log('Total shares loss:', ethers.utils.formatUnits(totalsharesLoss, 18));
      console.log('Total loss percentage:', ethers.utils.formatUnits(totalLossPercentage, 2), '%');
   

      // Verify significant loss occurred
      expect(finalBalance).to.be.lt(initialUserBalance);
      expect(finalSharesBalance).to.be.lt(initialSharesBalance2);
      expect(totalsharesLoss).to.be.gt(0);
      
    });
```
### Run it via `npx hardhat test --grep 'POC: User loses tokens when redemption is cancelled due to bonus multiplier increase'`

### Logs 

```
============MyTestLogsStartHere============
Initial user USDO balance: 1000.0
Initial user shares balance: 1000.0
Initial bonus multiplier: 1000000000000000000
User balance after redemption request: 0.0

--- Scenario 1: Normal Daily Increments (20% APY) ---
Multiplier after 10 days: 1006027397260273972
Multiplier increase after 10 days: 0.6 %
 USDO balance After: 999.999999999999999999
User shares after redemption request: 994.008714596949891663
share loss after 10 days: 5.991285403050108337
Loss percentage: 0.59 %

--- Scenario 2: Admin Directly Updates Multiplier ---
User balance before second redemption request: 999.999999999999999999
User shares balance before second redemption request: 999.999999999999999999
Current multiplier before admin update: 1006027397260273972
Multiplier after admin update: 1056328767123287670
Total multiplier increase from initial: 5.63 %
User balance after second cancellation: 999.999999999999999999
User shares after second cancellation: 946.674966282809421169
Total shares loss: 47.333748314140470494
Total loss percentage: 4.76 %
```

## [L-01] Inconsistent Use of the storage __gap Variable
### Summary

Multiple UUPS upgradeable contracts (`USDOExpressV2`, `AssetRegistry`, `UsycRedemption`, `LiquidityController`) lack storage gap reservations, creating significant risks for future upgrades and potential storage collision vulnerabilities when these contracts are extended or modified.

## Description

The codebase implements several contracts inheriting from `UUPSUpgradeable`, but there is an inconsistency in storage gap implementation:

**Contracts WITH storage gaps:**

-   `USDO` - implements `uint256[41] private __gap;`

**Contracts WITHOUT storage gaps:**

-   `USDOExpressV2`
-   `AssetRegistry`
-   `UsycRedemption`
-   `LiquidityController`

Storage gaps are reserved storage slots that allow future versions of upgradeable contracts to add new state variables without causing storage collisions with child contracts.

> Note!  
> Storage gap in UUPS contract Only Protects UUPS Variables

## Impact

1.  **Upgrade Path Limitations**: Future versions cannot safely add new state variables without risking storage collisions
    
2.  **Inheritance Issues**: If any of these contracts are extended by child contracts (now or in the future), adding state variables to the parent will corrupt child contract storage
    
3.  **Data Corruption Risk**: Upgrades that add state variables could overwrite existing data in derived contracts or create unexpected storage collisions
    

## Mitigation

Implement storage gaps (calculatedOnes) in all upgradeable contracts following OpenZeppelin's recommended pattern:
```solidity 
/**
 * @dev This empty reserved space is put in place to allow future versions to add new
 * variables without shifting down storage in the inheritance chain.
 * See https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps
 */
uint256[50] private __gap;
```

## [L-02] `AssetRegistery::_getFreshPrice` Uses deprecated `answerInRound`
### Summary

The `AssetRegistry::_getFreshPrice` function implements stale price checks using the deprecated `answeredInRound` parameter from Chainlink's `latestRoundData()` function. According to Chainlink's official documentation, this parameter is no longer maintained and should not be used for validation purposes.

## Description

The `_getFreshPrice` function retrieves price data from Chainlink price feeds and performs several validation checks to ensure data quality:

```solidity 
function _getFreshPrice(address priceFeed) internal view returns (uint256 price, uint8 decimals) {
    (uint80 roundId, int256 answer, , uint256 updatedAt, uint80 answeredInRound) = IPriceFeed(priceFeed)
        .latestRoundData();

    if (answer <= 0) revert AssetRegistryInvalidPrice(answer);
    if (block.timestamp - updatedAt > maxStalePeriod) {
        revert AssetRegistryStalePriceData(updatedAt, block.timestamp, maxStalePeriod);
    }

    // Check for incomplete round data
  
@>    if (answeredInRound < roundId) {
        revert AssetRegistryStalePriceData(updatedAt, block.timestamp, maxStalePeriod);
    }

    price = uint256(answer);
    decimals = IPriceFeed(priceFeed).decimals();
}
```
**Why This Is a Problem:**

According to Chainlink's official documentation, the `answeredInRound` parameter has been deprecated and is no longer guaranteed to be accurate or maintained.

## Impact

1.  **No Guarantee of Accuracy**: The `answeredInRound` value may not be reliably updated or maintained by Chainlink oracles across different feeds
2.  **Inconsistent Behavior**: The deprecated parameter may behave differently across various Chainlink price feeds, networks, or oracle versions
3.  **Price Query DOS**:The unreliable deprecated check may incorrectly reject valid price data, causing legitimate transactions to revert

## Mitigation

-   **Remove the deprecated `answeredInRound` check entirely** and rely on the robust `updatedAt` timestamp validation

## Proof Of Concept

**Reference Documentation:**

According to Chainlink's official API documentation for `latestRoundData()`:

**Source:** [https://docs.chain.link/data-feeds/api-reference#latestrounddata](https://hackenproof.com/redirect?url=https://docs.chain.link/data-feeds/api-reference#latestrounddata)

The documentation explicitly states that `answeredInRound` is deprecated and should not be used for validation purposes. The return values section shows:

-   `roundId`: The round ID
-   `answer`: The price
-   `startedAt`: Timestamp of when the round started
-   `updatedAt`: Timestamp of when the round was updated
-   `answeredInRound`: **Deprecated - Previously used for tracking round completion**

## [L-03] Incorrect Share Calculation in Minting Process Due to Future Multiplier Usage
### Summary

The `previewIssuance` function incorrectly calculates the amount of USD0 tokens to mint by applying a ratio of `curr/next` multipliers, where `next = curr + increment`. This causes new minters to receive shares calculated at a future bonus multiplier rate rather than the current rate, breaking the rebase token economics and unfairly diluting existing token holders.

## Description

USD0 is a shares-based rebase token where users hold shares and their token balance is calculated as `shares * bonusMultiplier / BASE`. When the `bonusMultiplier` increases (through `addBonusMultiplier`), all holders' token balances increase proportionally as a form of yield distribution.

The minting flow contains a flaw in the `previewIssuance` function:

```solidity 
function getBonusMultiplier() public view returns (uint256 curr, uint256 next) {
    curr = _usdo.bonusMultiplier();
    next = curr + _increment;  // next is always >= curr
}

function previewIssuance(uint256 usdoAmt) public view returns (uint256 usdoAmtCurr, uint256 usdoAmtNext) {
    (uint256 curr, uint256 next) = getBonusMultiplier();
    usdoAmtCurr = usdoAmt.mulDiv(curr, next);  //  Reduces mint amount
    usdoAmtNext = usdoAmtCurr.mulDiv(next, curr);
}
```
**The mathematical effect:**

```solidity
shares = (usdoAmtCurr * BASE) / curr
shares = (usdoAmt * curr/next * BASE) / curr
shares = (usdoAmt * BASE) / next
```

This means shares are effectively calculated using the **future multiplier** (`next`) instead of the current multiplier (`curr`).

## Impact

### 1\. Direct Loss of User Funds (Critical Mispricing)

Every user who mints USD0 tokens **immediately loses a percentage of their deposit** equal to the daily increment ratio. While the per-transaction loss appears small, it is **continuous, systematic, and permanent**.

This loss is **permanent and unrecoverable** at the time of minting. The user deposited full collateral but received fewer tokens than the collateral is worth at current market rates.

**Critical Factor: This Happens On Every Single Mint**

Unlike a one-time exploit, this bug affects:

-   Every user who mints
-   Every single minting transaction
-   Continuously throughout the protocol's lifetime
-   Compounds for users who mint regularly

**Annual Impact for Regular Minters:**

If a user mints once per day for a year (e.g., DCA strategy):

```solidity 
Daily loss = 0.0137% (at 5% APY)
Annual compound ≈ 365 × 0.0137% ≈ 5%
```

**The user loses approximately the entire annual yield through systematic underpricing!**

### 2\. Protocol Under-Collateralization Risk

The bug creates a systemic under-collateralization scenario that accumulates with every mint:

**Per-Transaction Analysis (10,000 USDC deposit, 5% APY):**

-   Treasury receives: 10,000 USDC (full collateral)
-   User receives: 9,998.63 USD0 tokens worth of shares
-   Missing backing: 1.37 USD0 per transaction

**Daily Volume Impact:**

For a protocol with $1M daily minting volume at 5% APY:

-   Daily collateral received: $1,000,000
-   Daily tokens issued: $998,630
-   Daily under-collateralization: **$1,370**
-   Annual cumulative: 1,370×365\=∗∗1,370 × 365 = \*\*1,370×365\=∗∗500,050\*\*

For a protocol with $10M daily minting volume at 10% APY:

-   Daily collateral received: $10,000,000
-   Daily tokens issued: $9,972,600
-   Daily under-collateralization: **$27,400**
-   Annual cumulative: 27,400×365\=∗∗27,400 × 365 = \*\*27,400×365\=∗∗10,001,000\*\*

> \[!NOTE\]  
> OpenEden has TVL about 280Mill

**Systematic Insolvency Risk:**

The protocol accumulates this "missing backing" continuously:

```solidity 
Theoretical Backing Deficit = Σ(daily_minting_volume × increment_rate)
```
Over time, this creates:

-   **Growing collateral gap** between what treasury holds and what shares represent
-   **Compounding accounting discrepancy** that increases with protocol adoption
-   **Systemic insolvency risk** if this goes undetected for extended periods

While individual transactions appear over-collateralized initially (105% at 5% APY), when the multiplier increases daily, the shares rebase to their "true" value, revealing the protocol has effectively taken excess collateral without proper accounting.

## Mitigation

Remove the `curr` ratio calculation from `previewIssuance`. Minting should always occur at the current bonus multiplier rate:
```solidity 
function previewIssuance(uint256 usdoAmt) public view returns (uint256 usdoAmtCurr, uint256 usdoAmtNext) {
    usdoAmtCurr = usdoAmt;  // Mint at current rate
    usdoAmtNext = usdoAmt.mulDiv(next, curr) // no change   
```

## Proof Of Concept
### Paste the following test in `USD0Express.ts`

```js
    it('POC: Users lose funds due to incorrect share calculation in previewIssuance', async function () {
      console.log('\n========================================');
      console.log('POC: Incorrect Share Calculation Bug');
      console.log('========================================\n');

      const depositAmount = ethers.utils.parseUnits('10000', 6); // 10,000 USDC

      // Give user USDC and approve
      await usdc.transfer(whitelistedUser.address, depositAmount);
      await usdc.connect(whitelistedUser).approve(usdoExpress.address, depositAmount);

      // Get current state
      const apy = await usdoExpress._apy();
      const increment = await usdoExpress._increment();
      const currentMultiplier = await usdo.bonusMultiplier();
      const BASE = ethers.utils.parseUnits('1', 18);

      console.log('Initial State:');
      console.log('- Deposit Amount: 10,000 USDC');
      console.log('- APY:', apy.toString(), 'bps (5%)');
      console.log('- Daily Increment:', ethers.utils.formatUnits(increment, 18));
      console.log('- Current Multiplier:', ethers.utils.formatUnits(currentMultiplier, 18));
      console.log('- Next Multiplier:', ethers.utils.formatUnits(currentMultiplier.add(increment), 18));

      // Preview mint to see the bug in action
      const preview = await usdoExpress.previewMint(usdc.address, depositAmount);
      console.log('\n--- Preview Mint Results ---');
      console.log('- Net Amount (after fee):', ethers.utils.formatUnits(preview.netAmt, 6), 'USDC');
      console.log('- Fee:', ethers.utils.formatUnits(preview.fee, 6), 'USDC');
      console.log('- USD0 Amount (Current):', ethers.utils.formatUnits(preview.usdoAmtCurr, 18), 'USD0');
      console.log('- USD0 Amount (Next):', ethers.utils.formatUnits(preview.usdoAmtNext, 18), 'USD0');

      // Calculate expected vs actual
      const netAfterFee = preview.netAmt; // 9990 USDC (after 0.1% fee)
      const expectedUsd0 = ethers.utils.parseUnits('9990', 18); // Should receive 9990 USD0
      const actualUsd0 = preview.usdoAmtCurr;
      const loss = expectedUsd0.sub(actualUsd0);
      const lossPercentage = loss.mul(10000).div(expectedUsd0); // in basis points

      console.log('\n--- Bug Impact Analysis ---');
      console.log('- Expected USD0 to receive:', ethers.utils.formatUnits(expectedUsd0, 18));
      console.log('- Actual USD0 received:', ethers.utils.formatUnits(actualUsd0, 18));
      console.log('- Immediate Loss:', ethers.utils.formatUnits(loss, 18), 'USD0');
      console.log('- Loss Percentage:', (lossPercentage.toNumber() / 100).toFixed(4) + '%');
      console.log('- Loss in Dollars: $' + ethers.utils.formatUnits(loss, 18));

      // Record initial balances
      const initialUserUsdcBalance = await usdc.balanceOf(whitelistedUser.address);
      const initialUserUsd0Balance = await usdo.balanceOf(whitelistedUser.address);
      expect(initialUserUsd0Balance).to.be.equal(0, 'User should start with 0 USD0');
      // Perform the mint
      await usdoExpress.connect(whitelistedUser).instantMint(
        usdc.address,
        whitelistedUser.address,
        depositAmount
      );

      const finalUserUsdcBalance = await usdc.balanceOf(whitelistedUser.address);
      const finalUserUsd0Balance = await usdo.balanceOf(whitelistedUser.address);

      console.log('\n--- Actual Mint Results ---');
      console.log('- USDC Spent:', ethers.utils.formatUnits(initialUserUsdcBalance.sub(finalUserUsdcBalance), 6));
      console.log('- USD0 Received:', ethers.utils.formatUnits(finalUserUsd0Balance, 18));
      

      // Mathematical proof that shares are calculated at future rate
      console.log('\n--- Mathematical Proof ---');
      console.log('Bugged Formula:');
      console.log('  usdoAmtCurr = netAmt × (curr / next)');
      console.log('  shares = usdoAmtCurr × BASE / curr');
      console.log('  shares = (netAmt × curr/next) × BASE / curr');
      console.log('  shares = netAmt × BASE / next  <-- Using FUTURE multiplier!');
      console.log('');
      console.log('Expected Formula:');
      console.log('  shares = netAmt × BASE / curr  <-- Using CURRENT multiplier');

      // Calculate annual impact for regular minters
      const dailyLossPercentage = lossPercentage.toNumber() / 100;
      const annualizedLoss = dailyLossPercentage * 365;

      console.log('\n--- Annual Impact for Regular Minters ---');
      console.log('- Daily loss per mint:', dailyLossPercentage.toFixed(4) + '%');
      console.log('- If minting once per day for a year:');
      console.log('  Annual loss ≈', annualizedLoss.toFixed(2) + '%');
      console.log('  (User loses approximately 75% from the APY!)');

      console.log('\n--- Volume Impact Projection ---');
      const dailyVolume = ethers.utils.parseUnits('5000000', 6); // $5M daily
      const dailyLossDollars = dailyVolume.mul(lossPercentage).div(10000);
      const annualLossDollars = dailyLossDollars.mul(365);

      console.log('- Assuming $5M daily minting volume:');
      console.log('  Daily losses: $' + ethers.utils.formatUnits(dailyLossDollars, 6));
      console.log('  Annual losses: $' + ethers.utils.formatUnits(annualLossDollars, 6));

      console.log('\n========================================\n');

      // Assertions to prove the bug
      expect(actualUsd0).to.be.lt(expectedUsd0, 'User should receive less USD0 than expected due to bug');
      expect(loss).to.be.gt(0, 'There should be a measurable loss');
      
    });


```

### Run it via `npx hardhat test --grep 'POC: Users lose funds due to incorrect share calculation in previewIssuance'`

### Logs 

```
========================================
POC: Incorrect Share Calculation Bug
========================================

Initial State:
- Deposit Amount: 10,000 USDC
- APY: 500 bps (5%)
- Daily Increment: 0.000136986301369863
- Current Multiplier: 1.0
- Next Multiplier: 1.000136986301369863

--- Preview Mint Results ---
- Net Amount (after fee): 9990.0 USDC
- Fee: 10.0 USDC
- USD0 Amount (Current): 9988.631694288453636624 USD0
- USD0 Amount (Next): 9989.999999999999999999 USD0

--- Bug Impact Analysis ---
- Expected USD0 to receive: 9990.0
- Actual USD0 received: 9988.631694288453636624
- Immediate Loss: 1.368305711546363376 USD0
- Loss Percentage: 0.0100%
- Loss in Dollars: $1.368305711546363376

--- Actual Mint Results ---
- USDC Spent: 10000.0
- USD0 Received: 9988.631694288453636624

--- Mathematical Proof ---
Bugged Formula:
  usdoAmtCurr = netAmt × (curr / next)
  shares = usdoAmtCurr × BASE / curr
  shares = (netAmt × curr/next) × BASE / curr
  shares = netAmt × BASE / next  <-- Using FUTURE multiplier!

Expected Formula:
  shares = netAmt × BASE / curr  <-- Using CURRENT multiplier

--- Annual Impact for Regular Minters ---
- Daily loss per mint: 0.0100%
- If minting once per day for a year:
  Annual loss ≈ 3.65%
  (User loses approximately 75% from the APY!)

--- Volume Impact Projection ---
- Assuming $5M daily minting volume:
  Daily losses: $500.0
  Annual losses: $182500.0

========================================
```
