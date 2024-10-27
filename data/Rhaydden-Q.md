# QA for Ronin


## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-payer-could-be-mismatched-in-v3swapexactinput-when-using-contract-balance) | Payer could be mismatched in `V3SwapExactInput` when using contract balance |
| [QA-02](#qa-02-incorrect-token-ratio-priority-logic-affects-position-representation) | Incorrect token ratio priority logic affects position representation |
| [QA-03](#qa-03-missing-publicly-available-tokens-in-whitelist-query-function-due-to-mismatched-authorization-logic) | Missing publicly available tokens in whitelist query function due to mismatched authorization logic |
| [QA-04](#qa-04-global-authorization-override-bypasses-token-specific-permissions) | Global authorization override bypasses token-specific permissions |
| [QA-05](#qa-05-integer-division-precision-loss-can-lead-to-0-token-migrations-in-v3migrator) | Integer division precision loss can lead to 0 token migrations in `V3Migrator` |
| [QA-06](#qa-06-hardcoded-init-code-hash-in-pairfor-creates-inflexibility) | Hardcoded init code hash in `pairFor` creates inflexibility |


## [QA-01] Payer could be mismatched in `V3SwapExactInput` when using contract balance


The problem here is related to the handling of the `Constants.CONTRACT_BALANCE` flag


```solidity
function v3SwapExactInput(
  address recipient,
  uint256 amountIn,
  uint256 amountOutMinimum,
  bytes calldata path,
  address payer
) internal {
  // use amountIn == Constants.CONTRACT_BALANCE as a flag to swap the entire balance of the contract
  if (amountIn == Constants.CONTRACT_BALANCE) {
    address tokenIn = path.decodeFirstToken();
    amountIn = ERC20(tokenIn).balanceOf(address(this));
  }

  // ... SNIP
}
```


When [the function calls `_swap`](https://github.com/ronin-chain/katana-operation-contracts/blob/27f9d28e00958bf3494fa405a8a5acdcd5ecdc5d/src/aggregate-router/modules/katana/v3/V3SwapRouter.sol#L155-L158), it passes `payer` as the address that should provide the tokens:

```solidity
   function _swap(int256 amount, address recipient, bytes calldata path, address payer, bool isExactIn)
     private
     returns (int256 amount0Delta, int256 amount1Delta, bool zeroForOne)
   {
     // ... SNIP
   }
```
This `payer` is [then used in the `katanaV3SwapCallback` function](https://github.com/ronin-chain/katana-operation-contracts/blob/27f9d28e00958bf3494fa405a8a5acdcd5ecdc5d/src/aggregate-router/modules/katana/v3/V3SwapRouter.sol#L67):

```solidity
   function katanaV3SwapCallback(int256 amount0Delta, int256 amount1Delta, bytes calldata data) external {
     // ... 
     (, address payer) = abi.decode(data, (bytes, address));
     // ...
     payOrPermit2Transfer(tokenIn, payer, msg.sender, amountToPay);
     // ... SNIP
   }
```
The point of concern here's that after setting `amountIn` to the contract's balance of the input token, the function continues to use `payer` as the address that will be paying for the swap. This could cause several issues.

If `payer` is set to an address other than `address(this)`, the `payOrPermit2Transfer` function will try to transfer tokens from that address, even though the amount was determined based on this contract's balance. This will cause failed transactions if the payer doesn't have sufficient balance, even though the contract itself has the tokens.

### Recommendation
Consider updating the payer when `Constants.CONTRACT_BALANCE` is used:

```diff
function v3SwapExactInput(
  address recipient,
  uint256 amountIn,
  uint256 amountOutMinimum,
  bytes calldata path,
  address payer
) internal {
  if (amountIn == Constants.CONTRACT_BALANCE) {
    address tokenIn = path.decodeFirstToken();
    amountIn = ERC20(tokenIn).balanceOf(address(this));
+    payer = address(this); // Update payer to be this contract
  }
```



## [QA-02] Incorrect token ratio priority logic affects position representation

The issue starts in [`NonfungibleTokenPositionDescriptor.sol`](https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/periphery/NonfungibleTokenPositionDescriptor.sol#L92-L110) where the `tokenRatioPriority` function assigns priorities to tokens that determine their ordering in pairs:

```solidity
function tokenRatioPriority(address token, uint256 chainId) public view returns (int256) {
    if (token == WRON) {
        return TokenRatioSortOrder.DENOMINATOR;  // Returns -100
    }
    if (chainId == 2020) {
        if (token == USDC) {
            return TokenRatioSortOrder.NUMERATOR_MOST;    // Returns 300
        } else if (token == WETH) {
            return TokenRatioSortOrder.NUMERATOR_MORE;    // Returns 200
        } else if (token == WBTC) {
            return TokenRatioSortOrder.DENOMINATOR_MOST;  // Returns -300
        } else {
            return 0;
        }
    }
    return 0;
}
```

These priorities are defined in [`TokenRatioSortOrder.sol`](https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/periphery/libraries/TokenRatioSortOrder.sol#L5-L13):
```solidity
library TokenRatioSortOrder {
    int256 constant NUMERATOR_MOST = 300;
    int256 constant NUMERATOR_MORE = 200;
    int256 constant NUMERATOR = 100;

    int256 constant DENOMINATOR_MOST = -300;  // WBTC gets this value
    int256 constant DENOMINATOR_MORE = -200;
    int256 constant DENOMINATOR = -100;       // WRON gets this value
}
```

The [`flipRatio`](https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/periphery/NonfungibleTokenPositionDescriptor.sol#L88-L90) function uses these priorities to determine token ordering:
```solidity
function flipRatio(address token0, address token1, uint256 chainId) public view returns (bool) {
    return tokenRatioPriority(token0, chainId) > tokenRatioPriority(token1, chainId);
}
```

This creates a logical contradiction where:
1. WBTC is assigned `DENOMINATOR_MOST (-300)` which means it should be the strongest denominator
2. But having the lowest priority value means it will never be chosen as the numerator in any pair
3. WRON with `DENOMINATOR (-100)` has a higher priority than WBTC, contradicting the intended ordering

This contradicts the naming convention where `DENOMINATOR_MOST` suggests `WBTC` should be the strongest denominator.

As a result, in`NonfungiblePositionManager.sol`, position metadata becomes inconsistent through the [`tokenURI`](https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/periphery/NonfungiblePositionManager.sol#L237-L240) function:
```solidity
function tokenURI(uint256 tokenId) public view override(ERC721, IERC721Metadata) returns (string memory) {
    require(_exists(tokenId));
    return INonfungibleTokenPositionDescriptor(_tokenDescriptor).tokenURI(this, tokenId);
}
```

 Also, position storage and interpretation become ambiguous whuch makes us to question which token is really token0 and token1:
```solidity
struct Position {
    // ...SNIP
    uint256 feeGrowthInside0LastX128;  // Which token is really token0?
    uint256 feeGrowthInside1LastX128;  // Which token is really token1?
    uint128 tokensOwed0;
    uint128 tokensOwed1;
}
```

In DEX interfaces, token pairs are displayed as "Price of Token A in terms of Token B", where:

- Token B is the quote token (denominator/price currency)
- Token A is the base token (numerator/asset being priced)
Using the WBTC/WRON example:

Current Implementation:

```
// WBTC priority: DENOMINATOR_MOST (-300)
// WRON priority: DENOMINATOR (-100)
// Since -300 < -100, WBTC becomes token0 (base) and WRON becomes token1 (quote)
// Display: "1 WBTC = X WRON"
```

This means users would see prices like:

- "1 WBTC = 50,000 WRON"

Now, most users are accustomed to seeing prices in terms of the native token (WRON). Having WRON as the quote token means larger, less intuitive numbers
Moreover, standard practice is "1 WRON = 0.00002 WBTC" rather than "1 WBTC = 50,000 WRON"


### Recommended Mitigation Steps

Fix for this is easy and could be done in 2 ways:

Consider either reversing the comparison in `flipRatio`:
```diff
function flipRatio(address token0, address token1, uint256 chainId) public view returns (bool) {
-   return tokenRatioPriority(token0, chainId) > tokenRatioPriority(token1, chainId);
+   return tokenRatioPriority(token0, chainId) < tokenRatioPriority(token1, chainId);
}
```

Or

Reverse the priority values in `TokenRatioSortOrder.sol`:
```diff
library TokenRatioSortOrder {
-   int256 constant NUMERATOR_MOST = 300;
-   int256 constant NUMERATOR_MORE = 200;
-   int256 constant NUMERATOR = 100;
-   int256 constant DENOMINATOR_MOST = -300;
-   int256 constant DENOMINATOR_MORE = -200;
-   int256 constant DENOMINATOR = -100;

+   int256 constant NUMERATOR_MOST = -300;
+   int256 constant NUMERATOR_MORE = -200;
+   int256 constant NUMERATOR = -100;
+   int256 constant DENOMINATOR_MOST = 300;
+   int256 constant DENOMINATOR_MORE = 200;
+   int256 constant DENOMINATOR = 100;
}
```





## [QA-03] Missing publicly available tokens in whitelist query function due to mismatched authorization logic

The issue in the [`getWhitelistedTokensFor`](https://github.com/ronin-chain/katana-operation-contracts/blob/27f9d28e00958bf3494fa405a8a5acdcd5ecdc5d/src/governance/KatanaGovernance.sol#L298-L303) function relates to how tokens are filtered and returned based on whitelist status.

Take a look at this part of the code:

```solidity
if (block.timestamp < whitelistUntil && $.allowed[account]) {
    tokens[count] = token;
    whitelistUntils[count] = whitelistUntil;
    ++count;
}
```

The problem here's that this function only returns tokens where:
1. The current time is less than the whitelist expiry (`block.timestamp < whitelistUntil`)
2. The account is explicitly allowed (`$.allowed[account]`)

Albeit, this logic contradicts the [`_isAuthorized`](https://github.com/ronin-chain/katana-operation-contracts/blob/27f9d28e00958bf3494fa405a8a5acdcd5ecdc5d/src/governance/KatanaGovernance.sol#L374-L381) function's behavior:

```solidity
function _isAuthorized(Permission storage $, address account) private view returns (bool) {
    uint256 expiry = $.whitelistUntil;
    if (expiry == UNAUTHORIZED) return false;
    if (expiry == AUTHORIZED || block.timestamp > expiry) return true;
    return $.allowed[account];
}
```

The inconsistency is that `_isAuthorized` considers a token to be authorized for everyone if:
1. The whitelist has expired (`block.timestamp > expiry`)
2. The token is publicly authorized (`expiry == AUTHORIZED`)

But `getWhitelistedTokensFor` fails to include tokens that are publicly available (when `whitelistUntil == AUTHORIZED`) or tokens that have expired whitelists (which should be available to everyone). This means the function will not return tokens that the user actually has permission to interact with.


### Recommendation
The condition in `getWhitelistedTokensFor` should mirror the logic in `_isAuthorized`:

```diff
  function getWhitelistedTokensFor(address account)
    external
    view
    returns (address[] memory tokens, uint40[] memory whitelistUntils)
  {
    unchecked {
      uint256 length = _tokens.length();
      tokens = new address[](length);
      whitelistUntils = new uint40[](length);
      uint256 count;
      address token;
      uint40 whitelistUntil;
      Permission storage $;

      for (uint256 i; i < length; ++i) {
        token = _tokens.at(i);
        $ = _permission[token];
        whitelistUntil = $.whitelistUntil;

-       if (block.timestamp < whitelistUntil && $.allowed[account]) {
+       if (whitelistUntil == AUTHORIZED || block.timestamp > whitelistUntil || 
+           (block.timestamp < whitelistUntil && $.allowed[account])) {
          tokens[count] = token;
          whitelistUntils[count] = whitelistUntil;
          ++count;
        }
      }

      assembly {
        mstore(tokens, count)
        mstore(whitelistUntils, count)
      }
    }
  }
```






## [QA-04] Global authorization override bypasses token-specific permissions


Firstly, take a look at the [`isAuthorized` function](https://github.com/ronin-chain/katana-operation-contracts/blob/27f9d28e00958bf3494fa405a8a5acdcd5ecdc5d/src/governance/KatanaGovernance.sol#L249-L253):
   This function is crucial as it determines whether an account is authorized to interact with a specific token. It's called in functions like `createPair`.

   ```solidity
   function isAuthorized(address token, address account) public view returns (bool authorized) {
       if (_isSkipped(account)) return true;
       authorized = _isAuthorized(_permission[token], account);
   }
   ```

And then [the `_isSkipped` function](https://github.com/ronin-chain/katana-operation-contracts/blob/27f9d28e00958bf3494fa405a8a5acdcd5ecdc5d/src/governance/KatanaGovernance.sol#L386-L388):
This checks if an account should bypass the normal authorization checks.

   ```solidity
   function _isSkipped(address account) internal view returns (bool) {
       return isAllowedActor[account] || allowedAll() || account == owner();
   }
   ```

Now, the problem here is from the combination of these functions. If `_isSkipped(account)` returns true, `isAuthorized` immediately returns true without checking the token-specific permissions.

   - If `allowedAll()` is set to true (via `setAllowedAll(true)`), every account will be authorized for every token.
   - This overrides any specific token permissions set via `setPermission`.
   - It creates a global override that negates the granular control the contract seems designed to provide.

Consider a scenario:
   
   a. Initially, the owner sets specific permissions:
      ```solidity
      setPermission(tokenA, someTimestamp, [address1, address2], [true, true]);
      setPermission(tokenB, 0, [], []); // Unauthorized for everyone
      ```
   
   b. Now, `isAuthorized(tokenA, address1)` returns true, and `isAuthorized(tokenB, address1)` returns false, as expected.
   
   c. Later, someone calls `setAllowedAll(true)`.
   
   d. Now, `isAuthorized(tokenA, anyAddress)` and `isAuthorized(tokenB, anyAddress)` both return true for any address.


This undermines the purpose of having token-specific permissions. It creates a risk where tokens intended to be restricted become accessible to everyone.

### Recommendations
We could remove the `allowedAll()` check from `_isSkipped`, or consider restructuring the authorization logic to still check token-specific permissions even for "skipped" accounts, except for the contract owner and explicitly allowed actors.






## [QA-05] Integer division precision loss can lead to 0 token migrations in `V3Migrator`

`V3Migrator.sol`doesn't check if `amount0V2ToMigrate` and `amount1V2ToMigrate` are greater than zero before attempting to migrate. This could lead to a problematic situation in the following scenario:

If `params.percentageToMigrate` is very small (but still > 0), due to integer division by 100, either `amount0V2ToMigrate` or `amount1V2ToMigrate` could end up being 0.

https://github.com/ronin-chain/katana-v3-contracts/blob/03c80179e04f40d96f06c451ea494bb18f2a58fc/src/periphery/V3Migrator.sol#L44-L47

```solidity
uint256 amount0V2ToMigrate = amount0V2.mul(params.percentageToMigrate) / 100;
uint256 amount1V2ToMigrate = amount1V2.mul(params.percentageToMigrate) / 100;
```

For example, if `amount0V2` is 99 and `percentageToMigrate` is 1, then `amount0V2ToMigrate` would be `99 * 1 / 100 = 0`.
This could lead to a case where the contract attempts to create a V3 position with zero amount for one of the tokens.


### Recommendations
Consider adding checks to ensure both migration amounts are greater than zero:

```diff
uint256 amount0V2ToMigrate = amount0V2.mul(params.percentageToMigrate) / 100;
uint256 amount1V2ToMigrate = amount1V2.mul(params.percentageToMigrate) / 100;

+ require(amount0V2ToMigrate > 0 && amount1V2ToMigrate > 0, "Migration amounts too small");
```








## [QA-06] Hardcoded init code hash in `pairFor` creates inflexibility

The [`pairFor`](pairFor) function uses a hardcoded init code hash for calculating pair addresses:

```solidity
function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
    (address token0, address token1) = sortTokens(tokenA, tokenB);
    pair = address(
        uint256(
            keccak256(
                abi.encodePacked(
                    hex"ff",
                    factory,
                    keccak256(abi.encodePacked(token0, token1)),
                    hex"e85772d2fe4ad93037659afaee57751696456eb5dd99987e43f3cf11c6e255a2" // init code hash
                )
            )
        )
    );
}
```

If the pair contract implementation changes, the init code hash will be invalid.  Also different deployments or networks might use different pair implementations.

### Recommendation


Consider making the init code hash configurable.  Add a function in the factory contract to retrieve the init code hash dynamically and modify the library to fetch it from there.