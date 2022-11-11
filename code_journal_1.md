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
