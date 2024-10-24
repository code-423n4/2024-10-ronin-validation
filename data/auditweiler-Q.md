The `V3Migrator`'s `receive()` override reports an incorrect `revert` message:

```solidity
receive() external payable {
@> require(msg.sender == WETH9, "Not WETH9");
}
```

https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/periphery/V3Migrator.sol#L32C3-L34C4


This is incorrect because the native wrapped token is `WRON`:

```solidity
v3migrator = address(new V3Migrator(factory, wron, nonfungiblePositionManager));
console.log("V3Migrator deployed:", v3migrator);
```

https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/script/DeployKatanaV3Periphery.s.sol#L54C5-L55C53

This can persuade users to transact using incorrect currencies.