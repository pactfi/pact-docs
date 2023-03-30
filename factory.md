# Factory
Build pools with factory contract. The purpose of this contract is to enable listing pools in a fully decentralized way. Factory ensures the uniqueness of pools with given configuration therefore less possibilities for liquidity defragmentation.

# Constant Product Factory

## Description of global state

| Constant name         | Description                                                                                |
| --------------------- | ------------------------------------------------------------------------------------------ |
| ADMIN                 | Admin address to be set in built pools                                                     |
| TREASURY              | Treasury address to be set in built pools                                                  |
| POOL_CONTRACT_VERSION | Version of the pool contract built by factory                                              |
| ALLOWED_FEE_BPS       | Fee bps allowed to be set in built pools                                                   |
| ALLOWED_PACT_FEE_BPS  | 1:1 mapping with ALLOWED_FEE_BPS of pact fee bps to be set in pool when fee bps is chosen  |

## Pools' uniqeuness
Pool's uniqeuness is determined by checking the existence of a box. The box's name is created by concatenating: primary asset id, secondary asset id, fee bps and version of pool to be built. 
The box contains the app id of a pool with a given configuration.

## List of possible transaction groups
ARC-4 specification is available under the "contract" key in the [../factory_interface.json](./factory_interface.json).

### Build
Build [constant product pool](./constant_product.md) for specified assets and fee bps. It sends 4 app call transactions - creating the pool, sends ALGO to satisfy pool's minimum balance, opts in to the assets and creates liquidity token.
Pool's uniqueness is validated here.
The call will fail, if:
- pool with given config already exists
- fee bps is not in `ALLOWED_FEE_BPS`
- primary asset id is greater or equal than secondary asset id 

Note: build only deploys the pool and configures it. To start swapping one should provide liquidity as sepcified in [add liquidity spec](./constant_product.md#add-liquidity).

### Get box key
Returns box key for provided assets and fee bps.

### Lookup
Lookup if the pool already exists. It checks if the box with the key, provided in app args, exists. If yes, it returns app id of pool, 0 otherwise.

### Admin operations
There are 3 admin actions which can be performed only by `ADMIN` address:
- update factory contract - it consists of 2 app calls - `update` and `post update`, where the latter sets new `POOL_CONTRACT_VERSION`, `ALLOWED_FEE_BPS`, `ALLOWED_PACT_FEE_BPS`
- change `ADMIN`
- change `TREASURY`