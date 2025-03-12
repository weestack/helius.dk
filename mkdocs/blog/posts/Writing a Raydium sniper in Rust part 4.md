---
date:
  created: 2025-03-12 
authors:
  - weestack
comments: true
blog_toc: true
categories:
  - Solana Sniper
tags:
    - Solana
---
![AI generated image of a friendly humanoid bot](/static/raydium_sniper_part_4/ai bot wrapped in paper.jpg)
# Writing a Raydium sniper in Rust part 4
This is part four of this series, where we are creating a Solana sniper bot for Raydium from scratch in Rust. See the first post here which includes an introduction about why we use a compiled language instead of JavaScript, as well as prerequisites and a brief introduction about me. You can find the latest [GitHub source here](https://github.com/weestack/Solana-Sniper), which will be updated with each new post.  
  
Solana needs to be wrapped for most meme coin swaps, but is there another reason for this command?
<!-- more -->

## Disclaimer
The information provided in this series of articles is for educational purposes only and should not be considered financial or investment advice. Trading, including sniping tokens, carries significant risks, and there is no guarantee of profit.

Any decisions you make based on the content shared here are solely your responsibility. I am not liable for any financial losses incurred as a result of implementing the strategies, code, or techniques discussed in these articles.

Always trade responsibly. Never risk more money than you can afford to lose. It is essential to conduct your own research and consult with a professional financial advisor if necessary before engaging in any trading activities.


## What to expect today
Last time, we explored swap and initialization transactions, delving deeper into how to extract actionable data from Solana's blockchain for efficient token sniping. This includes identifying transaction patterns.
  
Today, we will learn how to build our own CLI command to wrap SOL to wSOL, the common currency for swapping meme coins. Additionally, we will have a short discussion on transaction optimization.

## Wrapped Solana
The first question that might come to mind is: why do I need to wrap my Solana, and what does it even mean to wrap it?
  
Solana (SOL) is the native token of the Solana blockchain, used for gas fees and staking.
wSOL, however, is an SPL token with the exact same value as SOL. Just like any meme coin is created as an SPL token, wSOL is also created using Solana's SPL token program.
  
Holding SOL in the form of wSOL means you are always ready to trade without needing to add extra instructions for wrapping your SOL when a new meme coin is released.

## Creating a cli command
Now that we’ve covered the basics, let's create a CLI command to convert our regular, boring SOL into hot-swappable wSOL.

!!! tip "Create a new package with"

    ```Bash
    cargo new shell
    ```
    Create the following files:
    ```Bash
    shell/Cargo.toml
    shell/src/main.rs
    shell/src/commands/mod.rs
    shell/src/wrap_sol.rs
    ```

With that, we have created all the files needed for the wrap SOL command we are going to build.

!!! tip "Fill `shell/Cargo.toml` with"
    ```toml
    [package]
    name = "shell"
    authors.workspace = true
    edition.workspace = true
    homepage.workspace = true
    repository.workspace = true
    license.workspace = true
    keywords.workspace = true
    version.workspace = true
    readme.workspace = true
    categories.workspace = true
    publish.workspace = true
    
    [dependencies]
    clap = { workspace = true }
    utils = { workspace = true }
    solana-client = { workspace = true }
    solana-sdk = { workspace = true }
    spl-token-client = { workspace = true }
    spl-token = { workspace = true }
    tokio = { workspace = true }
    
    [lints]
    workspace = true
    ```
  
We have added `clap`, a common Rust library for creating CLI commands, which provides an easy syntax for defining command-line arguments directly from structs and supports hinting.
  
In addition to `clap` we have also introduced the `spl-token-client` and `spl-token` libraries, which are used for wrapping SOL to wSOL. 

!!! tip "Fill `shell/src/main.rs` with"
    ```Rust linenums="1"
    mod commands;
    use commands::wrap_sol::WrapArgs;
    use clap::{Parser, Subcommand};
    use solana_sdk::native_token::LAMPORTS_PER_SOL;
    use utils::env::env::Env;
    use crate::commands::wrap_sol;
    
    #[derive(Debug, Parser)] // requires `derive` feature
    #[command(name = "Solana Commands")]
    #[command(about = "CLI solana commands", long_about = None)]
    struct Cli {
        #[command(subcommand)]
        command: Commands
    }
    
    #[derive(Clone, Debug, Subcommand)]
    enum Commands {
        WrapSol(WrapArgs),
    }
    
    #[tokio::main]
    async fn main() {
        let env = Env::new().unwrap();
        let cli_command = Cli::parse();
    
        match cli_command.command {
            Commands::WrapSol(wrap_args) => {
                let amount = (wrap_args.amount as f64 * LAMPORTS_PER_SOL as f64) as u64;
    
                wrap_sol::wrap_sol_fn(
                    amount,
                    &env.private_key,
                    env.rpc_endpoint.to_string(),
                ).await
            }
        }
    }
    ```
A lot happens in the 37 lines of code above in `main.rs`, so lets break it down:
  
- At line 9, we define our shell and name it Solana Commands.
- At line 10, we provide a short description.
- From lines 11 to 14, we create the actual struct for the command shell, which takes in an `Enum Commands`
At lines 17 to 19, we define our first command, `WrapSol` with the arguments `WrapArgs` we will cover in another file. `clap` automatically parses `WrapSol` as `wrap-sol`, enabling us to run the shell command: `cargo run wrap-sol`

![CLI output of running cargo run shell](/static/raydium_sniper_part_4/Screenshot 2025-03-12 at 22.01.00.png)
/// caption
CLI output of running `cargo run shell`
///

The great thing about this layout is that if you wanted to create additional commands, you would simply add a new enum variant to `Commands. You could then call it using: `cargo run shell <new_command_name>` and handle it within main.rs using pattern matching.  

- From lines 26 to 36, we implement a simple match statement to handle WrapSol when it is passed as a command-line argument.

  
!!! tip "Fill `shell/src/commands/mod.rs` with"
    ```Rust linenums="1"
    pub mod wrap_sol;
    ```
We are making wrap_sol public so that it can be called elsewhere.

!!! tip "Fill `shell/src/commands/mod.rs` with"
    ```Rust linenums="1"
    use std::sync::Arc;
    use solana_client::nonblocking::rpc_client::RpcClient;
    use solana_sdk::signature::{Keypair};
    use solana_sdk::signer::Signer;
    use solana_sdk::transaction::Transaction;
    use spl_token_client::client::{ProgramClient, ProgramRpcClient, ProgramRpcClientSendTransaction};
    use spl_token_client::token::Token;
    
    #[derive(Debug, clap::Args, Clone)]
    pub struct WrapArgs {
        #[arg(
                long,
        )]
        pub amount: f32,
    }
    
    fn rpc(rpc_endpoint: String) -> Arc<RpcClient> {
        Arc::new(RpcClient::new(rpc_endpoint.to_string()))
    }
    fn program_rpc(rpc: Arc<RpcClient>) -> Arc<dyn ProgramClient<ProgramRpcClientSendTransaction>> {
        let program_client: Arc<dyn ProgramClient<ProgramRpcClientSendTransaction>> = Arc::new(
            ProgramRpcClient::new(rpc.clone(), ProgramRpcClientSendTransaction),
        );
        program_client
    }
    
    fn keypair_clone(kp: &Keypair) -> Keypair {
        Keypair::from_bytes(&kp.to_bytes()).expect("failed to copy keypair")
    }
    
    pub async fn wrap_sol_fn(wrap_amount: u64, keypair: &Arc<Keypair>, rpc_endpoint: String) {
        let wsol_wrap_amount: u64 = wrap_amount;
        let client = rpc(rpc_endpoint);
        let program_client = program_rpc(Arc::clone(&client));
        
        /* prepare the wSOL account */
        let in_token_client = Token::new(
            Arc::clone(&program_client),
            &spl_token::ID,
            &spl_token::native_mint::id(),
            None,
            Arc::new(keypair_clone(&keypair)),
        );
    
        let user_in_token_account = in_token_client.get_associated_token_address(&keypair.pubkey());
        let wsol_acc_exists = in_token_client
            .get_account_info(&user_in_token_account)
            .await;
    
        /* ensure wsol token program has not been closed! */
        if wsol_acc_exists.is_err() {
            in_token_client.create_associated_token_account(
                &keypair.pubkey()
            ).await.unwrap();
        }
    
        let user_in_acct = in_token_client
            .get_account_info(&user_in_token_account)
            .await.unwrap();
    
        let balance = user_in_acct.base.amount;
        if in_token_client.is_native() && balance < wsol_wrap_amount {
            let transfer_amt = wsol_wrap_amount - balance;
            let blockhash = client.get_latest_blockhash().await.unwrap();
            let transfer_instruction = solana_sdk::system_instruction::transfer(
                &keypair.pubkey(),
                &user_in_token_account,
                transfer_amt,
            );
            let sync_instruction =
                spl_token::instruction::sync_native(&spl_token::ID, &user_in_token_account).unwrap();
            let tx = Transaction::new_signed_with_payer(
                &[transfer_instruction, sync_instruction],
                Some(&keypair.pubkey()),
                &[&keypair],
                blockhash,
            );
    
            let signature = client.send_transaction(&tx).await.unwrap();
            println!("signature {signature:?}");
        }
    }
    ```
A lot happened again, so let's go over it step by step.  

Lines 9 to 15 define the `WrapArgs struct, which takes a float as an argument to specify how much SOL to wrap into wSOL.


### **WSOL transaction**
Now, let's break down the instructions from lines 32 to 80.

```Rust
/* prepare the wSOL account */
let in_token_client = Token::new(
    Arc::clone(&program_client),
    &spl_token::ID,
    &spl_token::native_mint::id(),
    None,
    Arc::new(keypair_clone(&keypair)),
);
```
Lines 37 to 43: We simply create a struct with the necessary information for the functions used later.



```Rust
let user_in_token_account = in_token_client.get_associated_token_address(&keypair.pubkey());
let wsol_acc_exists = in_token_client
    .get_account_info(&user_in_token_account)
    .await;
```
Line 45: We check if we can derive our wSOL account address and in lines 46 to 48: We fetch that address.


```Rust
/* ensure wsol token program has not been closed! */
if wsol_acc_exists.is_err() {
    in_token_client.create_associated_token_account(
        &keypair.pubkey()
    ).await.unwrap();
}
```
Lines 51 to 55: We create the wSOL address if it does not already exist.

```Rust
let user_in_acct = in_token_client
    .get_account_info(&user_in_token_account)
    .await.unwrap();
```
Line 57: We fetch our SOL account address.

That wasn’t so bad, right? 😉

The remainder of the code simply handles the amount, signing, and actually sending the transaction. It then prints the transaction signature so we can look it up on-chain using sites like [solana.fm](solana.fm) and [solscan.io](solscan.io).  

### Adding Private Key to Environment Variables
!!! danger "A word of caution:"
    While researching how to build my own Solana sniper, I reviewed numerous codebases, and almost all had one thing in common—they were malicious honeypots designed to steal unsuspecting victims' private keys and drain their entire SOL balance.   
    Always be extremely careful with the code you execute and where you store your private key. Even better, make sure you understand each line of the code before running any program that involves your private key.

The final step for wrapping SOL is to add our private key to our environment variables (env) so it can be used for signing and approving transactions in the `wrap-sol` command.
!!! tip "Update `pub fn new()` in `utils/src/env/env.rs` to"
    ```Rust linenums="1"
    pub fn new() -> Result<Self, EnvErrors> {
    // Test if .env exists or give the option to create one from .env.dist
    let path = Path::new(".env");
    if ! path.exists() {
        println!("Could not find .env, would you like to use .env.dist as a template instead? (y/n)");
        create_env().expect("Failed creating .env file");
    }
    dotenv::from_path(".env").ok();
    
    let loglevel = Arc::new(parse_log_level(env::var("LOG_LEVEL")?));
    
    let websocket_endpoint = Arc::new(env::var("WEBSOCKET_ENDPOINT")?);
    let rpc_endpoint = Arc::new(env::var("RPC_ENDPOINT")?);
    
    let private_key_path = env::var("PRIVATE_KEYPAIR")?;
    let private_key = Arc::new(Keypair::read_from_file(private_key_path).unwrap());
    
    let swap_amount = Arc::new(env::var("SWAP_AMOUNT")?.parse::<u64>().unwrap());
    let swap_priority_fee = Arc::new(env::var("SWAP_PRIORITY_FEE")?.parse::<u64>().unwrap());
    
    Ok(
        Self {
            loglevel,
            websocket_endpoint,
            rpc_endpoint,
            private_key,
            swap_amount,
            swap_priority_fee,
        }
    )
    }
    ```

Lines 15 to 16 read the `PRIVATE_KEYPAIR` directive from the .env file and load the private keypair into memory.

## Test run
With all that set up, let's test it out on the devnet and see if it works.

Before using your own private key, let's run a simple test on the Solana devnet using a newly generated random keypair and observe the results.

!!! Tip "Be careful not to overwrite your current private key with this command unless you have a backup."

    ```bash
    # Create a new key called private_key.json
    solana-keygen new -o private_key.json
    
    # Switch default config to use devnet
    solana config set --url https://api.devnet.solana.com
    
    # Get the public key of your wallet mine was 3Vy1sD9fwzUv6npTE8qM2o4DJGRRXdLryhyKfmjZByGu
    solana-keygen pubkey private_key.json
    
    # Request an airdrop with fake SOL on the devnet
    solana airdrop 2 3Vy1sD9fwzUv6npTE8qM2o4DJGRRXdLryhyKfmjZByGu
    ```

Lastly, edit the .env file and add or replace this line: `PRIVATE_KEYPAIR=private_key.json`  
Also, set the RPC_ENDPOINT to devnet while we are testing: `RPC_ENDPOINT=https://api.devnet.solana.com`


Once you have executed the commands from the previous steps, you should be all set to wrap some SOL into wSOL!

![Solana.fm with my wallet](/static/raydium_sniper_part_4/2025-03-12_23-04.png)
/// caption
Solana.fm with my wallet
///

Lets execute
```Bash
cargo run --bin shell wrap-sol --amount 0.2
```
so we exchange 0.2 SOL for 0.2 WSOL.
![Executing wsol wrap command](/static/raydium_sniper_part_4/Screenshot 2025-03-12 at 23.12.12.png)
/// caption
Executing wSOL wrap command
///

As you can see, I got the [signature](https://solana.fm/tx/MY4VpvZig1H6y2KHB9EuQuqYaZh7Yx7nEbUtHHhhkdxHxQcaZbgr5Yh13UxRq4U2TYJnfPURJ5au5PWTH8TqXnU?cluster=devnet-alpha).

!!! node "If you get this error, it likely means you either forgot to change the RPC_ENDPOINT to devnet or did not successfully receive the 2 SOL airdrop."
    ![Executing wsol wrap command error](/static/raydium_sniper_part_4/Screenshot 2025-03-12 at 23.16.25.png)
    /// caption
    Error wrapping sol
    ///

Great! Now, how do we confirm that the SOL was actually wrapped and not just burned? 
Navigate to [my wallet](https://solana.fm/address/3Vy1sD9fwzUv6npTE8qM2o4DJGRRXdLryhyKfmjZByGu/tokens?cluster=devnet-alpha) to check.

![Displaying wrapped sol](/static/raydium_sniper_part_4/2025-03-12_23-21.png)
/// caption
Displaying wrapped sol
///

## Why are we wrapping sol in a command?
We are wrapping SOL in a command because each instruction in a transaction increases computational usage (measured in compute units or CUs). The more CUs a transaction consumes, the less likely it is to be processed quickly.

Solana is designed to fit as many transactions as possible into a single block. Once a block is nearly full, some transactions will have to wait for the next block. However, if your swap transaction is small enough, it has a higher chance of being included faster potentially ahead of other, larger transactions that might be skipped. 

## Up next
In the next article, we will look into how to change our SOL to WSOL, which is needed for swapping, we will do this with a command line layer.