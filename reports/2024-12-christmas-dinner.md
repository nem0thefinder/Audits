# Findings Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-christmasdinnernonreentrant-modifier-is-not-implemented-correctly)|`christmasDinner::nonReentrant` modifier is not implemented correctl|HIGH|
|[H-02](#h-02-host-can-prevent-participants-from-refunding-their-deposits)|`Host` can prevent participants from refunding their `Deposits|HIGH|
|[H-03](#h-03-eth-deposited-fees-stuck-in-contract-due-to-missing-withdrawal-mechanism)|ETH Deposited Fees Stuck in Contract Due to Missing Withdrawal Mechanism|HIGH|
|[M-01](#m-01-christmasdinnersetdeadline-will-never-revert)|`christmasDinner::setDeadline` will never revert |MEDIUM|
|[M-02](#m-02-user-can-call-christmasdinnerrefund-and-still-particpate-in-the-event)|User can call `christmasDinner::refund` and still particpate in the event |MEDIUM|
|[M-03](#m-03-manual-participant-status-update-required-for-eth-depositors)|Manual `participant` Status Update Required for ETH Depositors|MEDIUM|
|[M-04](#m-04--use-of-transfer-opcode-to-send-eth)|use of "transfer" opcode to send ETH|MEDIUM|
|[L-01](#l-01-christmasdinnerdeposit-has-a-missing-functionality-as-mentioned-in-the-function-natspac)|`ChristmasDinner::deposit` Has a Missing functionality as mentioned in the function natspac|LOW|
|[L-02](#l-02-unauthorized-modification-of-user-participation-status)|Unauthorized Modification of User Participation Status|LOW|






## [H-01] `christmasDinner::nonReentrant` modifier is not implemented correctly             

### Description

The `christmasDinner::nonReentrant` modifier is not implemented correctly, leaving the contract vulnerable to reentrancy attacks.

[ChristmasDinner.sol#L77](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L77)

[ChristmasDinner.sol#L43](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L43)

 the `locked` state variable is reset to `false` after executing the function logic (`_`). This implementation fails to adequately lock the function during its execution, as reentrancy protection should ensure the `locked` state remains `true` until the function execution is entirely complete.

### Impact

* Vulnerability to Reentrancy Exploits: Functions using this modifier are susceptible to reentrancy attacks, which can allow malicious actors to repeatedly call the function before the first invocation completes.

### Mitigation

* Update the `nonReentrant` modifier to set `locked = true` before the function execution and reset it to `false` after completion. A correct implementation would look like this:

```diff

modifier nonReentrant() {
    require(!locked, "No re-entrancy");
+    locked = true; // Lock the function before execution
    _;
    locked = false; // Unlock the function after execution
}

```

## [H-02] `Host` can prevent participants from refunding their `Deposits`            



### Description

The `Host` can prevent participants who deposited in various ERC20 tokens from refunding their deposits.

[ChristmasDinner.sol#L194](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L194)

[ChristmasDinner.sol#L137](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L137)


When the host calls the `christmasDinner::withdraw` function, it will wipe the contract's balance in different tokens. Consequently, when a user attempts to refund their deposit, the transaction will revert because the contract has no tokens left.

### PoC

* use this test in `christmasDinnerTest.t.sol`
  ```Solidity

  function testHostCanPreventUsersFromRefunding() public {
      vm.prank(user1);
      cd.deposit(address(usdc),1e18);
      vm.prank(user2);
      cd.deposit(address(weth),1e18);
      assertEq(usdc.balanceOf(address(cd)),1e18);
      assertEq(weth.balanceOf(address(cd)),1e18);
     vm.warp(2 days); 
      // user 1 after 2 days want to refund his deposit but he can't 
      // Because deployer withdraw all tokens 
      vm.prank(deployer);
      cd.withdraw();
      vm.expectRevert();
      vm.prank(user1);
      cd.refund();

  }
  ```

### Impact

* Users cannot claim refunds using the `christmasDinner::refund` function and recover their deposits.


### Mitigation 

* The `christmasDinner::withdraw` function should only be callable after the `deadline` has passed.

```diff
    function withdraw() external onlyHost {
+        require(block.timestamp > deadline);
        address _host = getHost();
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
    }
```

## [H-03] ETH Deposited Fees Stuck in Contract Due to Missing Withdrawal Mechanism            



### Description

The deposited fees in ETH are stuck in the contract and cannot be withdrawn by the host due to a missing withdrawal mechanism. This issue prevents the host from accessing ETH funds deposited into the protocol.
The contract allows users to deposit ETH as part of their participation fees. However, there is no function implemented to enable the host to withdraw the ETH funds accumulated in the contract. This results in the ETH becoming permanently locked, rendering it inaccessible to the host or any other authorized party.

### PoC

* use this test in `christmasDinnerTest.t.sol`

  ```Solidity
      function testDepositedETHIsStuck() public {
          address payable _cd = payable(address(cd));
          vm.deal(user1, 10e18);
          vm.prank(user1);
          (bool sent, ) = _cd.call{value: 1e18}("");
          require(sent, "transfer failed");
          assertEq(user1.balance, 9e18); // Changed user dealing amount in setUp to 10e18
          assertEq(address(cd).balance, 1e18);
          vm.prank(deployer);
          // this function must wipe the contract 
          cd.withdraw();
          // ETH balance remain the same
          assertEq(address(cd).balance, 1e18);

      }

  ```

### Impact

* Locking of funds Deposited in native ETH


### Mitigation 

* Implement a \`withdraw\` function for the host to transfer the collected ETH to their address.
  ```diff
      function withdraw() external onlyHost {
          address _host = getHost();
          i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
          i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
          i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
  +        (bool success,)=payable(msg.sender).call{value:address(this).balance}("");
  +       require(success);
      }
  ```


## [M-01] `christmasDinner::setDeadline` will never revert             

### Description

The `christmasDinner::setDeadline` function is intended to revert if the `deadline` has already been set. However, due to a flaw in the current implementation, the revert statement will never be executed.\

[ChristmasDinner.sol#L182](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L182)

[ChristmasDinner.sol#L42](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L42)


The `christmasDinner::setDeadline` function uses a boolean variable, `deadlineSet`, to determine whether the `deadline` has already been set. If it is set, the function is supposed to revert. Otherwise, it sets the `deadline` to `block.timestamp + _days * 1 days`. However, since the `deadlineSet` variable is never updated to `true` after setting the `deadline`, the function does not revert and allows the `deadline` to be reset multiple times.

### Impact

* The host can set the event `deadline` multiple times, potentially causing inconsistencies in the event scheduling.

### PoC

* use this test in `christmasDinnerTest.t.sol`\`

```solidity
        function testUserCanSetDeadLineMultipleTimes() public {
        vm.startPrank(deployer);
        cd.setDeadline(8 days);
        cd.setDeadline(8 days);
        vm.stopPrank();
    }
```

![](data:image/png;base64,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)

### Mitigation 

* Update the function to set the `deadlineSet` boolean to `true` immediately after the `deadline` is assigned.

```diff
    function setDeadline(uint256 _days) external onlyHost {
        //@audit medium deadlineSet bool is never set to true
        if (deadlineSet) {
            revert DeadlineAlreadySet();
        } else {
            deadline = block.timestamp + _days * 1 days;
            emit DeadlineSet(deadline);
        }
+ deadlineSet = true;
    }

```

## [M-02] User can call `christmasDinner::refund` and still particpate in the event             



### Description

A user can call `christmasDinner::refund` to retrieve their deposit but still participate in the event.

[ChristmasDinner.sol#L137](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L137)

When a user calls `christmasDinner::refund`, it sends the user all deposited amounts in various tokens. However, it does not update the user's participation status, allowing them to remain marked as a participant in the event despite receiving a refund.

### PoC

* use this test in `christmasDinnerTest.t.sol`\`

```solidity
    function testUserRefundAndStillParticpateInEvent() public {
        vm.prank(user1);
        cd.deposit(address(weth),1e18);
        assertEq(weth.balanceOf(user1), 9e18); // i changed user dealing amount in setup to 10e18
        assertEq(weth.balanceOf(address(cd)), 1e18);
        vm.prank(user1);
        cd.refund();
        assertEq(weth.balanceOf(user1), 10e18);
        assertEq(weth.balanceOf(address(cd)), 0);
        assertEq(cd.getParticipationStatus(user1),true);
    

}
```


![](data:image/png;base64,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)

### Impact

* A user can refund all their deposits and still participate in the event, potentially disrupting the event's integrity and fairness.


### Mitigation 

* Update the user participation status to `false` after processing the refund to prevent participation after the refund is issued.
* We can also add mapping to check whether user called refund or not and add it as check in `changeParticipationStatus` function 

```diff
    function refund() external nonReentrant beforeDeadline {
        address payable _to = payable(msg.sender);
        _refundERC20(_to);
        _refundETH(_to);
+       participant[msg.sender] = false;
        emit Refunded(msg.sender);
    }
```

## [M-03] Manual `participant` Status Update Required for ETH Depositors            



### Description

Depositors in `ETH` must manually change their participation status, as it is not set automatically.\
[ChristmasDinner.sol#L205](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L205)

When a user deposits ETH, they trigger the `christmasDinner::receive` function. However, unlike the `christmasDinner::deposit` function, the `receive` function does not set `participant[msg.sender] = true`. As a result, the user needs to manually update their participation status via the `christmasDinner::changeParticipationStatus` function to mark their status as true.

### PoC

* use this test in `christmasDinnerTest.t.sol`

```solidity
    function testDepostiorsInETHMustChangeStatusManually() public {
        address payable _cd = payable(address(cd));
        vm.deal(user1, 10e18);
        vm.prank(user1);
        (bool sent, ) = _cd.call{value: 1e18}("");
        require(sent, "transfer failed");
        vm.startPrank(user1);
        // after the deposit status should automatically set to true
        assertEq(cd.getParticipationStatus(user1),false);
        // user must manually change his status
        cd.changeParticipationStatus();
        assertEq(cd.getParticipationStatus(user1),true);

}
```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAApwAAABpCAIAAAAY6WRxAAAgAElEQVR4Ae1du5LcRrLdH2AEgzHUhbneypZBUxEy5Mgfg7LlyFyTxlj8iWlz3Qk6inVpjCFv25d+gP9xI7Mqn0g8uhv94PAgGAQqKx8nTxUqATS65x/DwvbwtJft6WFW1zQXFGe9oBMMgAEwAAbAABg4joF/HGcGKzAABsAAGAADYODWGEBRv7URAR4wAAbAABgAA0cygKJ+JHEwAwNgAAyAATBwawygqN/aiAAPGAADYAAMgIEjGbhWUee36p539wL7fve8bxveshNODtv/9uurzx/43+93P9Wm3338ves8vksad58+vP74ryREEwyAATAABr4uBlYUdXur3RXhU7PMRb35o9KOon4Mt+/efP7w5rdVllTaUdRXUQUlMAAGwMBXxsBCUecbaK3l97snu7c+R6Io6key+tPPrz//utIWRX0lUVADA2AADHx1DMwXdbqfru6c7eZ93yr+w9P+acdP0J93D30/DPe75+fdvSp3TzNP2kdF3XQVx8MTxdSOCl4fhUozRJBGjXNqLO8+tafcH15JHb37ZHfJoWTaI3H3cLsSmk93D21CCTQMQykcyGco6pOQhiEglCS9vsiwBwNgAAyAga+MgdmiXtd0KoVy88619elh4MId989c1Pd70U3OpKAGvpLQmVhQDtWvNUwa3PTGWLNB0usACUd7wTnvcrkiWsmkQjv6eLsSvn38oM/D9bNt8+NSGwtJ0j9Kl0sNvizwRTpZpWZz//bRrktcQByCATAABsDA18TA4UXdVVpKlAsjFXWu836fKqjqNHqkoAayopBu/+1H7JxruU7IEYKvga805Oqj42w7cSrhZM/2oZE8ttvceE/c7p7182wtmVqevY9K+O6Nd/jbr68+/TxMBOISHu7Iu/OT79RR1P0w4RgMgAEw8JUycFRR10p5aFF3RbqsnUFIjbiF64ZlwuUywGuOItBlQyn0VuGYbqz55lhfF69ui8s31yohfRwuN9ntgIv6MAzjQFPC0x+/o6iHMUYDDIABMPB1MjBb1LmsuptlTjHdqXNT78KljtI+FUvVaUSFQircBWFoiAbff/uLCusYHQkY3xGcSkP2rBga3jQfU4Vudb0q6vTht96+q2khXH7HzQKpn2GIwpPv1J1nHIIBMAAGwMDXysBs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)

### Impact

* If the user does not manually update their status, they will be considered a `Funder` instead of a `Participant`.


### Mitigation

```diff
    receive() external payable {
+      participant[msg.sender] = true
        etherBalance[msg.sender] += msg.value;
        emit NewSignup(msg.sender, msg.value, true);
    }
```

## [M-04]  use of "transfer" opcode to send ETH            


### Description

in `chirstmasDinner::_refundETH` the `.transfer` opcode is used to handle ETH transfer, it does this by forwarding a fixed amount of 2300 gas. This is dangerous because

-  If the recipient is a  a multisig safe, with a `receive`/`fallback` function which requires >2300 gas, e.g safes that execute extra logic in the `receive`/`fallback` function, the transfer function will always fail for them due to out of gas errors.



### Impact

- `christamsDinner::refund` function will always revert and user can't refund his deposit 



### Mitigation

Use the ".call" opcode instead and follow CEI pattern 




## [L-01] `ChristmasDinner::deposit` Has a Missing functionality as mentioned in the function natspac            



### Description 

The `ChristmasDinner::deposit` function lacks an essential feature outlined in its natspac documentation.

 [ChristmasDinner.sol#L108](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L108)

 [ChristmasDinner.sol#L115](https://github.com/Cyfrin/2024-12-christmas-dinner/blob/9682dcc306db935a2511e1eb8280d17ef01e9004/src/ChristmasDinner.sol#L115)



The `ChristmasDinner::deposit` function lacks an essential feature outlined in its natspac documentation. According to the documentation, the function should enable users to sign up other users. However, the current implementation does not support this behavior since all the logic depends solely on `msg.sender`, preventing the caller from specifying a different address.


### Mitigation

* Update the function to accept an `address` parameter to allow callers to specify the address of the user being signed up.

## [L-02] Unauthorized Modification of User Participation Status             

### Description

A user can modify their `participationStatus` without calling the `christmasDinner::deposit` function, bypassing the intended workflow and gaining unauthorized access to the event.

The `christmasDinner::changeParticipationStatus` function can be called directly by any user, bypassing the `christmasDinner::deposit` function. This allows users to modify their `participationStatus` and participate in the event as depositors without make a deposit or follow the protocol user flow.

### Impact

* Users can participate in the event for free.
* Unauthorized users can become eligible to be the new host.

### PoC 

* use this test in `christmasDinnerTest.t.sol`

  ```Solidity
  function testUserCanChangeStatusWithoutCallingDeposit() public { 
      assertEq(cd.getParticipationStatus(user1),false);
      vm.prank(user1);
      cd.changeParticipationStatus();
      assertEq(cd.getParticipationStatus(user1),true);
  }

  ```

### Mitigation 

* There are ton of ideas to mitigate this such as :

1. create a mapping to track people who called `deposit` function and add it as condition in `changeParticipationStatus`
2. Make a condition that the balance of caller > 0


