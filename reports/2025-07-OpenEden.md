# Findings Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[M-01](https://github.com/nem0thefinder/Audits/new/main/reports#m-01-oracle-sandwich-attack-enables-risk-free-arbitrage-in-t-bill-vault)|Oracle Sandwich Attack Enables Risk-Free Arbitrage in T-Bill Vault|MEDIUM|
|[L-01](https://github.com/nem0thefinder/Audits/new/main/reports#l-01-critical-mispricing-protocol-assumes-perfect-usdc-peg)|Critical Mispricing: Protocol Assumes Perfect USDC Peg|LOW|

## [M-01] Oracle Sandwich Attack Enables Risk-Free Arbitrage in T-Bill Vault
### Summary 
Users can exploit the absence of time delays between `deposits/withdrawals` and `T-Bill` oracle price updates to extract risk-free profits through sandwich attacks. By monitoring oracle price feeds off-chain, attackers can selectively participate only in profitable price movements while avoiding any losses, creating an unfair advantage over legitimate investors.
## Description 

The vulnerability stems from the immediate execution of deposits and withdrawals in relation to oracle price updates, allowing sophisticated actors to sandwich oracle updates for guaranteed profits.

### Attack Mechanism

1. **Share Calculation**: When users call the `deposit` function, their share amount is calculated using the [shares formula](https://github.com/OpenEdenHQ/openeden.vault.audit/blob/d18288e944df21729b18d430b2afec2da99b6287/contracts/OpenEdenVaultV4Impl.sol#L934)

2. **Price Dependency**: The T-Bill/USDC rate is determined by the [`tbillRateFormula`](https://github.com/OpenEdenHQ/openeden.vault.audit/blob/d18288e944df21729b18d430b2afec2da99b6287/contracts/OpenEdenVaultV4Impl.sol#L649C9-L649C64), which fetches `tbillUsdPrice` from the [`tbillPriceOracle`](https://github.com/OpenEdenHQ/openeden.vault.audit/blob/d18288e944df21729b18d430b2afec2da99b6287/contracts/OpenEdenVaultV4Impl.sol#L635C9-L636C32) that update Price every day according to the [natspac](https://github.com/OpenEdenHQ/openeden.vault.audit/blob/d18288e944df21729b18d430b2afec2da99b6287/contracts/TBillPriceOracle.sol#L6C1-L13C4)

3. **Sandwich Strategy**: Attackers can:
   - Monitor oracle price update in mempool 
   - front run price updates
   - Redeem shares immediately after the price update
   - Skip participation entirely when price updates are negative
## ProofOfConecpt 


+ **Paste the following PoC in:** `"deposit/withdraw tbill/usdc 1:1"` 

+ **Run it via:**  `npx hardhat test --grep "Oracle Update can be sandwitched"`

```solidity 
    it ("Oracle Update can be sandwitched",async function () {
      // Fetch the current prices

      const tbillPrice= await tbillOracle.latestAnswer();
      console.log ("tbillPrice:",tbillPrice); // 1:1 tbill/USD

      // investor1 observed that there is an update
      // He made a deposit
      await vaultV4.connect(investor1).deposit(_100k,investor1.address);
      const inv1shares= await vaultV4.balanceOf(investor1.address);
      const amountInvested = _100k;
      console.log("inv1shares",inv1shares);

      // previewing converting user shares to assets before update
      const inv1SharesToAssets=await vaultV4.previewRedeem(inv1shares);
      console.log("inv1sharesToAssets",inv1SharesToAssets)

      // Updating the price to 1:1.2 USD
      const NewtbillOraclePrice = BigNumber.from("100999999"); // to fit in the 1% max deviation
      await tbillOracle.updatePrice(NewtbillOraclePrice);
      const newPrice = await tbillOracle.latestAnswer();
      console.log("newPrice",newPrice);

      // previewing shares to assets after update
      inv1SharesToAssets=await vaultV4.previewRedeem(inv1shares);
      console.log("inv1sharesToAssets",inv1SharesToAssets)

      // Investor1 tries to redeem his shares
      // Prepare redemption contract
      await usycTokenIns.connect(usycTreasuryAccount).approve(usycRedemptionIns.address, _10M);
      const investor1Balance=await vaultV4.balanceOf(investor1.address);

      // Investor1 redeems their shares
      const investor1RedeemTx = await vaultV4.connect(investor1).redeemIns(investor1Balance, investor1.address);
      const investor1Receipt = await investor1RedeemTx.wait();

      const investor1RedeemEvent = investor1Receipt.events?.find(e => e.event === 'ProcessWithdraw');

      if (investor1RedeemEvent) {
        // args[2] = total assets before fee deduction
        // args[4] = actual assets transferred to user (after fee deduction)
        // args[8] = total fee deducted

        const investor1AssetsReceived = investor1RedeemEvent.args[4]; // _assetsToUser
        const investor1FeesPaid = investor1RedeemEvent.args[8];       // _totalFee


        console.log(`Investor1 (non-partner) received assets: ${investor1AssetsReceived}`);
        console.log(`Investor1 total fees paid: ${investor1FeesPaid}`);

        const profit =  investor1AssetsReceived-amountInvested;
        console.log(`Investor1 profit: ${profit}`); // 900$ on 100k deposit approx 1% freeRisk Money
        }
      });


```


## Impact   

- **Risk-Free Returns**: Attackers capture ~1% profit per positive oracle update with zero downside risk
- **Scalable Attack**: Profits scale linearly with available capital (e.g., $1M investment (if under max cap) → ~$10k profit per update)
- **Frequency**: With daily oracle updates and selective participation, attackers can achieve outsized annual returns

## Mitigation 

1. `Deposit/Withdraw` Lock before oracle Price Update
2. Use `TWAP` instead of fixed price to use avg price at time interval

## [L-01] Critical Mispricing: Protocol Assumes Perfect USDC Peg

### Summary 

The `tbillUsdcRate()` function incorrectly assumes USDC maintains a perfect 1:1 peg with USD by using a hardcoded constant `ONE = 1e8` instead of fetching real-time USDC/USD pricing data from an oracle. This creates significant pricing inaccuracies and exposes the protocol to financial risks during market stress events when USDC depegs from its intended $1.00 value.




## Description

The current implementation in `tbillUsdcRate()` calculates the T-bill to USDC conversion rate using the formula:

```solidity
rate = (tbillUsdPrice * tbillDecimalScaleFactor) / ONE;
```

Where `ONE = 1e8` represents a fixed assumption that 1 USDC = $1.00. The function name and documentation claim to provide "T-bill to USDC rate" conversion, but it actually provides "T-bill to assumed-USD rate" since no actual USDC pricing oracle is consulted.

**Key Issues:**
- USDC price is hardcoded as exactly $1.00
- No real-time USDC/USD price feed is utilized

## Proof Of Concept 

 ### USDC trades below $1.00 (like the March 2023 banking crisis when USDC dropped to $0.87)
+ Protocol assumes: 1 USDC = $1.00
+ Market reality: 1 USDC = $0.87
+ User buys USDC on market at $0.87
+ Deposits 100K USDC to protocol (protocol values it at $1.00)
+ [Shares Minted](https://github.com/OpenEdenHQ/openeden.vault.audit/blob/d18288e944df21729b18d430b2afec2da99b6287/contracts/OpenEdenVaultV4Impl.sol#L934C8-L934C74) to user According to static Usdc/usd rate   = `(100,000e6 * 1e6)/ ((1e8*1e6)/1e8)` = `1e11 tbillShares`

+ Correct amount of shares according to real USDC value = `(100,000e6 * 1e6)/ ((1e8*1e6)/0.85e8)`= `8.5e10 tbillShares`

    **User gets 17.6% more shares than they should**

### Scenario 2: USDC Trades Above $1.00 (Premium Situations)

**Setup:**
- Market reality: 1 USDC = $1.05 (high demand scenario)
- Protocol assumes: 1 USDC = $1.00

#### Deposit Loss:
- User deposits 100K USDC (market value = $105K)
- Protocol values deposit at only $100K (4.8% undervaluation)
- **Shares minted (incorrect):** `1e11 tbillShares`
- **Shares should be (correct):** `(100,000e6 * 1e6) / ((1e8*1e6)/1.05e8) = 1.05e11 tbillShares`
- **User loses:** 4.8% fewer shares than deserved
<hr>

> 💡 **Tip:** This can be applied for different actions such as `redeem` 



## Mitigation

Implement proper dual-oracle architecture:

```solidity
function tbillUsdcRate() public view returns (uint256 rate) {
    // Get T-bill/USD price
    (, int256 tbillAnswer,, uint256 tbillUpdatedAt,) = tbillUsdPriceFeed.latestRoundData();
    
    // Get USDC/USD price  
    (, int256 usdcAnswer,, uint256 usdcUpdatedAt,) = usdcUsdPriceFeed.latestRoundData();
    
    // Validate both prices
    if (tbillAnswer <= 0 || usdcAnswer <= 0) revert InvalidOraclePrice();
    if (block.timestamp - tbillUpdatedAt > 7 days) revert TBillPriceOutdated();
    if (block.timestamp - usdcUpdatedAt > 24 hours) revert UsdcPriceOutdated();
    
    // Calculate T-bill/USDC = (T-bill/USD) / (USDC/USD)
    rate = (uint256(tbillAnswer) * tbillDecimalScaleFactor) / uint256(usdcAnswer);
}
```
