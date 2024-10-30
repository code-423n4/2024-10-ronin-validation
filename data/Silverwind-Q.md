## Missing NatSpec comment in *_updatePosition*
The NatSpec comment for the *_updatePosition* function does not include the parameter *liquidityDelta* which affects the position’s liquidity.

https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/core/KatanaV3Pool.sol#L342

# Mitigation
Update the NatSpec comment to accurately reflect the function's implementation with including the following line:

```solidity
/// @param liquidityDelta the change in liquidity for the position. A positive value increases liquidity, while a negative value decreases it.
```
