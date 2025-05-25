# Findings Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-teacher-wages-wrong-distribution)|Teacher Wages Wrong Distribution|High|


##  [H-01] Teacher Wages Wrong Distribution

### Describtion 
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

## Impact 
`graduateAndUpgrade` will be DoS'ed with OOF Error since amount needed exceed `bursary` amount

## Mitigation 

* Transfer `payPerTeacher` instead of `totalTeacherShare`
``` solidity
uint256 totalTeacherShare = (bursary * TEACHER_WAGE) / PRECISION;
uint256 payPerTeacher = totalTeacherShare / listOfTeachers.length;
```


