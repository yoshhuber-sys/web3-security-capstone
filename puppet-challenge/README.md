# damn-vulnerable-defi-exploits
Personal Study Case of smart contract security research &amp; exploits

## Overview
This repository demonstrates a self project analysis and exploit of the **UnstoppableVault** challenge from [Damn Vulnerable DeFi v4](https://damnvulnerabledefi.xyz).  

The exercise simulates a security research scenario, highlighting:
- Vulnerability identification in smart contract logic.
- Practical exploit execution.
- Mitigation recommendations.

## Challenge Description
- The vault is a tokenized DeFi vault offering **flash loans**.
- A **monitor contract** checks the liveness of flash loans.
- Initial state:
  - Vault has 1,000,000 DVT tokens.
  - Player has 10 DVT tokens.
  - Flash loans work correctly.
- Goal:
  - Demonstrate how flash loans can be halted by external actions.
  - Analyze why this occurs and how to prevent it.

---

## Vulnerability Analysis

### Observations
1. **Vault Accounting Assumption**:  
   - The vault uses `totalSupply()` to check liquidity for flash loans.  
   - It assumes that `totalSupply()` always matches the actual ERC20 token balance (`token.balanceOf(address(vault))`).

2. **External Transfers**:  
   - Any direct transfer of tokens to the vault bypasses the `deposit()` function.
   - This breaks the `totalSupply()` assumption.

3. **Impact**:  
   - Flash loan monitor fails if the vault’s `balanceOf` exceeds `totalSupply()`.
   - The vault is paused automatically, blocking further flash loans.
   - Ownership is reverted to deployer, preventing immediate recovery by the attacker.

### Root Cause
- **Unsafe assumption in internal accounting**:
  - `flashLoan()` relies on `totalSupply()` to calculate available liquidity.
  - ERC20 `transfer()` can bypass vault functions, creating discrepancies.

### Security Lesson
- Always **validate external state changes**.
- Avoid assumptions that the ERC20 balance is immutable outside the contract.
- Consider using `ERC20` hooks (`_beforeTokenTransfer`) or restricting direct transfers to vault address.

---

## Exploit Implementation

**Step 1:** Transfer tokens directly to vault:

```solidity
token.transfer(address(vault), 1); // disrupt vault accounting
```

**Step 2:** Monitor detects flash loan failure:

```solidity
vm.prank(deployer);
vm.expectEmit();
emit UnstoppableMonitor.FlashLoanStatus(false);
monitorContract.checkFlashLoan(100e18);
```

**Step 3:** Result:
After sending just 1 token directly to the vault:

a. **Vault pauses automatically**  
   The flash loan monitor detects the discrepancy and triggers the vault pause.

b. **Ownership reverts to deployer**  
   The monitor contract restores ownership to ensure safe recovery.

c. **Flash loans are halted**  
   Any further flash loan attempts fail, demonstrating how external token transfers can break contract assumptions.

d. **Security Takeaway**  
   Even a single token transferred outside the `deposit()` function can disrupt vault accounting.  
   This highlights the importance of **guarding against unexpected external state changes** in smart contracts, especially in DeFi protocols relying on internal supply tracking.

```text
PS E:\Project\damn-vulnerable-defi> forge test --match-contract UnstoppableChallenge
Warning: This is a nightly build of Foundry. It is recommended to use the latest stable version. To mute this warning set `FOUNDRY_DISABLE_NIGHTLY_WARNING` in your environment.

[⠰] Compiling...
[⠃] Compiling 1 files with Solc 0.8.25
[⠊] Solc 0.8.25 finished in 4.41s
Compiler run successful!

Ran 2 tests for test/unstoppable/Unstoppable.t.sol:UnstoppableChallenge
[PASS] test_assertInitialState() (gas: 57303)
[PASS] test_unstoppable() (gas: 66791)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.08ms (584.70µs CPU time)

Ran 1 test suite in 33.86ms (2.08ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
