katana-v3-contracts-03c80179e04f40d96f06c451ea494bb18f2a58fc\src\core\KatanaV3Pool.sol  [659~666]

Code examples:
if (!cache.computedLatestObservation) {
     (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
              cache.blockTimestamp,
                0,
              slot0Start.tick,
              slot0Start.observationIndex,
              cache.liquidityStart,
              slot0Start.observationCardinality
      );
the code executes an external call to observations.observeSingle() within the if condition, and this call is within a loop. if cache.computedLatestObservation is false, then this external function is called every time through the loop.