# API v1

All endpoints' addresses can be constructed by prefixing their path with the desired environment's API URL.
For production (mainnet) that would be: `https://api.pact.fi/api`.

## `/v1/prices`

Report most recent pricing information for various assets. The criteria to list an asset are:

1. Asset must be in a pool with over $1000 TVL
2. Asset must have pricing information in the database

Pricing information is only available if either:

- The asset in question has a trading pair with a stablecoin (USDC, USDt)
- The asset in question has a trading pair with an asset that has a trading pair with a stablecoin (USDC, USDt)
- The asset is a liquidity asset in a pool where both assets have a known price

### Request

- GET (no parameters)

### Response

- GET
  - JSON file with asset ID keys and decimal values. Algo is represented by a special ID = 0. The aforementioned stablecoins (USDC, USDt) have price of exactly 1. The quotes are denominated in USD.

#### Example

```
{"0":"0.37128472","312769":"1.00000000"}
```
