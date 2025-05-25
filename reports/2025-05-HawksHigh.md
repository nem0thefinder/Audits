# Findings Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-teacher-wages-wrong-distribution)|Teacher Wages Wrong Distribution|High|
|[H-02](#h-02-students-can-graduate-without-receiving-all-weekly-reviews)|Students Can Graduate Without Receiving All Weekly Reviews|High|
|[H-03](#h-03-unhandled-60-of-bursary-funds)|Unhandled 60% of Bursary Funds|High|
|[M-01](#m-01-missing-disableinitializers-in-upgradeable-contracts)|Missing `disableInitializers` in upgradeable contracts|Medium|
|[M-02](#m-01-missing-disableinitializers-in-upgradeable-contracts)| No storage Gaps in upgradeable Contracts|Medium|
|[M-03](#m-03-storage-layout-mismatch-between-levelone--leveltwo)| Storage Layout mismatch between LevelOne & LevelTwo|Medium|
|[L-01](#l-01-missing-review-count-update-in-givereview) |Missing Review Count Update in `giveReview`|Low|
[L-02](#l-02-upgrade-allowed-before-sessionend)| Upgrade Allowed Before `sessionEnd`|Low|
|[L-03](#l-03-unbounded-loops-can-lead-to-out-of-gas-errors-in-contracts)| Unbounded loops can lead to out-of-gas errors in contracts|Low|





##  [H-01] Teacher Wages Wrong Distribution

### Description 
The current implementation of `graduateAndUpgrade` incorrectly distributes **35% of the bursary to each teacher**, rather than splitting **35% among all teachers**. This causes the total payout to exceed 100% of the bursary, resulting in a failure due to insufficient funds.

[LevelOne.sol#L302C8-L302C70](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/3a7251910c31739505a8699c7a0fc1b7de2c30b5/src/LevelOne.sol#L302C8-L302C70)

[LevelOne.sol#L308](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/3a7251910c31739505a8699c7a0fc1b7de2c30b5/src/LevelOne.sol#L308)

```solidity

    function graduateAndUpgrade(address _levelTwo, bytes memory) public onlyPrincipal {
             ...

 @>       uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        ...

        for (uint256 n = 0; n < totalTeachers; n++) {
@>            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

    ...
    }

```

### ProofOfConcept

```solidity
function test_Teacher_Wages_Distribution_Is_Wrong() public {
    // Initializing new Teacher
    address jack = makeAddr("Third_teacher");

    // Adding Teachers
    _teachersAdded();
    vm.prank(principal);
    levelOneProxy.addTeacher(jack);

    // Adding Students
    _studentsEnrolled();

    // Starting A session
    vm.prank(principal);
    levelOneProxy.startSession(70);

    // Assert that jack is added as a teacher
    assertEq(levelOneProxy.getTotalTeachers(),3);

    // Upgrading and Distributing wages
    levelTwoImplementation = new LevelTwo();
    levelTwoImplementationAddress = address(levelTwoImplementation);

    bytes memory data = abi.encodeCall(LevelTwo.graduate, ());

    vm.startPrank(principal);

    // This will revert as contract give each teacher 35% and we have 3 teachers
    // 35% * 3 = 105% so it will revert with insufficient funds error
    vm.expectRevert();
    levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);
}

```

### Impact 
`graduateAndUpgrade` will be DoS'ed with OOF Error since amount needed exceed `bursary` amount

### Mitigation 

* Transfer `payPerTeacher` instead of `totalTeacherShare`
``` solidity
uint256 totalTeacherShare = (bursary * TEACHER_WAGE) / PRECISION;
uint256 payPerTeacher = totalTeacherShare / listOfTeachers.length;
```

## [H-02] Students Can Graduate Without Receiving All Weekly Reviews

### Description 


Each student is expected to receive 4 weekly reviews during the session. However, there is:

1. **No validation** in `graduateAndUpgrade()` to check whether each student has received 4 reviews.

[LevelOne.sol#L294C1-L312C6](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/3a7251910c31739505a8699c7a0fc1b7de2c30b5/src/LevelOne.sol#L294C1-L312C6)

``` solidity


function graduateAndUpgrade(address _levelTwo, bytes memory) public onlyPrincipal {
 ...
@Missing> Check on students number of reviews
    }

```

2. An **incomplete implementation** of `giveReview()`, where `reviewCount[_student]` is never incremented, making it impossible to verify the number of reviews each student has received.

[LevelOne.sol#L277C1-L293C6](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/3a7251910c31739505a8699c7a0fc1b7de2c30b5/src/LevelOne.sol#L277C1-L293C6)

``` solidity
    function giveReview(address _student, bool review) public onlyTeacher {
      ...
@Missing> reviewCount[_student]+=1

        emit ReviewGiven(_student, review, studentScore[_student]);
    }
```
This enables the principal to prematurely upgrade the system **even if students haven't been fairly assessed**.

### Impact 

+ Invariant mentioned in the DOCS will break

``` md
Students must have gotten all reviews before system upgrade. System upgrade should not occur if any student has not gotten 4 reviews (one for each week)
```

### Mitigation 

1. Increment   `reviewCount[_student]` inside `giveReview`
2. Add check mechanims on students reviews count in `graduateAndUpgrade`

## [H-03] Unhandled 60% of Bursary Funds

### Description

In the current implementation of the `graduateAndUpgrade` function, only 40% of the `bursary` is distributed:

* 5% is sent to the principal.

* 35% is allocated to teachers (divided among them).

The remaining **60% of the bursary is left unaccounted for and permanently locked** in the contract with no mechanism to withdraw, reallocate, or utilize it. this will lead to a significant accumulation of inaccessible funds, reducing the efficiency and usability of the contract.

[LevelOne.sol#L302C8-L303C71](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/3a7251910c31739505a8699c7a0fc1b7de2c30b5/src/LevelOne.sol#L302C8-L303C71)

```solidity


    function graduateAndUpgrade(address _levelTwo, bytes memory) public onlyPrincipal {
...

 @>       uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
  @>      uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

...

```

### Impact 

60% of the `bursary` will be locked in the contract 

### Mitigation 

Implement a clear handling strategy for the remaining 60%


## [M-01] Missing `disableInitializers` in upgradeable contracts

### Description 

`levelOne`,`levelTwo` is missing `disableInitializers` call and due to the usage of a proxy upgradeable contract without calling this function in the constructor of the logic contract. This oversight introduces a severe risk, allowing potential attackers to initialize the implementation contract itself.

### Impact 

Attacker can re-initiallize the contract and hijack it

## Mitigation 
``` diff
+	constructor() external {
+		disableInitializers()
+	}
```

## [M-02] No storage Gaps in upgradeable Contracts

### Description 

For upgradeable contracts, there must be storage gap to "allow developers to freely add new state variables in the future without compromising the storage compatibility with existing deployments". Otherwise it may be very difficult to write new implementation code. Without storage gap, the variable in child contract might be overwritten by the upgraded base contract if new variables are added to the base contract. This could have unintended and very serious consequences to the child contracts.

### Impact 

Potential storage collison would happen if devs decided to add new variables 

### Mitigation 
+ Recommend adding appropriate storage gap at the end of upgradeable contracts such as the below.

```solidity
uint256[50] private __gap;
```
## [M-03] Storage Layout mismatch between LevelOne & LevelTwo

### Description 

The following table shows the missmatch between L1,L2 Storage Layout 

| **Slot** | **Variable**               | **Type**                      | **Description**                             | **LevelOne (V1)** | **LevelTwo (V2)** |
| -------- | -------------------------- | ----------------------------- | ------------------------------------------- | ----------------- | ----------------- |
| 0        | `principal`                | `address`                     | Address of the principal                    | ✅ (address)       | ✅ (address)       |
| 1        | `inSession`                | `bool`                        | Boolean indicating if the session is active | ✅ (bool)          | ✅ (bool)          |
| 2        | `schoolFees`               | `uint256`                     | The school fees                             | ✅ (uint256)       | ❌ (Removed)       |
| 3        | `reviewTime`               | `uint256 (constant)`          | Duration of the review period (immutable)   | ✅ (constant)      | ❌ (Not present)   |
| 4        | `sessionEnd`               | `uint256`                     | Timestamp when the session ends             | ✅ (uint256)       | ✅ (uint256)       |
| 5        | `bursary`                  | `uint256`                     | Amount of bursary                           | ✅ (uint256)       | ✅ (uint256)       |
| 6        | `cutOffScore`              | `uint256`                     | Cut-off score for eligibility               | ✅ (uint256)       | ✅ (uint256)       |
| 7        | `isTeacher` (mapping)      | `mapping(address => bool)`    | Mapping to track if an address is a teacher | ✅ (mapping)       | ✅ (mapping)       |
| 8        | `isStudent` (mapping)      | `mapping(address => bool)`    | Mapping to track if an address is a student | ✅ (mapping)       | ✅ (mapping)       |
| 9        | `studentScore` (mapping)   | `mapping(address => uint256)` | Mapping to store student scores             | ✅ (mapping)       | ✅ (mapping)       |
| 10       | `reviewCount` (mapping)    | `mapping(address => uint256)` | Mapping to store review count               | ✅ (mapping)       | ❌ (Not present)   |
| 11       | `lastReviewTime` (mapping) | `mapping(address => uint256)` | Mapping to store last review time           | ✅ (mapping)       | ❌ (Not present)   |
| 12       | `listOfStudents`           | `address[]`                   | Array of student addresses                  | ✅ (array)         | ✅ (array)         |
| 13       | `listOfTeachers`           | `address[]`                   | Array of teacher addresses                  | ✅ (array)         | ✅ (array)         |
| 14       | `TEACHER_WAGE`             | `uint256 (constant)`          | Percentage of teacher wage (immutable)      | ✅ (constant)      | ❌ (Not present)   |
| 15       | `PRINCIPAL_WAGE`           | `uint256 (constant)`          | Percentage of principal wage (immutable)    | ✅ (constant)      | ❌ (Not present)   |
| 16       | `PRECISION`                | `uint256 (constant)`          | Constant value used for precision           | ✅ (constant)      | ✅ (constant)      |
| 17       | `usdc`                     | `IERC20`                      | The USDC token address                      | ✅ (IERC20)        | ✅ (IERC20)        |
| 10       | `TEACHER_WAGE_L2`          | `uint256 (constant)`          | Percentage of teacher wage (immutable)      | ❌ (Not present)   | ✅ (constant)      |
| 11       | `PRINCIPAL_WAGE_L2`        | `uint256 (constant)`          | Percentage of principal wage (immutable)    | ❌ (Not present)   | ✅ (constant)      |

** Key Differences
**
1. **Removed Variables:**

   * `schoolFees`, `reviewTime`, `reviewCount`, and `lastReviewTime` are removed in `LevelTwo`, which can lead to data corruption if the proxy is not handled correctly.

2. **Reordered Variables:**

   * The positions of `sessionEnd`, `cutOffScore`,`bursary` and other variables have changed between versions. This could cause misalignment of storage in the proxy.

3. **New Variables:**

   * `TEACHER_WAGE_L2` and `PRINCIPAL_WAGE_L2` are introduced in `LevelTwo`, potentially causing confusion or errors if not correctly handled during the upgrade.


### Impact 

* This invariant would break as `bursary` storage slot will mismatch
```md
remaining 60% should reflect in the bursary after upgrade

```

### Mitigation 

To ensure compatibility and prevent data corruption when upgrading, always maintain the same storage layout across different contract versions or properly migrate the state variables.

## [L-01] Missing Review Count Update in `giveReview`

### Description 

The `giveReview` function allows teachers to submit weekly reviews on students. However, the function **does not increment** **`reviewCount[_student]`**, even though it checks that the student hasn’t been reviewed more than 5 times.

[LevelOne.sol#L281](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/3a7251910c31739505a8699c7a0fc1b7de2c30b5/src/LevelOne.sol#L281)

  ``` solidity 
    function giveReview(address _student, bool review) public onlyTeacher {
      ...
     @> require(reviewCount[_student] < 5, "Student review count exceeded!!!");
       ...
     @Missing> reviewCount[_student]+=1 
    }

 ```

### Impact 

* A teacher can submit infinite reviews for a student since reviewCount\[\_student] is never incremented.

* The require(reviewCount\[\_student] < 5, ...) check becomes meaningless.

### Mitigation 

Increment `reviewCount[_student]`


## [L-02] Upgrade Allowed Before `sessionEnd`

### Description 

The system currently permits a contract upgrade from `LevelOne` to `LevelTwo` **even if the** **`sessionEnd`** **has not been reached**, which violates the intended design rule that upgrades should only occur after the session is complete.

According to the system specification, upgrades should only be allowed **after the session has ended**. However, in the current implementation, there is **no guard or check** in place to prevent upgrades during an active session. This allows a principal to prematurely upgrade the system, potentially bypassing session-related constraints or logic.

### Impact 

Protocol Can Upgrade from `LevelOne` to `LevelTwo` while session is active which violate the last invariant in the docs 

[LastInvariant](https://github.com/CodeHawks-Contests/2025-05-hawk-high/tree/main?tab=readme-ov-file#invariants)

### ProofOfConcept 

``` solidity

function test_can_before_sessionEnd() public schoolInSession {
    levelTwoImplementation = new LevelTwo();
    levelTwoImplementationAddress = address(levelTwoImplementation);

    bytes memory data = abi.encodeCall(LevelTwo.graduate, ());
    
    // Assert that session is still in progress
    assert(block.timestamp < levelOneProxy.sessionEnd());

    // Attempt to upgrade before session end
    vm.prank(principal);
    levelOneProxy.graduateAndUpgrade(levelTwoImplementationAddress, data);

    // Interact with upgraded contract
    LevelTwo levelTwoProxy = LevelTwo(proxyAddress);
}

```

### Mitigation 
+ Add similar check in `graduateAndUpgrade` function

``` solidity
require(block.timestamp >= sessionEnd, "Cannot upgrade before session ends");
```

## [L-03] Unbounded loops can lead to out-of-gas errors in contracts

### Description

**Instances**

* `removeTeacher` Function in `LevelOne.sol` Itereates over All registered teachers

* `expel` Function in `LevelOne.sol` iterates over All registered students

* `graduateAndUpgrade` Function in `LevelOne.sol` iterates over All registered teachers

As these arrays grows looping over them the gas consumed will incerease


### Impact 
These Instances can be DoS'ed with OOG Error which lead to the following Issues 

+ Principal Can't Remove Teachers
+ Principal Cant expel any students before upgrading to level two
+ Principal can't pay teachers or himself
+ Principal can't upgrade to level two

### ProofOfConcept 

If we tested this among one of the instances such as `removeTeacher` Function we conclude that removing the last teacher consumes approximately 10 times more gas than removing the first one. This pattern highlights the linear gas complexity of the operation and should be taken into account when analyzing similar design choice  elsewhere in the contract.

``` solidity
function test_gas_RemoveFirstTeacher() public {
    // Arrange: Add 100 teachers
    for (uint256 i = 1; i <= 100; i++) {
        vm.prank(principal);
        levelOneProxy.addTeacher(address(uint160(i)));
    }

    // Confirm initial state
    assertEq(levelOneProxy.getTotalTeachers(), 100);

    // Act: Remove first teacher and measure gas
    vm.startPrank(principal);
    uint256 gasStart = gasleft();
    levelOneProxy.removeTeacher(address(uint160(1)));
    uint256 gasUsed = gasStart - gasleft();
    vm.stopPrank();

    // Assert post state
    assertEq(levelOneProxy.getTotalTeachers(), 99);

    // Log
    console2.log("Gas used to remove first teacher:", gasUsed); // 5643
}

```

``` solidity

function test_gas_RemoveLastTeacher() public {
    // Arrange: Add 100 teachers
    for (uint256 i = 1; i <= 100; i++) {
        vm.prank(principal);
        levelOneProxy.addTeacher(address(uint160(i)));
    }

    // Confirm initial state
    assertEq(levelOneProxy.getTotalTeachers(), 100);

    // Act: Remove last teacher and measure gas
    vm.startPrank(principal);
    uint256 gasStart = gasleft();
    levelOneProxy.removeTeacher(address(uint160(100)));
    uint256 gasUsed = gasStart - gasleft();
    vm.stopPrank();

    // Assert post state
    assertEq(levelOneProxy.getTotalTeachers(), 99);

    // Log
    console2.log("Gas used to remove last teacher:", gasUsed); // 45639 10x
}
```





