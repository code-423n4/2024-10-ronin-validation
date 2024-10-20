## Summary

`Pragma solidity =0.7.6` contains several bugs known. One of them that affects this codebase, is the `MissingSideEffectsOnSelectorAccess`, see in more detail in this link: https://00xsev.github.io/solidityBugsByVersion/.

## Description

When accessing the `.selector` member on an expression with side-effects, like an assignment, a function call or a conditional, the expression would not be evaluated in the legacy code generation.
Example of `.selector` in `KatanaV3Pool::balance0`: https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/core/KatanaV3Pool.sol#L148-L153.

## Severity

Low Risk

## Recommendations

Use the newest version of solidity.