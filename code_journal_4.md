# CW20

CW20 is a specification for fungible tokens based on CosmWasm. The name and design is loosely based on Ethereum's ERC20 standard, but many changes have been made. The types in here can be imported by contracts that wish to implement this spec, or by contracts that call to any standard cw20 contract.

The specification is split into multiple sections, a contract may only implement some of this functionality, but must implement the base.

## Base

This handles **balances** and **transfers**. Note that all amounts are handled as `Uint128` (128 bit integers with JSON string representation). Handling decimals is left to the UI and not interpreted

### Queries

1. `Balance{address}` - Returns the balance of the given address. Returns "0" if the address is unknown to the contract. Return type is `BalanceResponse{balance}`.

2. `TokenInfo{}` - Returns the token info of the contract. Return type is `TokenInfoResponse{name, symbol, decimal, total_supply}`.

### Receiver

The counter-part to `Send` is `Receive`, which must be implemented by any contract that wishes to manage CW20 tokens. This is generally _not_ implemented by any CW20 contract.

1. `Receive{sender, amount, msg}` - This is designed to handle `Send` messages. The address of the contract is stored in `info.sender` so it cannot be faked. The contract should ensure the sender matches the token contract it expects to handle, and not allow arbitrary addresses.

The `sender` is the original account requesting to move the tokens and `msg` is a `Binary` data that can be decoded into a contract-specific message. This can be empty if we have only one default action, or it may be a `ReceiveMsg` variant to clarify the intention. For example, if I send to a uniswap contract, I can specify which token I want to swap against using this field.

## Allowances

A contract may allow actors to delegate some of their balance to other accounts. This is not as essential as with ERC20 as we use `Send`/`Receive` to send tokens to a contract, not `Approve`/`TransferFrom`. But it is still a nice use-case, and you can see how the Cosmos SDK wants to add payment allowances to native tokens. This is mainly designed to provide access to other public-key-based accounts.

There was an issue with race conditions in the original ERC20 approval spec. If you had an approval of 50 and I then want to reduce it to 20, I submit a Tx to set the allowance to 20. If you see that and immediately submit a tx using the entire 50, you then get access to the other 20. Not only did you quickly spend the 50 before I could reduce it, you get another 20 for free.

The solution discussed in the Ethereum community was an `IncreaseAllowance` and `DecreaseAllowance` operator (instead of `Approve`). To originally set an approval, use `IncreaseAllowance`, which works fine with no previous allowance. `DecreaseAllowance` is meant to be robust, that is if you decrease by more than the current allowance (eg. the user spent some in the middle), it will just round down to 0 and not make any underflow error.

### Messages

1. `IncreaseAllowance{spender, amount, expires}` - Sets or increases the allowance such that `spender` may access up to `amount + current_allowance` tokens from the `info.sender` account. This may optionally come with an `Expiration` time, which if set limits when the approval can be used (by time or height).

2. `DecreaseAllowance{spender, amount, expires}` - Decreases or clears the allowance such that `spender` may access up to `current_allowance - amount` tokens from the `info.sender` account. This may optionally come with an `Expiration` time, which if set limits when the approval can be used (by time or height). If `amount >= current_allowance`, this will clear the allowance (delete it).

3. `TransferFrom{owner, recipient, amount}` - This makes use of an allowance and if there was a valid, un-expired pre-approval for the `info.sender`, then we move `amount` tokens from `owner` to `recipient` and deduct it from the available allowance.

4. `SendFrom{owner, contract, amount, msg}` - `SendFrom` is to `Send`, what `TransferFrom` is to `Transfer`. This allows a pre-approved account to not just transfer the tokens, but to send them to another contract to trigger a given action. **Note** `SendFrom` will set the `Receive{sender}` to be the `info.sender` (the account that triggered the transfer) rather than the `owner` account (the account the money is coming from). This is an open question whether we should switch this?

5. `BurnFrom{owner, amount}` - This works like `TransferFrom`, but burns the tokens instead of transfering them. This will reduce the owner's balance, `total_supply` and the caller's allowance.

### Queries

1. `Allowance{owner, spender}` - This returns the available allowance that `spender` can access from the `owner`'s account, along with the expiration info. Return type is `AllowanceResponse{balance, expiration}`.

## Mintable

This allows another contract to mint new tokens, possibly with a cap. There is only one minter specified here, if you want more complex access management, please use a multisig or other contract as the minter address and handle updating the ACL there.

### Messages

1. `Mint{recipient, amount}` - If the `info.sender` is the allowed minter, this will create `amount` new tokens (updating total supply) and add them to the balance of `recipient`, as long as it does not exceed the cap.

2. `UpdateMinter { new_minter: Option<String> }` - Callable only by the current minter. If `new_minter` is `Some(address)` the minter is set to the specified address, otherwise the minter is removed and no future minters may be set.

### Queries

1. `Minter{}` - Returns who and how much can be minted. Return type is `MinterResponse {minter, cap}`. Cap may be unset.

If the cap is set, it defines the maximum `total_supply` that may ever exist. If initial supply is 1000 and cap is `Some(2000)`, you can only mint 1000 more tokens. However, if someone then burns 500 tokens, the minter can mint those 500 again. This allows for dynamic token supply within a set of parameters, especially when the minter is a smart contract.

## Code

### Instantiate

    #[cfg_attr(not(feature = "library"), entry_point)]
    pub fn instantiate(
        mut deps: DepsMut,
        _env: Env,
        _info: MessageInfo,
        msg: InstantiateMsg,
    ) -> Result<Response, ContractError> { ... }

To create an instance of a cw20 token, we need to pass to the `instantiate` entry point the following fields:

    pub struct InstantiateMsg {
        pub name: String,
        pub symbol: String,
        pub decimals: u8,
        pub initial_balances: Vec<Cw20Coin>,
        pub mint: Option<MinterResponse>,
        pub marketing: Option<InstantiateMarketingInfo>,
    }

`initial_balances` contains a list of the addresses and their respective balances that will be funded with cw20 tokens at instantiation.

    pub struct Cw20Coin {
        pub address: String,
        pub amount: Uint128,
    }

`mint` is an optional field that is used to set the `total_supply`. If `None`, there's an unlimited cap.

    pub struct MinterResponse {
        pub minter: String,
        pub cap: Option<Uint128>,
    }

`marketing` is also an optional field to define information about your project.

    pub struct InstantiateMarketingInfo {
        pub project: Option<String>,
        pub description: Option<String>,
        pub marketing: Option<String>,
        pub logo: Option<Logo>,
    }

If the details provided in `InstantiateMsg` are formatted correctly, then it creates the accounts with their respective `BALANCES`, and saves them to storage returning the `total_supply`

    msg.validate()?;
    let total_supply = create_accounts(&mut deps, &msg.initial_balances)?;

When creating accounts, it first validates that no addresses are duplicated

    pub fn validate_accounts(accounts: &[Cw20Coin]) -> Result<(), ContractError> {
        let mut addresses = accounts.iter().map(|c| &c.address).collect::<Vec<_>>();
        addresses.sort();
        addresses.dedup();

Then sets the `total_supply` to `Uint128::zero()`, validates each address of the `initial_balances` list and iterates over each account, adding each individual balance to the `total_supply`.

    pub fn create_accounts(
        deps: &mut DepsMut,
        accounts: &[Cw20Coin],
    ) -> Result<Uint128, ContractError> {
        validate_accounts(accounts)?;

        let mut total_supply = Uint128::zero();
        for row in accounts {
            let address = deps.api.addr_validate(&row.address)?;
            BALANCES.save(deps.storage, &address, &row.amount)?;
            total_supply += row.amount;
        }

        Ok(total_supply)
    }

If `total_supply` was manually set in `MinterResponse`, it checks that the tokens minted are lower than the limit or returns an error.

    if let Some(limit) = msg.get_cap() {
        if total_supply > limit {
            return Err(StdError::generic_err("Initial supply greater than cap").into());
        }
    }

Then `mint` details are saved and the address that can mint tokens is validated

    let mint = match msg.mint {
        Some(m) => Some(MinterData {
            minter: deps.api.addr_validate(&m.minter)?,
            cap: m.cap,
        }),
        None => None,
    };

`TokenInfo` is stored in the blockchain

    let data = TokenInfo {
        name: msg.name,
        symbol: msg.symbol,
        decimals: msg.decimals,
        total_supply,
        mint,
    };
    TOKEN_INFO.save(deps.storage, &data)?;

It does some checks for `MarketingInfo` and stores them in the blockchain as well.

### Execute

    #[cfg_attr(not(feature = "library"), entry_point)]
    pub fn execute(
        deps: DepsMut,
        env: Env,
        info: MessageInfo,
        msg: ExecuteMsg,
    ) -> Result<Response, ContractError> {
        match msg { ... }

1. `Transfer{recipient, amount}` - Moves amount tokens from the `info.sender` account to the `recipient` account. This is designed to send to an address controlled by a private key and **does not trigger** any actions on the recipient if it is a contract.

```
    ExecuteMsg::Transfer { recipient, amount } => {
        execute_transfer(deps, env, info, recipient, amount)
    }
```

2. `Burn{amount}` - Removes `amount` tokens from the balance of `info.sender` and reduces `total_supply` by the same amount.

```
    ExecuteMsg::Burn { amount } => execute_burn(deps, env, info, amount),
```

3. `Send{contract, amount, msg}` - Moves `amount` tokens from the `info.sender` account to the `contract` account. `contract` must be an address of a contract that implements the `Receiver` interface. The `msg` will be passed to the recipient contract, along with the amount.

   ````
    ExecuteMsg::Send {
        contract,
        amount,
        msg,
    } => execute_send(deps, env, info, contract, amount, msg),```

   ````

4. `Mint{recipient, amount}` - If the `info.sender` is the allowed minter, this will create `amount` new tokens (updating total supply) and add them to the balance of `recipient`, as long as it does not exceed the cap.

```
    ExecuteMsg::Mint { recipient, amount } => execute_mint(deps, env, info, recipient, amount),

```

5. `IncreaseAllowance{spender, amount, expires}` - Sets or increases the allowance such that `spender` may access up to `amount + current_allowance` tokens from the `info.sender` account. This may optionally come with an `Expiration` time, which if set limits when the approval can be used (by time or height).

```
    ExecuteMsg::IncreaseAllowance {
        spender,
        amount,
        expires,
    } => execute_increase_allowance(deps, env, info, spender, amount, expires),
```

6. `DecreaseAllowance{spender, amount, expires}` - Decreases or clears the allowance such that `spender` may access up to `current_allowance - amount` tokens from the `info.sender` account. This may optionally come with an `Expiration` time, which if set limits when the approval can be used (by time or height). If `amount >= current_allowance`, this will clear the allowance (delete it).

```
    ExecuteMsg::DecreaseAllowance {
        spender,
        amount,
        expires,
    } => execute_decrease_allowance(deps, env, info, spender, amount, expires),
```

7. `TransferFrom{owner, recipient, amount}` - This makes use of an allowance and if there was a valid, un-expired pre-approval for the `info.sender`, then we move `amount` tokens from `owner` to `recipient` and deduct it from the available allowance.

```
    ExecuteMsg::TransferFrom {
        owner,
        recipient,
        amount,
    } => execute_transfer_from(deps, env, info, owner, recipient, amount),
```

8. `BurnFrom{owner, amount}` - This works like `TransferFrom`, but burns the tokens instead of transfering them. This will reduce the owner's balance, `total_supply` and the caller's allowance.

```
    ExecuteMsg::BurnFrom { owner, amount } => execute_burn_from(deps, env, info, owner, amount),
```

9. `SendFrom{owner, contract, amount, msg}` - `SendFrom` is to `Send`, what `TransferFrom` is to `Transfer`. This allows a pre-approved account to not just transfer the tokens, but to send them to another contract to trigger a given action. **Note** `SendFrom` will set the `Receive{sender}` to be the `info.sender` (the account that triggered the transfer) rather than the `owner` account (the account the money is coming from). This is an open question whether we should switch this?

```
    ExecuteMsg::SendFrom {
        owner,
        contract,
        amount,
        msg,
    } => execute_send_from(deps, env, info, owner, contract, amount, msg),
```
