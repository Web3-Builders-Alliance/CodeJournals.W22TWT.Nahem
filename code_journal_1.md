# cw-template

To create a project using cargo from an specific Github repo, run the following command:

    cargo generate --git https://github.com/CosmWasm/cw-template.git --name PROJECT_NAME

This time we're using the cw-template from the CosmWasm repo with full features (the non-minimal version).

## Organisation

Cargo creates a new folder named PROJECT_NAME with some files and folders inside. Here's a short description of the most relevant files/folders that cargo creates for us.

1. The .cargo file is a hidden folder that contains the configuration file for cargo. It's mainly used to create aliases and make cargo commands shorter and easier to visualise or remember.

2. The circleci and github/workflow folders contain configuration for continious integration. We're not digging into it yet as this is useful for larger projects and teams to automate checks when uploading code to Git repos.

3. The src folder is where the magic happens, it contains the pieces that, together, define the smart contracts of our project.

   1. There's a bin folder with a schema.rs file. This file is used to create the schema/PROJECT_NAME.json file when running the `cargo schema` command.

   2. The contract.rs file is the main file that contains the logic of our program. It has all the entry points of our contract that are exposed to the blockchain to interact with.

   3. Inside the error.rs file is where we define the error types our contract is going to use. The more specific the error, the better UX.

   4. If we want to keep our code clean, we might want to use helper functions to not repeat ourselves too much. We can create functions here that will be used along the contract here and import them later on.

   5. The integration_tests.rs file is used to test that our smart contract behaves as it should (or as we think it should). We must write tests to check if our program is actually doing what we think we want, and fix bugs until it does. It's impossible to test every scenario, but it's always a good practice to test core fucntionallity and interaction between parts of our contract.

   6. The lib.rs file works like an index. It tells cargo how ours files and folders are structured so we can import them into our contract files. We'll have to manually add to it as we create new files and folders to our smart contract.

   7. We define the messages we're going to be using in the contract in the msg.rs file. Messages are usually structs or enums, depending on the case. Responses from queries are defined here too.

   8. The state.rs file is where we define how we're going to keep the state of our contract. Usually, we define here the `state` as a struct and then a `STATE` constant that's going to wrap the `state` either in an Item or a Map, depending on the logic of our cotnract. This is going to be stored on-chain and we can query it, update it or create new instances.

4. The .editorconfig file is used to format the workspace.

5. The .gitignore file tells git what files should not be uploaded to the online repos. This is very useful for keeping private data and files locally, like private keys, apikeys and anything we don't want to expose to the Internet.

6. The cargo.lock file is created by cargo and it's not intended to be manually edited.

7. Cargo.toml is where we want to include our dependencies. As we need to use external crates or packages, we should manually add them under `[dependencies]`. **Tip**: you can use VSCODE extension called Better TOML to format the files with .toml extension.

## Code

`use` is similar to import in other programming languages. In this case we're importing the struct `state` and the const `STATE` that are in the state.rs file so we can use them in our smart contract without passing the complete path.

> use crate::state::{State, STATE};

`const` is the short for constant and is used to define values in our contract that won't change. For naming convenction, we should use uppercase and separate words with `_`. It's always necessary to define the type with `:`,in this case is a `String` slice.

> const CONTRACT_NAME: &str = "crates.io:cw-template";

This is a macro to define that the following function will be an entry point for the program. `entry_point` are exposed to the blockchain and can be interacted with.

> #[cfg_attr(not(feature = "library"), entry_point)]

`instantiate` is a public function that's accesible from other files outside this contract. It's similar to a constructor function in other programming languages. Basically, it's used to initialize the contract with the default values after deployment in the blockchain. The values inside `()` are the variables we pass to the function so they can be used inside its scope. We always need to specify its type and we're going to talk about them below. The `->` tells what type of output the function should return, in this case, it's either a `Response` when everything goes as planned or a `ContractError` if something went wrong, wrapped in a Result type to contain the error and not cause a kernel panic. Then, the actual commands of the function are found inside the `{}` which are explained below line by line.

    pub fn instantiate(
        deps: DepsMut,
        \_env: Env,
        info: MessageInfo,
        msg: InstantiateMsg,
    ) -> Result<Response, ContractError> { ... }

`deps` is short for dependencies and there're of type `DepsMut`, which means there're mutable (their value can be changed or mutated). We usually pass dependecies to access storage.

> deps: DepsMut,

`env` is short for environment and this is where the blockchain state is usually maintained. Things like block `time`, `height`, `chain_id` etc. are stored inside `env`. In this case there's a `_` before the variable to tell the compiler we're not using this variable inside the function, so there's no warning at compile time. There's no use in passing variables we're not going to use inside the function, but this time we do it to keep things standardised.

> \_env: Env,

`info` is short for information and is of type `MessageInfo`. It's a struct that contains the `sender` which signed the transaction and the `funds` that are sent to the contract.

> info: MessageInfo,

`msg` is short for message and is of type `InstantiateMsg`. This is a custom type we previously defined in the msg.rs file and will contain the fields needed to intantiate the contract. In our case we need to pass the value at which we want to initialise the counter.

> msg: InstantiateMsg,

`let` is the keyword we use to define a variable in Rust. Variables are immutable by default until we say otherwise. In this case we're defining a variable `state` which is of type `State` and has two fields: `count` and `owner`. We get the `count` from the value passed in the `msg` and the owner in this case is whoever signs the transaction, `info.sender`. Note that we `.clone` the value because we use it later when adding attributes to the `Response`. and in Rust, there can only be one owner of a variable at a time.

    let state = State {
        count: msg.count,
        owner: info.sender.clone(),
    };

The `set_contract_version( ... )` function is used when we instantiate the contract to store the original version in the blockchain so we can then migrate it if changes in the code are needed. We pass as parameters `deps.storage` to save configuration on-chain, also `CONTRACT_NAME` and `CONTRACT_VERSION` constants defined before.

> set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;

`STATE.save( ... )` is used to store the state of the contract that we just defined on-chain. We pass `deps.storage` to give the contract mutable access to storage and `&state` to save the initial configuration. Note that we use `&` in this case, which means we are borrowing the variable `state` so we don't own it (cannot be modified). There's also a `?` which is used in Rust to handle errors. If something went wrong, we'll return a `ContractError` wrapped in a `Result`.

> STATE.save(deps.storage, &state)?;

If anything broke before, it means that the function didn't throw any errors so we should return an `Ok(Response::new() ... )` with some attributes to help index the transaction later on. The attributes we pass are discretionary and should be related to what the function did, in this case, we define a `method` called `instantiate` and add the `owner` and `count` we just set.

    Ok(Response::new()
        .add_attribute("method", "instantiate")
        .add_attribute("owner", info.sender)
        .add_attribute("count", msg.count.to_string()))
    }

The `pub fn execute( ... )` function is similar to the instantiate function but is a bit more complex as it contains all possible executable messages our contract needs to handle.

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
