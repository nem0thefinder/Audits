# Findings Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#h-01--assets-permanently-locked-due-to-killswitch-flag)|Assets Permanently Locked Due to KillSwitch Flag|HIGH|
|[H-02](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#h-02-zero-slippage-protection-in-toke-strategies-allocation)| Zero Slippage Protection in Toke strategies Allocation|HIGH|
|[H-03](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#h-03-deallocation-accounting-mismatch-between-vault-and-adapter)|Deallocation Accounting Mismatch Between Vault and Adapter|HIGH|
|[M-01](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#m-01-admin-can-bypass-permissionedcalls-protection-using-multicall)|Admin Can Bypass permissionedCalls Protection Using Multicall|MEDIUM|
|[L-01](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#l-01-missing-addresses-verification-in-zeroxswapverifier)|Missing addresses Verification in ZeroXSwapVerifier|LOW|
|[L-02](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#l-02-toke-rewards-permanently-locked-in-strategy-adapter)|TOKE Rewards Permanently Locked in Strategy adapter|LOW|
|[L-03](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#l-03--broken-isvalidsignature-leads-to-fund-freezing)|Broken isValidSignature leads to fund freezing |LOW|
|[L-04](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#l-04-inverted-comparison-operator-allows-operators-admin-level-allocation-privileges)|Inverted Comparison Operator Allows Operators Admin-Level Allocation Privileges|LOW|
|[L-05](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#l-05--reserve-drainage-due-to-incorrect-balance-measurement)|Reserve Drainage Due to Incorrect Balance Measurement|LOW|
|[I-01](https://github.com/nem0thefinder/Audits/blob/main/reports/2025-10-AlchemixV3.md#i-01-hardcoded-slippage-in-myt-strategy)|Hardcoded Slippage in MYT Strategy|INFO|








## [H-01]  Assets Permanently Locked Due to KillSwitch Flag
### Summary 
Funds are permanently locked when allocating to a strategy with an active `killSwitch`. The vault transfers assets to the strategy, but the strategy returns early without allocating them to the underlying protocol. The vault's allocation tracking remains at zero, making the funds unrecoverable through normal deallocation flow. No emergency rescue mechanism exists.
## Description

The bug occurs in the interaction between `AlchemixAllocator`, `VaultV2`, and `MYTStrategy`:

1.  `AlchemixAllocator.allocate()` does not check the strategy's `killSwitch` status

**AlchemixAllocator.sol:**

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    
    // Missing: killSwitch validation
    
    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    // ... cap checks ...
    
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```

1.  `VaultV2.allocateInternal()` transfers funds to the strategy

**VaultV2.sol:**

```solidity
function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
    require(isAdapter[adapter], ErrorsLib.NotAdapter());
    accrueInterest();
    
 @>   SafeERC20Lib.safeTransfer(asset, adapter, assets);
  @>  (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
    
    for (uint256 i; i < ids.length; i++) {
        Caps storage _caps = caps[ids[i]];
        _caps.allocation = (int256(_caps.allocation) + change).toUint256(); // change=0
        // ... cap checks ...
    }
}
```

1.  `MYTStrategy.allocate()` returns `(ids(), 0)` when `killSwitch` is true **MYTStrategy.sol:**

```solidity
function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
    external
    onlyVault
    returns (bytes32[] memory strategyIds, int256 change)
{
// Returns 0 change after receiving funds and don't continue to _allocate
@>    if (killSwitch) {
@>       return (ids(), int256(0)); 
    }
    require(assets > 0, "Zero amount");
    uint256 oldAllocation = abi.decode(data, (uint256));
    uint256 amountAllocated = _allocate(assets);
    uint256 newAllocation = oldAllocation + amountAllocated;
    emit Allocate(amountAllocated, address(this));
    return (ids(), int256(newAllocation) - int256(oldAllocation));
}
```

1.  Funds remain in the strategy contract as underlying tokens
2.  `vault.allocation(id)` stays at 0 because the returned `change` is 0
3.  Deallocation is impossible because it requires `allocation > 0`

### Execution Flow

```
AlchemixAllocator.allocate(strategy, 100 ether)
    └─> VaultV2.allocate(strategy, data, 100 ether)
        ├─> Transfer 100 ether to strategy ✓
        └─> strategy.allocate(data, 100 ether)
            └─> Returns (ids(), 0) due to killSwitch
        └─> vault.allocation += 0 (no change)
        
Result: 100 ether stuck in strategy, allocation = 0
```

## Impact

1.  **Permanent fund lock**: Allocated funds remain in the strategy contract indefinitely
2.  **Broken accounting**: Vault tracking shows zero allocation despite funds being transferred
3.  **No recovery mechanism**:
    -   Deallocation requires `allocation > 0` but it remains `0`
    -   No emergency withdrawal function exists in MYTStrategy
    -   Funds cannot be transferred back to vault

## Mitigation

-   Add `killSwitch` validation in `AlchemixAllocator.allocate()` before initiating the allocation:

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    
    // Check killSwitch status before allocation
  @>  if (IMYTStrategy(adapter).killSwitch()) {
  @>      return; // Exit early if strategy is paused
    }
    
// @Cropped
}
```

## [H-02] Zero Slippage Protection in Toke strategies Allocation
### Summary

The `_allocate()` function accepts zero shares from the AutoUSD vault by setting `minSharesOut = 0` in the `router.depositMax()` call. This allows the strategy to lose up to 100% of deposited funds through price manipulation, sandwich attacks, or unfavorable market conditions without any transaction revert

## Description

The vulnerability exists in the following code:

```solidity
TokeAutoUSDCStrategy.sol
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, 
            "Strategy balance is less than amount");
    
    TokenUtils.safeApprove(address(usdc), address(router), amount);
    
    // @audit-issue Setting minSharesOut to 0 is dangerous
    uint256 shares = router.depositMax(autoUSD, address(this), 0);
    //                                                          ^^^ No slippage protection
    
    TokenUtils.safeApprove(address(autoUSD), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    
    return amount;
}
```

The `depositMax()` function's third parameter (`minSharesOut`) is hardcoded to `0`, which means the function will accept **any amount of shares returned**, including:

-   Zero shares (100% loss)
-   Extremely small amounts due to slippage
-   Manipulated amounts from MEV attacks

## Impact

1.  **Sandwich Attack (89%+ loss)**
    
    -   Attacker front-runs the allocation transaction
    -   Inflates the AutoUSD vault share price via large donation or manipulation
    -   Strategy receives drastically fewer shares for the same USDC amount
    -   Attacker back-runs to extract profit
    -   **Demonstrated loss: 89.53% of funds**
2.  **Price Manipulation (100% loss possible)**
    
    -   Malicious actor manipulates vault state before strategy deposit
    -   Router returns zero shares due to rounding or extreme price manipulation
    -   Strategy accepts the zero shares and proceeds
    -   **Result: Complete loss of deposited USDC**
3.  **Unfavorable Market Conditions (Variable loss)**
    
    -   Sudden market volatility causes poor exchange rates
    -   No minimum threshold to protect against extreme slippage
    -   Strategy unknowingly accepts substantial losses

## Mitigation

-   Accept `minSharesOut` as function param and pass it to `depositMax` to have dynamic slippage on each allocation

## [H-03] Deallocation Accounting Mismatch Between Vault and Adapter

### Summary 


The adapter's `deallocate` function returns the requested deallocation amount instead of the actual amount received from the external strategy, causing the vault to pull more funds than the adapter actually withdrew. This creates an accounting mismatch where the allocation cap decreases by the requested amount, but the adapter must cover slippage losses from its own balance.

## Description

> Note!!
> 
> > This apply for all strategies When deallocating funds from an external strategy:

1.  The vault calls `deallocateInternal` requesting withdrawal of `assets` amount (e.g., 1000 USDC)
2.  The adapter withdraws from the external strategy via `_deallocate(amount)`
3.  The external strategy may return less than requested due to slippage (e.g., request 1000 USDC, receive 980 USDC)
4.  **The vault correctly updates the allocation cap by the requested amount** (decreases by 1000 USDC)
5.  The adapter emits a loss event but still returns `withdrawReturn = amount` (the full 1000 USDC)
6.  Then there are two Execution Paths
    1.  The require statement that check the adapter balance will revert if there's no balance cover the amount (DoS deallocation)
    2.  the require statement pass since the adapter balance cover the amount (there was prev.balance or team manual intervention)
7.  The adapter approves the full requested amount for transfer (1000 USDC)
8.  **The vault pulls the full requested amount** via `SafeERC20Lib.safeTransferFrom` (1000 USDC)

**The Problem**:

-   **Allocation cap update is CORRECT**: Decreases by 1000 USDC (the amount requested from the strategy)
-   **Transfer amount is WRONG**: Vault pulls 1000 USDC but adapter only received 980 USDC from the external strategy
-   The adapter must have sufficient USDC balance to cover the slippage loss (20 USDC in the example), otherwise the `require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, ...)` check will fail
-   **Expected behavior**: Vault should update allocation cap by requested amount (1000 USDC) but pull only the actual amount received (980 USDC)

 [VaultV2::_dealocateInternal](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/lib/vault-v2/src/VaultV2.sol#L596C1-L614C6)

```solidity
    function deallocateInternal(address adapter, bytes memory data, uint256 assets)
        internal
        returns (bytes32[] memory)
    {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

@>>    (bytes32[] memory ids, int256 change) = IAdapter(adapter).deallocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
   @>>         _caps.allocation = (int256(_caps.allocation) + change).toUint256();
        }

        SafeERC20Lib.safeTransferFrom(asset, adapter, address(this), assets);
        emit EventsLib.Deallocate(msg.sender, adapter, assets, ids, change);
        return ids;
    }
```

[AaveV3ARBUSDCStrategy::_deallocate](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol#L45C1-L58C1)

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
        // withdraw exact underlying amount back to this adapter
        pool.withdraw(address(usdc), amount, address(this));
        uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
        uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
@>        if (usdcRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
        }
@>        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
@>        TokenUtils.safeApprove(address(usdc), msg.sender, amount);
@>        return amount;
    }

```

The protocol acknowledges slippage is expected (via `slippageBPS` and `_previewAdjustedWithdraw`), but the implementation forces the adapter to cover losses rather than properly accounting for them.

## Impact

1.  **Fragile Dependency**: Adapter requires spare USDC balance to cover slippage, creating an implicit requirement that's not guaranteed
2.  **Incorrect Loss Attribution**: Slippage losses are absorbed by the adapter's balance rather than properly attributed to vault depositors. The vault's accounting is correct (allocation decreases by requested amount), but the actual transfer forces the adapter to make up the difference
3.  **Potential DoS**: If adapter doesn't have sufficient balance to cover slippage, deallocations will revert

## Mitigation:

### Strategies/Adapters

1.  **Calculate slippage-adjusted minimum**: `minAcceptable = amount - (amount * slippageBPS / 10000)`
2.  **Validate received amount meets minimum**: `require(redeemedAmount >= minAcceptable, "Slippage exceeded")`
3.  **Return allocation change based on requested amount**: Keep current logic where `change = int256(newAllocation) - int256(oldAllocation)` (based on `amount` requested, not `redeemedAmount`)
4.  **Approve only the actual redeemed amount**: `TokenUtils.safeApprove(address(usdc), msg.sender, redeemedAmount)`
5.  **Communicate actual redeemed amount to vault**: The adapter needs a way to return `redeemedAmount` separately from the allocation `change`

### MorpheusVault

1.  **Update allocation cap by the change** (already implemented correctly): Based on requested `assets` amount
2.  **Transfer only the actual redeemed amount**: `SafeERC20Lib.safeTransferFrom(asset, adapter, address(this), redeemedAmount)` instead of `assets`


## [M-01] Admin Can Bypass `permissionedCalls` Protection Using Multicall
### Summary 

The `AlchemistAllocator` contract blocks direct calls to `allocate()` and `deallocate()` functions through its `proxy()` function. However, an admin can bypass this protection by wrapping these calls inside the vault's `multicall()` function, completely defeating the intended privilege separation.

## Description

In `alchemistAllocator` constructor set `MorpheusVault::Allocate,Deallocate` functions as permissioned calls to prevent calling them directly from the `permissionedProxy`

```solidity
AlchemistAllocator.sol
constructor(address _vault, address _admin, address _operator) {
    // Block direct allocate/deallocate calls via proxy
    permissionedCalls[0x5c9ce04d] = true; // allocate
    permissionedCalls[0x4b219d16] = true; // deallocate
}
```

The issue here that the admin can bypass this through calling `MorpheusVault::multiCall` and route the calls to `Morphues::Allocate,dellocate` and since `multiCall` is not permissionedCall the call will succeed

```solidity
VaultV2.sol
// Vault has a multicall function (NOT blocked)
function multicall(bytes[] calldata data) external {
    for (uint256 i = 0; i < data.length; i++) {
        (bool success,) = address(this).delegatecall(data[i]);
        require(success);
    }
}
```

### Attack Flow

```
1. Admin calls: allocator.proxy(vault, multicall([allocate_calldata]))
2. proxy() extracts selector → 0xac9650d8 (multicall)
3. permissionedCalls[0xac9650d8] = false → Check passes ✓
4. proxy() calls: vault.multicall([allocate_calldata])
5. multicall() does: delegatecall(allocate_calldata)
6. allocate() executes with msg.sender = allocator address
7. isAllocator[allocator] = true → Access check passes ✓
8. Funds allocated successfully → BYPASS COMPLETE
```

## Impact

-   **Defeats Security Model**: The entire `permissionedCalls` protection is bypassed
-   **AlchemistAllocator Logic**:This path Ignore any custom logic, checks, or accounting in the `AlchemistAllocator`

## Mitigation

1.  Add multicall selector to permissionedCalls if we don't need to call it via proxy OR
2.  Restrict calling `allocate`and `deallocate` from `multicall`

## [L-01] Missing addresses Verification in ZeroXSwapVerifier

### Summary 
The `ZeroXSwapVerifier` library fails to validate the `recipient` and `owner`in 0x swap calldata, allowing operator or attackers to redirect swapped funds to arbitrary addresses during atomic deallocations.

## Description

When the vault executes atomic deallocations or force atomic deallocations, it relies on `ZeroXSwapVerifier` to validate 0x API calldata before execution. The verifier currently checks:

-   Sell token matches expected token
-   Slippage is within bounds
-   Action types are permitted

However, it doesn't verify `owner` is the legit owner or the `recipient` field in the `SlippageAndActions` struct, which specifies where the swapped tokens are sent.

### In the atomic deallocation flow:

1.  Allocator or user (via force deallocate ) provides 0x calldata
2.  Strategy calls `ZeroXSwapVerifier.verifySwapCalldata()`
3.  Strategy executes the calldata via 0xSettler
4.  **0xSettler automatically sends output tokens to `saa.recipient`**

Without recipient and owner validation, an user or operator can set `recipient` to their own address, causing the 0xSettler to send all swapped funds to the beneficiary address instead of the strategy or vault.

```solidity
function _verifyExecuteCalldata(bytes calldata data, address owner, ...) internal view {
    (SlippageAndActions memory saa,) = abi.decode(data, (SlippageAndActions, bytes));
    // Missing: owner and recepient checks
    _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
}

function _verifyExecuteMetaTxnCalldata(bytes calldata data, address owner, ...) internal view {
    (SlippageAndActions memory saa,,,) = abi.decode(data, (SlippageAndActions, bytes[], address, bytes));
  // Missing: owner and recepient checks
    _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
}
```

## Impact

-   **Direct Fund Theft**: User can set his address as `recepient` in the calldata when he is calling force deallocate or malicous operator can do the same on nomral atomic dellocation.
    
-   **Trust assumption violation**: The verifier's entire purpose is to validate untrusted calldata; without `recipient` and `owner` checks it fails this core objective
    

## Mitigation

-   Add `recipient` validation and `owner` in both verification functions `_verifyExecuteCalldata` and `_verifyExecuteMetaTxnCalldata`

## [L-02] TOKE Rewards Permanently Locked in Strategy adapter
### Summary 
TOKE reward tokens claimed from the Tokemak rewarder are permanently locked in the `TokeAutoUSDStrategy` contract with no mechanism to retrieve them, resulting in continuous value loss to the protocol and users.

## Description

The `TokeAutoUSDStrategy` contract stakes `autoUSD` shares received from allocating to `autoUsdVault` in a Tokemak `rewarder` that distributes `TOKE` tokens as rewards. During deallocation, the strategy claims these TOKE rewards:

```solidity 
TokeAutoUSDStrategy.sol
function _deallocate(uint256 amount) internal override returns (uint256) {
    // ...
 @>   rewarder.withdraw(address(this), sharesNeeded, true);  // claim=true
    // TOKE tokens are transferred to the strategy here
    
  @>  autoUSD.redeem(sharesNeeded, address(this), address(this));
    // Only autoUSD → USDC conversion happens

    // approve only usdc to vault to pull out from the adpater
    TokenUtils.safeApprove(address(usdc), msg.sender, amount);
    return amount;
    // TOKE tokens remain in the strategy contract
}
```

The issue here that `deallocate` function don't handle `Toke` tokens received from the rewarder leaving them locked in the contract. making the whole rewarder staking process useless.

## Impact

-   **Financial Loss:** TOKE rewards accumulate indefinitely in the strategy contract
-   **No Recovery Mechanism:** Tokens are permanently inaccessible to protocol, users, and governance

## Mitigation

1.  **Implement reward Handling:** override `MYT::claimRewards` and implement your custom logic for handling toke rewards OR
    
2.  **Token Rescue Function:** Add rescue function that allow trusted roles to withdraw those tokens.

## [L-03]  Broken isValidSignature leads to fund freezing 

### Summary 
The `isValidSignature()` function in `MYTStrategy` attempts to delegate signature verification to Permit2 by calling `IPermit2(permit2Address).isValidSignature()`, which does not exist in the Permit2 contract. This causes all Permit2 operations where the strategy contract acts as the token owner to revert, resulting in a complete denial of service of any Permit2-based functionality

## Description

The current implementation of `isValidSignature()` in `MYTStrategy.sol`:

```solidity
function isValidSignature(bytes32 _hash, bytes memory _signature) public view returns (bytes4) {
    return IPermit2(permit2Address).isValidSignature(_hash, _signature);
}
```

This implementation is fundamentally broken because:

1.  **Permit2 does not have an `isValidSignature` function** - The Permit2 contract (`SignatureTransfer` and `AllowanceTransfer`) does not expose any `isValidSignature` function. The interface `IPermit2` defined in the contract is incorrect.

[`SignatureTransfer.sol`](https://github.com/Uniswap/permit2/blob/main/src/SignatureTransfer.sol)

[`AllowanceTransfer.sol`](https://github.com/Uniswap/permit2/blob/main/src/AllowanceTransfer.sol)

1.  **Backwards delegation logic** - When Permit2 calls `permitTransferFrom()` with a smart contract as the owner, it internally calls that contract's `isValidSignature()` for ERC-1271 validation. The strategy contract should validate signatures itself, not delegate back to Permit2.
    
2.  **Incorrect understanding of ERC-1271 pattern** - The ERC-1271 standard requires the implementing contract to verify signatures locally and return the magic value `0x1626ba7e` if valid. The current implementation attempts to call an external contract instead.
    

### **How Permit2 Actually Works:**

When `ISignatureTransfer(permit2).permitTransferFrom(permit, details, owner, signature)` is called it initiaite an intenral call to `signatureVerify` lib which validate the sig:

[`_permitTransfer`]:(https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/SignatureTransfer.sol#L65)

```solidity
 function _permitTransferFrom(
        PermitTransferFrom memory permit,
        SignatureTransferDetails calldata transferDetails,
        address owner,
        bytes32 dataHash,
        bytes calldata signature
    ) private {
        uint256 requestedAmount = transferDetails.requestedAmount;
// @Cropped
@>>        signature.verify(_hashTypedData(dataHash), owner);

        ERC20(permit.permitted.token).safeTransferFrom(owner, transferDetails.to, requestedAmount);
    }

```

[`Signature::verify`]:(https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/libraries/SignatureVerification.sol#L42C8-L44C101)

```solidity
// Inside Permit2's SignatureVerification.verify()
if (owner.code.length > 0) {
    // Owner is a contract - call its isValidSignature
    bytes4 magicValue = IERC1271(owner).isValidSignature(hash, signature);
    if (magicValue != 0x1626ba7e) revert InvalidContractSignature();
}
```

When the owner is the strategy contract, Permit2 calls the strategy's `isValidSignature()`, which then tries to call Permit2's non-existent `isValidSignature()`, causing the transaction to revert.

## Impact

1.  **PermenantFreezeOfFunds**\- Based on protocol team intentions `permit2` gonna used with `0x` in strategies that don't have the ability to instantly withdraw, or require swaps.leaving the funds allocated forever
2.  **100% failure rate** - Any call to `permitTransferFrom()` where the strategy is the token owner will always revert
3.  **No workaround available** - MytStrategy is not upgradeable and The function will revert before any token transfer logic executes
4.  **Affects all future Permit2 features** - Any planned signature-based transfers, gasless transactions, or whitelisted allocator operations using Permit2 will be permanently broken
5.  **Blocks ZeroX integration** -Third-party protocols attempting to integrate with the strategy via Permit2 will fail

## Mitigation

-   Replace the current implementation with proper ERC-1271 signature verification

```solidity
import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

/// @notice ERC-1271 signature verification for Permit2 compatibility
/// @dev Permit2 calls this function to validate signatures when the strategy is the token owner
function isValidSignature(bytes32 _hash, bytes memory _signature) 
    public 
    view 
    returns (bytes4) 
{
    // Recover the signer from the signature
    address recovered = ECDSA.recover(_hash, _signature);
    
    // Check if the signer is authorized (owner or whitelisted allocator)
    if (recovered == owner() || whitelistedAllocators[recovered]) {
        return 0x1626ba7e; // ERC-1271 magic value indicating valid signature
    }
    
    return 0xffffffff; // Invalid signature
}
```
## [L-04] Inverted Comparison Operator Allows Operators Admin-Level Allocation Privileges
### Summary

The `allocate()` function in `AlchemistAllocator` uses an inverted comparison operator (`>` instead of `<`) when applying the `daoTarget` cap for operators. This bug will grant operators the same allocation privileges as admins once `daoTarget` is properly implemented, bypassing intended governance restrictions.

## Description

The `AlchemistAllocator` contract implements a privilege separation model where admins have unrestricted allocation rights, while operators should be subject to additional DAO-governed caps. The contract has two symmetric functions: `allocate()` and `deallocate()`.

**Wrong implementation in [`allocate()`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L34C8-L40C10)

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    // FIXME get this from the StrategyClassificationProxy for the respective risk class
    uint256 daoTarget = type(uint256).max;
    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
    if (msg.sender != admin) {
        // caller is operator
    @>>    adjusted = adjusted > daoTarget ? adjusted : daoTarget;  
    }
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```

**Correct implementation in [`deallocate()`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L56C7-L62C10)

```solidity
function deallocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    uint256 daoTarget = type(uint256).max;
    uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
    if (msg.sender != admin) {
        // caller is operator
   @>>     adjusted = adjusted < daoTarget ? adjusted : daoTarget; 
    }
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.deallocate(adapter, oldAllocation, amount);
}
```

### Key Differences

| Function | Operator Check | Result |
| --- | --- | --- |
| `allocate()` | `adjusted > daoTarget ? adjusted : daoTarget` | Takes **maximum** of both values (wrong) |
| `deallocate()` | `adjusted < daoTarget ? adjusted : daoTarget` | Takes **minimum** of both values (correc) |

### Current State vs Future Impact

**Currently:** The bug is dormant because `daoTarget` is hardcoded to `type(uint256).max`, making the check ineffective.

**When implemented:** Per the FIXME comment, `daoTarget` will be fetched from `StrategyClassificationProxy`. Once this happens, the inverted comparison will immediately allow operators to bypass governance-imposed caps.

## Impact

### Access Control Bypass

When `daoTarget` is properly implemented, operators will be able to allocate funds up to `max(adjusted, daoTarget)` instead of the intended `min(adjusted, daoTarget)`, effectively granting them admin-level privileges.

## Mitigation

Change the comparison operator in `allocate()` from `>` to `<`:

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    // FIXME get this from the StrategyClassificationProxy for the respective risk class
    uint256 daoTarget = type(uint256).max;
    uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
    if (msg.sender != admin) {
        // caller is operator
-       adjusted = adjusted > daoTarget ? adjusted : daoTarget;
+       adjusted = adjusted < daoTarget ? adjusted : daoTarget;
    }
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```

This change aligns `allocate()` with the correct logic already implemented in `deallocate()`.

## [L-05]  Reserve Drainage Due to Incorrect Balance Measurement
### Summary 
The `_deallocate()` function reads balances **after** the vault withdrawal instead of before and after. This makes `wethRedeemed` always equal zero, preventing validation of the actual amount received. When the external vault returns less than requested (due to fees or slippage), the strategy silently uses reserves to cover the shortfall. This drains accumulated yield until reserves are depleted, at which point all withdrawals fail permanently.

## Description

### The Bug

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    // ... 
    vault.withdraw(amount, address(this), address(this)); // Withdrawal happens here
    
    //  BOTH reads happen AFTER the withdrawal
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore; // Always ≈ 0
    
    // This check is meaningless since wethRedeemed ≈ 0
    require(wethRedeemed + wethBalanceBefore >= amount, ...);
    
    // This only checks total balance, not what the vault actually returned
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, ...);
    
    return amount; // Claims success even if vault returned less
}
```

**The Critical Flaw:**

-   Both `wethBalanceBefore` and `wethBalanceAfter` are read **after** `vault.withdraw()` completes
-   `wethRedeemed = wethBalanceAfter - wethBalanceBefore ≈ 0` (both values are identical)
-   The function cannot detect when the vault returns less than requested
-   Subsequent checks pass if the strategy has sufficient **total balance** (reserves + withdrawn amount)
-   This causes reserves to be silently consumed to cover vault shortfalls

### How It Fails

**Scenario:** External vault has 2% withdrawal fee

```
Strategy state:
├─ Reserves: 50 WETH (accumulated yield/rewards)
└─ Allocated: 800 vault shares

User withdraws 100 WETH:

1. vault.withdraw(100, this, this) 
   → Vault sends 98 WETH (2% fee)

2. Strategy balance: 50 + 98 = 148 WETH

3. wethBalanceBefore = 148 (read after) 
   wethBalanceAfter = 148 (read after) 
   wethRedeemed = 0

4. require(148 >= 100) Passes (checks total balance, not vault output)

5. Morpheus takes 100 WETH

6. Reserves now: 48 WETH (lost 2 WETH)

Result: Vault gave 98, Morpheus took 100, reserves covered the 2 WETH difference.
```

**Over multiple withdrawals, reserves drain completely:**

```
Withdrawal #1: Reserves 50 → 48 WETH (-2)
Withdrawal #2: Reserves 48 → 46 WETH (-2)
Withdrawal #3: Reserves 46 → 44 WETH (-2)
...
Withdrawal #25: Reserves 2 → 0 WETH (-2)
Withdrawal #26:  REVERTS (no reserves left to cover shortfall)
```

**Worst case:** If vault returns 0 , entire deallocation comes from reserves if while allocated assets remain untouched.

## Impact

### 1\. Theft of Unclaimed Yield

Strategy reserves could contain accumulated yield, harvested rewards, and profits. The bug silently drains these reserves to cover vault shortfalls. Users lose all accumulated profits over time.

### 2\. Temporary Freezing of Funds

Once reserves are depleted, all withdrawals requiring deallocation permanently fail:

```
Reserves: 0 WETH
vault.withdraw(100) → returns 98 WETH
require(98 >= 100)  REVERTS till reserves are updated
```

User funds become temp frozen in the external vault till reserves are updated.

## Mitigation

### Fix: Measure Balance Before and After Withdrawal

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
  Measure balance BEFORE withdrawal
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    // Approve and execute withdrawal
    TokenUtils.safeApprove(address(weth), address(vault), amount);
    vault.withdraw(amount, address(this), address(this));
    
    // Measure balance AFTER withdrawal
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    // Calculate actual amount received from vault
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    
    //  Emit event if vault returned less than requested
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    
    // Revert if vault didn't return full amount
    // This protects reserves from being used to cover vault shortfalls
    require(wethRedeemed >= amount, "Vault returned less than requested amount");
    
    // Approve Morpheus to take the amount
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

### Alternative: Slippage Tolerance

If the protocol wants to accept vault fees/slippage:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    TokenUtils.safeApprove(address(weth), address(vault), amount);
    vault.withdraw(amount, address(this), address(this));
    
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    
    // Option: Allow up to 0.5% slippage
    uint256 minAcceptable = amount * 9950 / 10000; // 99.5% of requested
    require(wethRedeemed >= minAcceptable, "Vault slippage exceeds tolerance");
    
    // Return ACTUAL amount received, not requested amount
    TokenUtils.safeApprove(address(weth), msg.sender, wethRedeemed);
    return wethRedeemed; // Return actual, not amount
}
```
## [I-01] Hardcoded Slippage in MYT Strategy

### Summary
The `slippageBPS` parameter is set once in the constructor with no mechanism to update it afterward. While stored as a mutable state variable, the absence of a setter function makes it effectively hardcoded, preventing the strategy from adapting to changing market conditions.

## Description

The `MYTStrategy` contract sets `slippageBPS` during deployment:

```solidity
constructor(address _myt, StrategyParams memory _params, address _permit2Address, address _receiptToken) {
    // ...
    slippageBPS = _params.slippageBPS;
    // No setter function exists to update this value
}
```

The variable is declared as a regular state variable (not immutable or constant):

```solidity
uint256 public slippageBPS;
```

This design suggests the intent was to make it updateable, but no setter function was implemented. The contract includes setters for other critical parameters (`setKillSwitch`, `setAdditionalIncentives`, `setPermit2Address`), making this omission inconsistent with the overall design pattern.

During market volatility, if actual slippage exceeds the hardcoded `slippageBPS` tolerance, deallocations will revert

> Note!!!
> 
> > AlchemixV3 deal with approx 20 strategy on multiple network so adaptive slippage is essential.

## Impact

When market conditions changes than conditions anticipated at deployment the following will happen:

1.  **Withdrawal failures**: Deallocations revert when actual market slippage exceeds the unchangeable `slippageBPS` threshold
2.  **Fund freezing**: Users cannot withdraw their assets despite having valid balances
3.  **Extended duration**: Funds remain inaccessible until market volatility decreases to acceptable levels, which can take hours or days
4.  **No recovery mechanism**: Even the contract owner cannot adjust slippage to restore functionality without full contract redeployment

> Note!!!
> 
> > In markets this is not temporary situation as markets evolve over time.

## Mitigation

Add an owner-controlled setter function with reasonable bounds to allow slippage adjustment:

```solidity

function setSlippageBPS(uint256 newSlippageBPS) external onlyOwner {
    require(newSlippageBPS <= 1000, "Slippage exceeds maximum (10%)");
    slippageBPS = newSlippageBPS;
}
```

This allows the strategy to adapt to market conditions while maintaining reasonable upper bounds to protect users from excessive or tight slippage tolerance


