# cw-plus/cw1-whitelist

To clone a Github repo like cw-plus, run the following command:

    git clone https://github.com/CosmWasm/cw-plus cw-plus

It will create a folder called cw-plus that contains a collection of specification and contracts designed for use on real networks. They are designed not just as examples, but to solve real-world use cases, and to provide a reusable basis to build many custom contracts.

## Mechanism

The smart contract contains a list of admins, defined during instantiation, that can execute messages via the contract. Think of it as if it were a proxy contract.

If the mutable flag is set to true during instantiation, admins can add (or delete) others admins to the list.

There are 3 types of messages to be sent by admins:

1. `Execute`: Basically, the contract re-dispatches all the messages with the contract's address as the sender address. A list of messages must be passed within the `Execute` message.

2. `Freeze`: used to revoke admins the right to perform further actions, making the contract immutable.

3. `UpdateAdmins`: if the contract is mutable, the list of admins can be modified by any of the current admins. A new list of all admins should be sent along the `UpdateAdmins` message.

Anyone can query if a certain address is part of the admin list or if it can modify the contract. Both queries return a a boolean type.

The unit tests check that all functions (instantiate, execute, freeze, update admins, query_admin_list, and query_can_execute) work as they should. They also check that the contract throws an error when anyone different from the admins try to send a message on behalf of the contract or if they try to modify the admin list.

## Organisation

The smart contract is comprise of different files or crates:

1. The contract.rs file is the main file that contains the logic of our program. It has all the entry points of our contract that are exposed to the blockchain to interact with.

2. The error crate is where the error types are defined.

3. The integration_tests.rs file is used to test that our smart contract behaves as it should (or as we think it should). It's impossible to test every scenario, but it's always a good practice to test core functionallity and interaction between parts of our contract.

4. The lib.rs file works like an index. It tells cargo how ours files and folders are structured so we can import them into our contract files. We'll have to manually add to it as we create new files and folders to our smart contract.

5. We define the messages we're going to be using in the contract in the msg.rs file. Messages are usually structs or enums, depending on the case. Responses from queries are defined here too.

6. The state.rs file is where we define how we're going to keep the state of our contract. Usually, we define here the `state` as a struct and then a `STATE` constant that's going to wrap the `state` either in an Item or a Map, depending on the logic of our contract. This is going to be stored on-chain and we can query it, update it or create new instances.

7. Cargo.toml is where we want to include our dependencies. As we need to use external crates or packages, we should manually add them under `[dependencies]`. **Tip**: you can use VSCODE extension called Better TOML to format the files with .toml extension.

## State

The macro `derive` is used to implement specified traits for the following structure.

    #[derive(Serialize, Deserialize, Clone, PartialEq, Eq, JsonSchema, Debug, Default)]

We create a struct called `AdminList` that contains two fields:

1. A list of admin addresses in the form of a vector. Vectors are basically lists that can grow in size.

2. A mutable field that, when set to true, allows the admins in the list to add / remove other admins and execute messages via the smart contract.

   pub struct AdminList {
   pub admins: Vec<Addr>,
   pub mutable: bool,
   }

The `AdminList` struct has two implementations that are defined as functions:

1.  `is_admin` takes the structure itself and an address (as a sliced string - &str) and returns either `true` or `false` if the `addr` is contained in the `admins` list.

2.  `can_modify` takes the structure itself and an address (as a sliced string - &str) and returns either `true` or `false` if the following conditions are met:

    1. the `addr` is contained in the `admins` list
    2. the `mutable` field is set to `true`

    impl AdminList {
    /// returns true if the address is a registered admin
    pub fn is_admin(&self, addr: impl AsRef<str>) -> bool {
    let addr = addr.as_ref();
    self.admins.iter().any(|a| a.as_ref() == addr)
    }

        /// returns true if the address is a registered admin and the config is mutable
        pub fn can_modify(&self, addr: &str) -> bool {
            self.mutable && self.is_admin(addr)
        }

    }

`ADMIN_LIST` is defined as a public constant that will hold the `AdminList` struct inside as an `Item` with `admin_list` as its storage_key. This is where we'll store our state in the blockchain and can be saved, modified and updated accordingly.

    pub const ADMIN_LIST: Item<AdminList> = Item::new("admin_list");

## Contract

In this section, I'm going to explain the most relevant pieces that conform the contract.

### Instantiate

Entry points, or handlers, are where messages or queries are handled by the contract. This macro explicitly flags the following function as an entry point and excludes it from being bundled in the library.

    #[cfg_attr(not(feature = "library"), entry_point)]

Instantiate messages, as defined by the `InstantiateMsg` struct, are handled by the instantiate function. `instantiate` works similar to the constructor function in other programming languages. It's used to initialize the contract with the default values after deployment in the blockchain and saves the state. It returns a `Response` wrapped into a `StdResult` which handles errors if there were any with. If there're no errors, `cfg` (short for configuration) is saved into the `ADMIN_LIST`, stored on-chain, and returns a default `Response` (wrapped into a `StdResult`); otherwise, `?` will handle the error.

    pub fn instantiate(
        deps: DepsMut,
        _env: Env,
        _info: MessageInfo,
        msg: InstantiateMsg,
    ) -> StdResult<Response> {
        set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;
        let cfg = AdminList {
            admins: map_validate(deps.api, &msg.admins)?,
            mutable: msg.mutable,
        };
        ADMIN_LIST.save(deps.storage, &cfg)?;
        Ok(Response::default())
    }

Because we pass admin addresses as strings, we need to validate them before we save them into the `admins` list, or return an error if any.

    pub fn map_validate(api: &dyn Api, admins: &[String]) -> StdResult<Vec<Addr>> {
        admins.iter().map(|addr| api.addr_validate(addr)).collect()
    }

### Execute

The `execute` function in this contract is passing an `Empty` message to be executed, but this can be changed to use a different type to add support for custom messages.

    pub fn execute(
        deps: DepsMut,
        env: Env,
        info: MessageInfo,
        msg: ExecuteMsg<Empty>,
    ) -> Result<Response<Empty>, ContractError> {
        match msg {
            ExecuteMsg::Execute { msgs } => execute_execute(deps, env, info, msgs),
            ExecuteMsg::Freeze {} => execute_freeze(deps, env, info),
            ExecuteMsg::UpdateAdmins { admins } => execute_update_admins(deps, env, info, admins),
        }
    }

There are 3 different types of messages handled by the `match msg` control flow and mapped to its respective function. We defined them in a previous section of this document.

    match msg {
        ExecuteMsg::Execute { msgs } => execute_execute(deps, env, info, msgs),
        ExecuteMsg::Freeze {} => execute_freeze(deps, env, info),
        ExecuteMsg::UpdateAdmins { admins } => execute_update_admins(deps, env, info, admins),
    }

In the `execute_execute` function, `where` is used to add the following traits to the block.

    where
        T: Clone + fmt::Debug + PartialEq + JsonSchema,
    {

If the sender of the transaction can't execute messages (a.k.a. is not in the admin list) we return an `Unauthorized` error.

    if !can_execute(deps.as_ref(), info.sender.as_ref())? {
        Err(ContractError::Unauthorized {})
    }

Otherwise, we send a `Response` and bulk add messages to the list of messages to process (fire and forget). We also add to the `Response` the attribute shown below to be indexed later by either the front end or the block explorer.

    else {
        let res = Response::new()
            .add_messages(msgs)
            .add_attribute("action", "execute");
        Ok(res)
    }

In the `execute_freeze` function, we need to lead the `ADMIN_LIST` from storage and save it to the `cfg` variable. If there's an error, then it'll be handled by the `?` operator.

    let mut cfg = ADMIN_LIST.load(deps.storage)?;

Again, if the sender of the transaction is not allowed to modify the `AdminList` (a.k.a. is not in the admin list) we return an `Unauthorized` error.

    if !cfg.can_modify(info.sender.as_ref()) {
        Err(ContractError::Unauthorized {})
    }

Otherwise, we change the `mutable` field of `AdminList` to `false`, save `cfg` to storage, and return a `response` wrapped into a `Result` with its respective attribute.

    else {
        cfg.mutable = false;
        ADMIN_LIST.save(deps.storage, &cfg)?;

        let res = Response::new().add_attribute("action", "freeze");
        Ok(res)
    }

The `execute_update_admins` function takes a list of the new admins and saves it to storage, after validating each address for separate, only if the sender is allowed to modify the contract. Otherwise, an error is returned.

    pub fn execute_update_admins(
        deps: DepsMut,
        _env: Env,
        info: MessageInfo,
        admins: Vec<String>,
    ) -> Result<Response, ContractError> {
        let mut cfg = ADMIN_LIST.load(deps.storage)?;
        if !cfg.can_modify(info.sender.as_ref()) {
            Err(ContractError::Unauthorized {})
        } else {
            cfg.admins = map_validate(deps.api, &admins)?;
            ADMIN_LIST.save(deps.storage, &cfg)?;

            let res = Response::new().add_attribute("action", "update_admins");
            Ok(res)
        }
    }

### Query

The `query` function is similar to the `execute` function but there're two main differences. Dependencies are immutable (read-only) and instead of returning a `Response`, it returns a `StdResult` wrapping a `Binary`. There are two type of query messages that are handled by the `match msg` control flow, which maps each one to their corresponding function.

    #[cfg_attr(not(feature = "library"), entry_point)]
    pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
        match msg {
            QueryMsg::AdminList {} => to_binary(&query_admin_list(deps)?),
            QueryMsg::CanExecute { sender, msg } => to_binary(&query_can_execute(deps, sender, msg)?),
        }
    }

The `query_admin_list` function loads the `ADMIN_LIST` from storage and returns a tailored `AdminListResponse` wrapped into a `StdResult` type that contains the list of admins and whether the contract is mutable or not.

    pub fn query_admin_list(deps: Deps) -> StdResult<AdminListResponse> {
        let cfg = ADMIN_LIST.load(deps.storage)?;
        Ok(AdminListResponse {
            admins: cfg.admins.into_iter().map(|a| a.into()).collect(),
            mutable: cfg.mutable,
        })
    }

`query_can_execute` checks if the sender is allowed to execute messages and also returns a custom `CanExecuteResponse` struct which has a `true` or `false` value inside.

    pub fn query_can_execute(
        deps: Deps,
        sender: String,
        _msg: CosmosMsg,
    ) -> StdResult<CanExecuteResponse> {
        Ok(CanExecuteResponse {
            can_execute: can_execute(deps, &sender)?,
        })
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

We now double check that the instantiate function is properly working. For that, we'll use the `query` fucntion passing the `GetCount` parameter, unwrap it to make sure if doesn't fail, and then we need to convert it back `from_binary` to be able to compare the value obtained with the hardcoded `count` previously set. The dependencies are passed as reference when querying the contract because we don't need to modify blockchain state.

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

This is a very basic contract that needs to be tweaked a little bit in order to make it useful. It's meant to be used as a skeleton to have the structure ready to add some extra custom logic to it.

1. The error `Unauthorized` can be more specific, telling that only the admins of the contract can either modify the admin list or send messages.

2. The Empty Message we pass to the `execute` function can be modified to be of a different type to add support for custom messages

3. For safety, it's better to add / remove admins instead of passing the whole list every time we want to make changes. If we make a mistake with the list or send it empty, we might lose control of the contract and its funds.

4. Checking if the admin list is empty can prevent losing control of the contract, not allowing to save it to storage and returning an error.
