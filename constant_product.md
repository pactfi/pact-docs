# Description of global state

| Constant name      | API               | Description                                                                     |
| ------------------ | ----------------- | ------------------------------------------------------------------------------- |
| TOTAL_LIQUIDITY    | `"L"`             | Total liquidity already distributed                                             |
| TOTAL_PRIMARY      | `"A"`             | Amount of the primary asset or ALGO                                             |
| TOTAL_SECONDARY    | `"B"`             | Amount of the secondary asset                                                   |
| LIQUIDITY_TOKEN_ID | `"LTID"`          | Asset ID of liquidity token                                                     |
| CONFIG             | `"CONFIG"`        | Concatenated asset ID of primary asset, secondary asset and fee bps of the pool |
| CONTRACT_NAME      | `"CONTRACT_NAME"` | Contract's name, here "PACT AMM"                                                |
| VERSION            | `"VERSION"`       | Contract's version                                                              |
| ADMIN              | `"ADMIN"`         | Contract's admin                                                                |
| TREASURY           | `"TREASURY"`      | Contract's treasury                                                             |
| FEE_BPS            | `"FEE_BPS"`       | Contract's fee bps                                                              |
| PACT_FEE_BPS       | `"PACT_FEE_BPS"`  | Contract's pact fee bps included in fee bps                                     |


# List of possible transaction groups

## Description of main contract arguments

Arguments which are passed as Arg[0]:

| Constant name          | API        | Description                                                 |
| ---------------------- | ---------- | ----------------------------------------------------------- |
| CREATE_LIQUIDITY_TOKEN | `"CLT"`    | Create liquidity token via an inner transaction             |
| OPT_IN_TO_ASSETS       | `"OPTIN"`  | Opt-in to secondary asset and possibly to primary asset too |
| ADD_LIQUIDITY          | `"ADDLIQ"` | Add liquidity to pool                                       |
| REMOVE_LIQUIDITY       | `"REMLIQ"` | Remove liquidity from pool                                  |
| SWAP                   | `"SWAP"`   | Swap                                                        |

Primary asset is always the asset with a smaller ID.

## Variables explanation

- `sp` - `SuggestedParams` without modification, for example from `AlgodClient`
- `deployer` - Address of some account that will be the contract creator
- `sender` - Address of an account interacting with the contract
- `app_id` - ID of the deployed contract
- `contract_address` - Address of the contract instance on the blockchain, computed from contract ID
- `primary_asset_id` - The first exchanged asset, must be 0 if Algos are exchanged, otherwise it's an ASA ID
- `secondary_asset_id` - The second asset's ID on the blockchain
- `liquidity_asset_id` - Extracted from app's global state, should be accessible after `CREATE_LIQUIDITY_TOKEN` is used
- `min_expected` - the minimum amount the user expects to receive from swap/add_liquidity

## Special helper functions

sp_large_fee sets the fee to a higher amount to pay for inner transaction fees in all app calls.

```python
def sp_large_fee(sp: SuggestedParams, amount: int) -> SuggestedParams:
    """
    Get a copy of suggested params but with a larger fee
    """
    sp = copy(sp)
    sp.flat_fee = True
    sp.fee = amount
    return sp
```

## Main contract create call

`approval_program_compiled_bytes` and `clear_program_compiled_bytes` are raw program bytes with initial values baked in - `"primary_asset_id"`, `"secondary_asset_id"`, `"FEE_BPS"`.

```python
ApplicationCreateTxn(
    sender=deployer,
    sp=sp,
    on_complete=OnComplete.NoOpOC,
    approval_program=approval_program_compiled_bytes,
    clear_program=clear_program_compiled_bytes,
    global_schema=StateSchema(4,1),
    local_schema=StateSchema(0,0),
    foreign_assets=[primary_asset_id, secondary_asset_id]
)
```

## Setup

We recommend doing all of the setup actions within one atomic group.
The necessary transactions to start the contract are described below:
Due to minimum balance requirements, the smart contract needs a certain amount of algos to do the following:

- opt-in to primary asset (in some cases)
- opt-in to secondary asset
- create liquidity token

Each of the above consumes 100000 microAlgo, plus there is a base minimum balance requirement of 100000 microAlgo.

There is no need to opt-in to primary asset if the primary asset is _Algos_.

Contract address is only known after contract has been created, therefore it's not possible to do the entire setup process in just one group.

```python
initial_deposit = 400000 if primary_asset_id else 300000

txns = [
    PaymentTxn(
        sender=deployer,
        sp=sp,
        receiver=contract_address,
        amt=initial_deposit,
    ),
    # Action: "OPTIN"
    ApplicationNoOpTxn(
        sender=deployer,
        sp=sp_large_fee(sp, amount=3000),
        index=app_id,
        app_args=[ACTION.OPT_IN_TO_ASSETS],
        foreign_assets=[primary_asset_id, secondary_asset_id]
    ),
    # Action: "CLT"
    ApplicationNoOpTxn(
        sender=deployer,
        sp=sp_large_fee(sp, amount=2000),
        index=app_id,
        app_args=[ACTION.CREATE_LIQUIDITY_TOKEN],
        foreign_assets=[primary_asset_id, secondary_asset_id]
    ),
]
```

"CLT" and "OPTIN" can be done in any order.
It musn't be possible to execute "CLT" more than once.

## Add liquidity

1. Algos to ASA exchange

```python
# amount_primary is the amount of liquidity of the primary asset user wishes to add
# amount_secondary is the amount of secondary token

txns = [
    PaymentTxn(
        sender=sender,
        sp=sp,
        receiver=contract_address,
        amt=amount_primary,
    ),
    AssetTransferTxn(
        sender=sender,
        sp=sp,
        receiver=contract_address,
        amt=amount_secondary,
        index=secondary_asset_id
    ),
    ApplicationNoOpTxn(
        sender=sender,
        sp=sp_large_fee(sp, 3000),
        index=app_id,
        app_args=[ACTION.ADD_LIQUIDITY, min_expected],
        foreign_assets=[primary_asset_id, secondary_asset_id, liquidity_asset_id]
    )
]
```

2. ASA to ASA exchange
   The only difference lies in the second transaction in the group

```python
txns = [
    AssetTransferTxn(
        sender=sender,
        sp=sp,
        receiver=contract_address,
        amt=amount_primary,
        index=primary_asset_id
    ),
    AssetTransferTxn(
        sender=sender,
        sp=sp,
        receiver=contract_address,
        amt=amount_secondary,
        index=secondary_asset_id
    ),
    ApplicationNoOpTxn(
        sender=sender,
        sp=sp_large_fee(sp, 3000),
        index=app_id,
        app_args=[ACTION.ADD_LIQUIDITY, min_expected],
        foreign_assets=[primary_asset_id, secondary_asset_id, liquidity_asset_id]
    )
]
```

## Remove liquidity

```python
# amount_liquidity is the amount of pool shares, that the liquidity provider wants to convert back to assets

txns = [
    AssetTransferTxn(
        sender=sender,
        sp=sp,
        receiver=contract_address,
        amt=amount_liquidity,
        index=liquidity_asset_id
    ),
    ApplicationNoOpTxn(
        sender=sender,
        sp=sp_large_fee(sp, 3000),
        index=app_id,
        app_args=[ACTION.REMOVE_LIQUIDITY, min_expected_primary, min_expected_secondary],
        foreign_assets=[primary_asset_id, secondary_asset_id]
    )
]
```

## Swap

There are three variations of the swap action:

1. swap primary asset & the primary asset is Algos
2. swap primary asset & the primary asset is ASA
3. swap secondary asset (it must be an ASA)

`amount` is the amount intended to be swapped

Case #1 (swap Algos to something else):

```python
txns = [
    PaymentTxn(
        sender=sender,
        sp=sp,
        receiver=contract_address,
        amt=amount,
    ),
    ApplicationNoOpTxn(
        sender=sender,
        sp=sp_large_fee(sp, 2000),
        index=app_id,
        app_args=[ACTION.SWAP, min_expected],
        foreign_assets=[primary_asset_id, secondary_asset_id]
    )
]
```

Case #2 and #3 (send assets to contract and swap them):

`asset_to_swap` must be equal to either `primary_asset_id` or `secondary_asset_id`.
The smart contract will figure out which asset is being deposited, make appropriate calculations and send back the other asset via an inner txn.

```python
txns = [
    AssetTransferTxn(
        sender=sender,
        sp=sp,
        receiver=contract_address,
        amt=amount,
        index=asset_to_swap
    ),
    ApplicationNoOpTxn(
        sender=sender,
        sp=sp_large_fee(sp, 2000),
        index=app_id,
        app_args=[ACTION.SWAP, min_expected],
        foreign_assets=[primary_asset_id, secondary_asset_id]
    )
]
```
