## Issues found
|Severtity|Number of issues found|
| ------- | -------------------- |
| High    | 0                    |
| Medium  | 0                    |
| Low     | 0                    |
| Info    | 1                    |
| Gas     | 0                    |
| Total   | 1                    |

## Findings

## Informational

### [I-1]: `NonfungibleTokenPositionDescriptor:nativeCurrencyLabel`   Consider renaming the "b" variable to "currencyLabelBytes" to make it clearer
```solidity
bytes memory b = new bytes(len);
    for (uint256 i = 0; i < len; i++) {
      b[i] = nativeCurrencyLabelBytes[i];
    }
    return string(b);
```