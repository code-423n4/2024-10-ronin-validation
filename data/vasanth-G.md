# Gas Optimization in katana-v3-contracts/src/core/KatanaV3Factory.sol

## Optimize Storage Access

[Line 83](https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/core/KatanaV3Factory.sol#L83)

Accessing storage variables is costly, so minimize the number of reads.

Example Change: Instead of reading feeAmountTickSpacing[fee] multiple times, store it in a local variable.

```solidity
int24 tickSpacing = feeAmountTickSpacing[fee];
require(tickSpacing != 0, "KatanaV3Factory: INVALID_FEE");
```







