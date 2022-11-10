# Code Study Journals for Winter 2022-23 Cohort - WBA

Mastery of a skill comes from immersing yourself in the study and use of it in many different ways.

Here are the steps to complete each one:

1. Select a piece of code/ smart contract. Sometimes, this will be your own smart contract, but to start there will be choices provided, or you can find a sample on your own- CW-Plus is a great resource you will learn to use during this course. It is a collection of CosmWasm smart contracts, many of which are production ready. The faster you get comfortable with CW Plus, the better off you will be.
2. You will do all steps in the github classroom repo.
3. You should annotate the code- in other words, make notes on the document explaining what is happening through the code.

   1. What are the concepts (borrowing, ownership, vectors etc)
   2. What is the organization?
   3. What is the contract doing? What is the mechanism?
   4. How could it be better? More efficient? Safer?
   5. During the first month, you will make changes to the code that you think would make it better. Or write your own versions.

4. Once a month, we will ask you to make a video of your code study, and show an example of how you made the code better, test, compile, and deploy it on video.
5. As we get further into the course, (when we get into CW and Smart contracts more deeply), you will also be able to select code from other chains/ in other languages and convert them into Rust/CW.

We ask that you interpret and apply these instructions in the way you understand them. Not everyone will do so the same way. There are no wrong answers- the only way to fail this part is to not do it and ship on time.

We are interested in how YOU do this process.

## EXAMPLE:

    use cosmwasm_std::Uint128; // You can comment on the side, like this
    use schemars::JsonSchema;
    use serde::{Deserialize, Serialize};

    use cw20::{Cw20ReceiveMsg, Denom};
    pub use cw_controllers::ClaimsResponse;
    use cw_utils::Duration;

    #[derive(Serialize, Deserialize, Clone, PartialEq, JsonSchema, Debug)]
    pub struct InstantiateMsg {
        /// denom of the token to stake
        pub denom: Denom,
        pub tokens_per_weight: Uint128,
        pub min_bond: Uint128,
        pub unbonding_period: Duration,

        // admin can only add/remove hooks, not change other parameters
        pub admin: Option<String>,
    }

    YOU CAN ALSO BOLD/ CAPS AND COMMENT ABOVE A SECTION LIKE THIS.

    #[derive(Serialize, Deserialize, Clone, PartialEq, JsonSchema, Debug)]
    #[serde(rename_all = "snake_case")]
    pub enum ExecuteMsg {
        /// Bond will bond all staking tokens sent with the message and update membership weight
        Bond {},
        /// Unbond will start the unbonding process for the given number of tokens.
        /// The sender immediately loses weight from these tokens, and can claim them
        /// back to his wallet after `unbonding_period`
        Unbond { tokens: Uint128 },
        /// Claim is used to claim your native tokens that you previously "unbonded"
        /// after the contract-defined waiting period (eg. 1 week)
        Claim {},

        /    Hooks {},

    THEN YOU CAN COPY THE QUESTIONS BELOW LIKE THIS AND ANSWER RIGHT BELOW EACH ONE.

    What are the concepts (borrowing, ownership, vectors etc)
    The Concepts in the Code are… they are —---
    What is the organization?
    The code is organized……
    What is the contract doing? What is the mechanism?
    The contract is a staking contract and the mechanism…

    How could it be better? More efficient? Safer?
    The code could be safer and better if…..
    During the first month, you will make changes to the code that you think would make it better.
    After Cluster 2, we will ask you to imitate the code in your own way and write your own as part of the process.
