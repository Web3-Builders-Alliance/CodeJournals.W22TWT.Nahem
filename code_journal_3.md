# White-Whale-Defi-Platform/migaloo-core

## TerraSwap Factory

Migaloo's factory contract is used to create pair (pool) contracts. Pools are comprised of two tokens, which can be either native, ibc or cw20 tokens. Once a pool is created it's stored in state, meaning the factory acts as a pool registry, which can be queried later for reference. Note that the pool factory is permissioned, meaning the messages can only be executed by the owner of the contract.

To create a project using cargo from an specific Github repo, run the following command:

    cargo generate --git https://github.com/White-Whale-Defi-Platform/migaloo-core.git --name PROJECT_NAME

This time we're using the migaloo-core from the White-Whale-Defi-Platform repo, specifically the terraswap factory pool commands.

## Mechanism

The smart contract is basically a counter app that keeps its state on-chain.

It can be either incremented by `1` by anyone or reset to an arbitrary number by the contract owner only.

Anyone can query the current value of the counter.

The tests check that all functions (instantiate, increment, reset and query_count) work as they should but also check that the contract throws an error when anyone different from the owner tries to reset the count.

## Organisation

Cargo creates a new folder named PROJECT_NAME with some files and folders inside. Here's a short description of the most relevant files/folders that cargo creates for us.

1. The .cargo file is a hidden folder that contains the configuration file for cargo. It's mainly used to create aliases and make cargo commands shorter and easier to visualise or remember.

2. The circleci and github/workflow folders contain configuration for continious integration. We're not digging into it yet as this is useful for larger projects and teams to automate checks when uploading code to Git repos.

3. The src folder is where the magic happens, it contains the pieces that, together, define the smart contracts of our project.

   1. There's a bin folder with a schema.rs file. This file is used to create the schema/PROJECT_NAME.json file when running the `cargo schema` command.

   2. The contract.rs file is the main file that contains the logic of our program. It has all the entry points of our contract that are exposed to the blockchain to interact with.

   3. Inside the error.rs file is where we define the error types our contract is going to use. The more specific the error, the better UX.

   4. If we want to keep our code clean, we might want to use helper functions to not repeat ourselves too much. We can create functions here that will be used along the contract here and import them later on.

   5. The integration_tests.rs file is used to test that our smart contract behaves as it should (or as we think it should). We must write tests to check if our program is actually doing what we think we want, and fix bugs until it does. It's impossible to test every scenario, but it's always a good practice to test core functionallity and interaction between parts of our contract.

   6. The lib.rs file works like an index. It tells cargo how ours files and folders are structured so we can import them into our contract files. We'll have to manually add to it as we create new files and folders to our smart contract.

   7. We define the messages we're going to be using in the contract in the msg.rs file. Messages are usually structs or enums, depending on the case. Responses from queries are defined here too.

   8. The state.rs file is where we define how we're going to keep the state of our contract. Usually, we define here the `state` as a struct and then a `STATE` constant that's going to wrap the `state` either in an Item or a Map, depending on the logic of our contract. This is going to be stored on-chain and we can query it, update it or create new instances.

4. The .editorconfig file is used to format the workspace.

5. The .gitignore file tells git what files should not be uploaded to the online repos. This is very useful for keeping private data and files locally, like private keys, apikeys and anything we don't want to expose to the Internet.

6. The cargo.lock file is created by cargo and it's not intended to be manually edited.

7. Cargo.toml is where we want to include our dependencies. As we need to use external crates or packages, we should manually add them under `[dependencies]`. **Tip**: you can use VSCODE extension called Better TOML to format the files with .toml extension.

## Contract

`use` is similar to import in other programming languages. In this case we're importing the struct `state` and the const `STATE` that are in the state.rs file so we can use them in our smart contract without passing the complete path.

    use crate::state::{State, STATE};

`const` is the short for constant and is used to define values in our contract that won't change. For naming convention, we should use uppercase and separate words with `_`. It's always necessary to define the type with `:`,in this case is a `String` slice.

    const CONTRACT_NAME: &str = "crates.io:cw-template";

This is a macro to define that the following function will be an entry point for the program. `entry_point` are exposed to the blockchain and can be interacted with.

    #[cfg_attr(not(feature = "library"), entry_point)]

### Instantiate

`instantiate` is a public function that's accesible from other files outside this contract. It's similar to a constructor function in other programming languages. Basically, it's used to initialize the contract with the default values after deployment in the blockchain. The values inside `()` are the variables we pass to the function so they can be used inside its scope. We always need to specify its type and we're going to talk about them below. The `->` tells what type of output the function should return, in this case, it's either a `Response` when everything goes as planned or a `ContractError` if something went wrong, wrapped in a Result type to contain the error and not cause a kernel panic. Then, the actual commands of the function are found inside the `{}` which are explained below line by line.

    pub fn instantiate(
        deps: DepsMut,
        \_env: Env,
        info: MessageInfo,
        msg: InstantiateMsg,
    ) -> Result<Response, ContractError> { ... }

`deps` is short for dependencies and there're of type `DepsMut`, which means there're mutable (their value can be changed or mutated). We usually pass dependecies to access storage.

    deps: DepsMut,

`env` is short for environment and this is where the blockchain state is usually maintained. Things like block `time`, `height`, `chain_id` etc. are stored inside `env`. In this case there's a `_` before the variable to tell the compiler we're not using this variable inside the function, so there's no warning at compile time. There's no use in passing variables we're not going to use inside the function, but this time we do it to keep things standardised.

    \_env: Env,

`info` is short for information and is of type `MessageInfo`. It's a struct that contains the `sender` which signed the transaction and the `funds` that are sent to the contract.

    info: MessageInfo,

`msg` is short for message and is of type `InstantiateMsg`. This is a custom type we previously defined in the msg.rs file and will contain the fields needed to intantiate the contract. In our case we need to pass the value at which we want to initialise the counter.

    msg: InstantiateMsg,

`let` is the keyword we use to define a variable in Rust. Variables are immutable by default until we say otherwise. In this case we're defining a variable `state` which is of type `State` and has two fields: `count` and `owner`. We get the `count` from the value passed in the `msg` and the owner in this case is whoever signs the transaction, `info.sender`. Note that we `.clone` the value because we use it later when adding attributes to the `Response`. and in Rust, there can only be one owner of a variable at a time.

    let state = State {
        count: msg.count,
        owner: info.sender.clone(),
    };

The `set_contract_version( ... )` function is used when we instantiate the contract to store the original version in the blockchain so we can then migrate it if changes in the code are needed. We pass as parameters `deps.storage` to save configuration on-chain, also `CONTRACT_NAME` and `CONTRACT_VERSION` constants defined before.

    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;

`STATE.save( ... )` is used to store the state of the contract that we just defined on-chain. We pass `deps.storage` to give the contract mutable access to storage and `&state` to save the initial configuration. Note that we use `&` in this case, which means we are borrowing the variable `state` so we don't own it (cannot be modified). There's also a `?` which is used in Rust to handle errors. If something went wrong, we'll return a `ContractError` wrapped in a `Result`.

    STATE.save(deps.storage, &state)?;

If anything broke before, it means that the function didn't throw any errors so we should return an `Ok(Response::new() ... )` with some attributes to help index the transaction later on. The attributes we pass are discretionary and should be related to what the function did, in this case, we define a `method` called `instantiate` and add the `owner` and `count` we just set.

    Ok(Response::new()
        .add_attribute("method", "instantiate")
        .add_attribute("owner", info.sender)
        .add_attribute("count", msg.count.to_string()))
    }

### Execute

The `pub fn execute( ... )` function is similar to `instantiate` but is a bit more complex as it contains all possible executable messages our contract needs to handle. `match` can be used to run code conditionally (controls flow based on pattern matching). Every pattern must be handled exhaustively, either explicitly or by using wildcards like `_`. In our case, the contract has only two possible `ExecuteMsg`: `Increment` and `Reset`. For each message, we map it to its respective function using the `=>` followed by the function name and passing the corresponding parameters.

    #[cfg_attr(not(feature = "library"), entry_point)]
    pub fn execute(
        deps: DepsMut,
        _env: Env,
        info: MessageInfo,
        msg: ExecuteMsg,
    ) -> Result<Response, ContractError> {
        match msg {
            ExecuteMsg::Increment {} => execute::increment(deps),
            ExecuteMsg::Reset { count } => execute::reset(deps, info, count),
        }
    }

`mod` is used to organise code into modules. In order to use the `execute` module from other crates we need add the `pub` keyword.

    pub mod execute { ... }

`use super::*` is a wildcard import and is implemented to bring in everything from the parent module.

    use super::*;

The `increment` function takes the dependencies as mutable and returns a `Result` wrapping either a `Response` or a `ContractError`. We update the `STATE` on chain, defined as `mut`, with the previous value of `state.count` incremented by `1`. If there are no errors, we return the `state` wrapped in a `Result` and add an attribute to be able to index it later; the `?` handles the error if any.

    pub fn increment(deps: DepsMut) -> Result<Response, ContractError> {
        STATE.update(deps.storage, |mut state| -> Result<_, ContractError> {
            state.count += 1;
            Ok(state)
        })?;

        Ok(Response::new().add_attribute("action", "increment"))
    }

Only the owner of the contract is allowed to reset the counter. If the `sender` is not the `owner`, we throw an error and stop processing the remaining instructions in the function. We should always check for errors first to avoid doing further calculations and save on gas / time.

    if info.sender != state.owner {
        return Err(ContractError::Unauthorized {});
    }

### Query

The `query( ... )` function is similar to the `execute( ... )` function but there're two main differences. First, because it's used to query the state, dependencies don't need to be mutable, think of it as a read-only function. And last, instead of returning a `Response`, it returns a `StdResult` wrapping a `Binary`, which in turn is a wrapper around a vector that adds `base64` de/serialization with `serde`.

    #[cfg_attr(not(feature = "library"), entry_point)]
    pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
        match msg {
            QueryMsg::GetCount {} => to_binary(&query::count(deps)?),
        }
    }

Because we don't need to `save` or `update` the blockchain state when using a query, we use `load` instead and pass the dependencies to the function. If there was nothing to load, the `?` will handle the error safely. We previously defined a struct called `GetCountResponse` in the msg.rs crate that returns the `count` with it.

    pub fn count(deps: Deps) -> StdResult<GetCountResponse> {
        let state = STATE.load(deps.storage)?;
        Ok(GetCountResponse { count: state.count })
    }

### Tests

Writing unit tests when developing smart contracts is very useful because they help us verify that the functions we wrote do what they should.

We need to import some structures that simulate the state of the blockchain, create a fake storage and simulate the user signing the transaction and sending funds. This is all achieved using the testing module from the standard library of CosmWasm and must only be used for testing purposes.

    use cosmwasm_std::testing::{mock_dependencies, mock_env, mock_info};

First, we create a mutable variable `deps` and assign to it to the result from the `mock_dependencies()` function, which creates all external requirements that can be injected for unit tests.

        let mut deps = mock_dependencies();

Also, we create a variable named `msg` that simulates the `InstantiateMsg` with an arbitrary number for `count`.

    let msg = InstantiateMsg { count: 17 };

We simulate a dummy user `creator` who sends `1000 earth` coins to the contract address.

    let info = mock_info("creator", &coins(1000, "earth"));

Then we call the instantiate( ... ) functions passing all variables previously defined and save the result to a variable `res`, short from response. We call the `unwrap()` method to make sure the function returns an Ok value, but if there's an error we want the test to fail.

    let res = instantiate(deps.as_mut(), mock_env(), info, msg).unwrap();

`assert_eq!( ... , ... )` compares both values and panics if they're different. This is very useful to check that the expected response from the `instantiate` function matches the value we hardcoded before in the `InstantiateMsg { count: 17 }` we passed. If anything panics then the test will pass.

    assert_eq!(0, res.messages.len());

We now double check that the instantiate function is properly working. For that, we'll use the `query` function passing the `GetCount` parameter, unwrap it to make sure if doesn't fail, and then we need to convert it back `from_binary` to be able to compare the value obtained with the hardcoded `count` previously set. The dependencies are passed as reference when querying the contract because we don't need to modify blockchain state.

    let res = query(deps.as_ref(), mock_env(), QueryMsg::GetCount {}).unwrap();
    let value: GetCountResponse = from_binary(&res).unwrap();
    assert_eq!(17, value.count);

Once we check that `instantiate()` works fine, we do the same for `increment()` and `reset()`. Bear in mind that we need to `instantiate()` the contract before we can either use the `increment()` or `reset()` functions. We should also check that the contract returns an error when trying to reset the counter when the sender is different from the contract owner.

    let unauth_info = mock_info("anyone", &coins(2, "token"));
    let msg = ExecuteMsg::Reset { count: 5 };
    let res = execute(deps.as_mut(), mock_env(), unauth_info, msg);
    match res {
        Err(ContractError::Unauthorized {}) => {}
        _ => panic!("Must return unauthorized error"),
    }

## Improvements

This is a very basic contract and is not optimised or organised in the best way posible.

1. The contract.rs crate can be simplified a litle bit more, moving the `execute`, `query` and `tests` modules to independent crates and importing them at the very top of the contract.

2. We can always check for more cases for failure, for example, making sure that we don't overflow the counter when adding `1` to the maximum value `i32` allows.

3. More functionality can also be added, like decrementing the counter or even allowing incrementing it by an arbitrary number instead of `1`.

4. The error `Unauthorized` can be more specific, telling that only the owner of the contract can reset the count. Also, when doing more checks for failure, more specific errors can be added to the error.rs crate.

5. After incrementing or resetting the count, we could add the current count as an attribute to the `Response`.

6. We could also check that the attributes sent in the response of all functions are what we expect to receive.

### https://github.com/White-Whale-Defi-Platform/migaloo-core/blob/ccf27954742720600cac727fc8517ddf67c7d5f7/contracts/liquidity_hub/pool-network/terraswap_factory/src/commands.rs

use cosmwasm_std::{
to_binary, wasm_execute, CosmosMsg, DepsMut, Env, ReplyOn, Response, SubMsg, WasmMsg,
};

use terraswap::asset::AssetInfo;
use terraswap::pair::{
FeatureToggle, InstantiateMsg as PairInstantiateMsg, MigrateMsg as PairMigrateMsg, PoolFee,
};
use terraswap::querier::query_balance;

use crate::error::ContractError;
use crate::state::{
add_allow_native_token, pair_key, Config, TmpPairInfo, CONFIG, PAIRS, TMP_PAIR_INFO,
};

/// Updates the contract's [Config]
pub fn update_config(
deps: DepsMut,
owner: Option<String>,
fee_collector_addr: Option<String>,
token_code_id: Option<u64>,
pair_code_id: Option<u64>,
) -> Result<Response, ContractError> {
let mut config: Config = CONFIG.load(deps.storage)?;

    if let Some(owner) = owner {
        // validate address format
        let _ = deps.api.addr_validate(&owner)?;

        config.owner = deps.api.addr_canonicalize(&owner)?;
    }

    if let Some(token_code_id) = token_code_id {
        config.token_code_id = token_code_id;
    }

    if let Some(pair_code_id) = pair_code_id {
        config.pair_code_id = pair_code_id;
    }

    if let Some(fee_collector_addr) = fee_collector_addr {
        config.fee_collector_addr = deps.api.addr_validate(fee_collector_addr.as_str())?;
    }

    CONFIG.save(deps.storage, &config)?;

    Ok(Response::new().add_attribute("action", "update_config"))

}

/// Updates a pair config
pub fn update_pair_config(
deps: DepsMut,
pair_addr: String,
owner: Option<String>,
fee_collector_addr: Option<String>,
pool_fees: Option<PoolFee>,
feature_toggle: Option<FeatureToggle>,
) -> Result<Response, ContractError> {
Ok(Response::new()
.add_message(wasm_execute(
deps.api.addr_validate(pair_addr.as_str())?.to_string(),
&terraswap::pair::ExecuteMsg::UpdateConfig {
owner,
fee_collector_addr,
pool_fees,
feature_toggle,
},
vec![],
)?)
.add_attribute("action", "update_pair_config"))
}

/// Creates a Pair
pub fn create_pair(
deps: DepsMut,
env: Env,
asset_infos: [AssetInfo; 2],
pool_fees: PoolFee,
) -> Result<Response, ContractError> {
let config: Config = CONFIG.load(deps.storage)?;

    if asset_infos[0] == asset_infos[1] {
        return Err(ContractError::SameAsset {});
    }

    let asset_1_decimal =
        match asset_infos[0].query_decimals(env.contract.address.clone(), &deps.querier) {
            Ok(decimal) => decimal,
            Err(_) => {
                return Err(ContractError::InvalidAsset {
                    asset: asset_infos[0].to_string(),
                });
            }
        };

    let asset_2_decimal =
        match asset_infos[1].query_decimals(env.contract.address.clone(), &deps.querier) {
            Ok(decimal) => decimal,
            Err(_) => {
                return Err(ContractError::InvalidAsset {
                    asset: asset_infos[1].to_string(),
                });
            }
        };

    let raw_infos = [
        asset_infos[0].to_raw(deps.api)?,
        asset_infos[1].to_raw(deps.api)?,
    ];

    let asset_decimals = [asset_1_decimal, asset_2_decimal];

    let pair_key = pair_key(&raw_infos);
    if let Ok(Some(_)) = PAIRS.may_load(deps.storage, &pair_key) {
        return Err(ContractError::ExistingPair {});
    }

    TMP_PAIR_INFO.save(
        deps.storage,
        &TmpPairInfo {
            pair_key,
            asset_infos: raw_infos,
            asset_decimals,
        },
    )?;

    // prepare labels for creating the pair token with a meaningful name
    let asset0_label = asset_infos[0].clone().get_label(&deps.as_ref())?;
    let asset1_label = asset_infos[1].clone().get_label(&deps.as_ref())?;
    let pair_label = format!("{}-{} pair", asset0_label, asset1_label);

    Ok(Response::new()
        .add_attributes(vec![
            ("action", "create_pair"),
            ("pair", &format!("{}-{}", asset0_label, asset1_label)),
            ("pair_label", pair_label.as_str()),
        ])
        .add_submessage(SubMsg {
            id: 1,
            gas_limit: None,
            msg: CosmosMsg::Wasm(WasmMsg::Instantiate {
                code_id: config.pair_code_id,
                funds: vec![],
                admin: Some(env.contract.address.to_string()),
                label: pair_label,
                msg: to_binary(&PairInstantiateMsg {
                    asset_infos,
                    token_code_id: config.token_code_id,
                    asset_decimals,
                    pool_fees,
                    fee_collector_addr: config.fee_collector_addr.to_string(),
                })?,
            }),
            reply_on: ReplyOn::Success,
        }))

}

pub fn remove_pair(
deps: DepsMut,
\_env: Env,
asset_infos: [AssetInfo; 2],
) -> Result<Response, ContractError> {
let raw_infos = [
asset_infos[0].to_raw(deps.api)?,
asset_infos[1].to_raw(deps.api)?,
];

    let pair_key = pair_key(&raw_infos);
    let pair = PAIRS.may_load(deps.storage, &pair_key)?;

    let Some(pair) = pair else {
        return Err(ContractError::UnExistingPair {});
    };

    PAIRS.remove(deps.storage, &pair_key);

    Ok(Response::new().add_attributes(vec![
        ("action", "remove_pair"),
        (
            "pair_contract_addr",
            deps.api.addr_humanize(&pair.contract_addr)?.as_ref(),
        ),
    ]))

}

/// Adds native/ibc token with decimals to the factory's whitelist so it can create pairs with that asset
pub fn add_native_token_decimals(
deps: DepsMut,
env: Env,
denom: String,
decimals: u8,
) -> Result<Response, ContractError> {
let balance = query_balance(&deps.querier, env.contract.address, denom.to_string())?;
if balance.is_zero() {
return Err(ContractError::InvalidVerificationBalance {});
}

    add_allow_native_token(deps.storage, denom.to_string(), decimals)?;

    Ok(Response::new().add_attributes(vec![
        ("action", "add_allow_native_token"),
        ("denom", &denom),
        ("decimals", &decimals.to_string()),
    ]))

}

pub fn execute_migrate_pair(
deps: DepsMut,
contract: String,
code_id: Option<u64>,
) -> Result<Response, ContractError> {
let config: Config = CONFIG.load(deps.storage)?;
let code_id = code_id.unwrap_or(config.pair_code_id);

    Ok(
        Response::new().add_message(CosmosMsg::Wasm(WasmMsg::Migrate {
            contract_addr: contract,
            new_code_id: code_id,
            msg: to_binary(&PairMigrateMsg {})?,
        })),
    )

}
