# Router

Route transactions between pools constant product and stable swap ones.
The purpose of the router is to make multiple swaps atomically within a single transaction group.
This includes swaps through multiple pairs (e.g. swap from ALGO to USDC, then from USDC to USDt).
Using a router allows the user to swap the entire amount and not worry about receiving a partial amount and change, or having the transaction canceled.

## User roles

- Swapper: Swaps assets through multiple pools
- Admin: Can perform privileged operations
- Updater: Can update the contract code
- Deleter: Can delete the contract

## Admin operations

The administrator can perform two very important operations:

- `SUPEROPTIN`: Persistently opt-in the router contract to an asset. Thanks to this, the swappers won't need to opt-in to the asset every time they want to use it.
- `SUPEROPTOUT`: Close-out all assets to a beneficiary address. This is useful when the router contract is no longer needed. The assets are closed-out to the asset creator's address.

## Swapper operations

### OPTIN

Opt-in to assets that are used in swaps (must not have been super-opted-in). Can only come before a swap.
`OPTIN` must not be called if the router contract has already been super-opted-in to all of the assets.

Args:

- `assets` (`uint64[]`): The list of asset IDs to opt-in to

Assets:
The same assets as the `assets` argument

### SWAP

Perform a swap through the router contract.
Immediately preceding the first swap app call a deposit must be made to the router contract.

The necessary arguments for the swap are described in `router_interface.json`

### SWAP (2 hops)

This is a special swap method that performs 2 swaps in a row instead of just one.
It is functionally equivalent to calling `SWAP` twice in a row, but it is more efficient (saves 1 app call).

### OPTOUT

Opt-out of assets that are used in swaps. Can only come after a swap.
If the group doesn't have an opt-in it must not have an opt-out.
If the group has an opt-in it must have an opt-out with the same assets.

Args:

- `assets` (`uint64[]`): The list of asset IDs to opt-in to

Assets:
The same assets as the `assets` argument

## Atomic transaction group structure

To make a swap using the Router the following preconditions must be met:

1. The router is opted-in to all assets that are used for swaps.
2. The pools used in the route are pact.fi pools.

The transactions in the group are as follows:

1. (optional) opt-in to assets
2. deposit of the source asset (the first one to be swapped) to the router contract
3. call to the router contract

- This step can be repeated multiple times, depending on the amount of pools that the swap goes through
- There is an optimization of this step, which goes through 2 pools within a single app call, it's described elsewhere
- The route must be continuous (i.e. the first pool output must be the same as second pool input and so on)

4. (optional) opt-out of assets used in step 1

### Routing example

Consider a swap from $PLANET to $ALGO.
An off chain algorithm has determined that the best course of action is to perform multiple swaps instead of trading $PLANET/$ALGO pair directly (for example due to the fact that other pairs are more liquid).

Such a trade could happen like this:

1. Swap $PLANET to $USDt
2. Swap received $USDt to $USDC
3. Swap received $USDC to $ALGO

In such case the router guarantees using of all intermediary assets received on the route and the atomicity of the transaction.

The composition of the atomic group transaction from the example above would look like this:

1. opt-in to assets: PLANET, USDt, USDC (ALGO is not an ASA)
2. deposit of PLANET to the router contract
3. swap (2 pools): PLANET to USDt, USDt to USDC
4. swap (1 pool): USDC to ALGO
5. opt-out of assets: PLANET, USDt, USDC

Equally valid, albeit more expensive would be:

1. [opt-in same as above]
2. [deposit same as above]
3. swap (1 pool): PLANET to USDt
4. swap (1 pool): USDt to USDC
5. swap (1 pool): USDC to ALGO
6. [opt-out same as above]

#### Routing example with ABI

1. opt-in to assets: `OPTIN([PLANET, USDt, USDC])`
2. deposit of PLANET to the router contract
3. swap (2 pools): PLANET to USDt, USDt to USDC:
   `SWAP(PLANET, USDt, POOL_1_ID, POOL_1_ADDR, "PACT_SWAP_ASA_ASA", USDt, USDC, POOL_2_ID, POOL_2_ADDR, "PACT_SWAP_ASA_ASA", 0)`
4. swap (1 pool): USDC to ALGO (ALGO have special ID = 0):
   `SWAP(USDC, 0, POOL_3_ID, POOL_3_ADDR, "PACT_SWAP_ASA_ALGOS", MIN_EXPECTED)`
5. opt-out of assets:
   `OPTOUT([PLANET, USDt, USDC], [PLANET_CREATOR_ADDR, USDt_CREATOR_ADDR, USDC_CREATOR_ADDR])`

## Method signatures

The available method signatures from the ABI are described in [.router_interface.json](./router_interface.json).

### Checking minimum expected amount

One of the parameters for the router swap is the minimum expected value. This works very similarly to the minimum expected in pool swap.
It only makes sense to specify this parameter in the last swap in the sequence. Otherwise this parameter is ignored.

Example txn group that uses the min_expected parameter:

1. deposit of PLANET to the router contract
2. swap (1 pool): PLANET to USDt
3. swap (1 pool): USDt to USDC
4. swap (1 pool): USDC to ALGO (min_expected = N)
5. opt-out of assets: PLANET, USDt, USDC

Should the received amount of microALGO be less than N after those three swaps the transaction will fail. If transactions 2 or 3 include the min_expected parameter it will be ignored.
