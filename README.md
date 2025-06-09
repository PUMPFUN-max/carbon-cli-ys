# Carbon CLI YS
You can now pass the yellowstone compiled instruction type to the generated decoder. 
Official carbon cli link: https://github.com/sevenlabs-hq/carbon
## Overview

Carbon CLI streamlines the development of Solana transaction parsing infrastructure by:

- **IDL Parsing**: Converting Anchor and Codama IDL files into Rust decoders
- **Code Generation**: Creating type-safe parsers for accounts, instructions, events, and types




## Installation

```bash
git clone <repository-url>
cd carbon-cli-ys
cargo build --release
```

## Quick Start

### Parse an IDL file
```bash
# Parse from local IDL file
cargo run parse --idl ./path/to/program.json --output ./decoders --as-crate

# Parse from program address
cargo run parse --idl <PROGRAM_ADDRESS> --url mainnet-beta --output ./decoders --as-crate
```

### Scaffold a new project
DONT USE IT, IT WILL ADD THE CRATES FROM ACTUAL CARBON CLI.

## Commands

### `parse`
Generate decoder code from IDL files or program addresses. 

**Options:**
- `-i, --idl <SOURCE>` - IDL file path (.json) or Solana program address
- `-o, --output <PATH>` - Output directory for generated code
- `-c, --as-crate` - Generate as a Rust crate (includes Cargo.toml)
- `-s, --standard <STANDARD>` - IDL standard: `anchor` (default) or `codama`
- `-e, --event-hints <HINTS>` - Comma-separated event type names (Codama only)
- `-u, --url <URL>` - Network URL: `mainnet-beta`, `devnet`, or custom RPC URL

**Examples:**
```bash
# Parse Anchor IDL from file
cargo run parse --idl ./jupiter.json --output ./decoders --as-crate

# Parse from program address
cargo run parse --idl JUP4Fb2cqiRUcaTHdrPC8h2gNsA2ETXiPDD33WcGuJB --url mainnet-beta --output ./decoders --as-crate

# Parse Codama IDL with event hints
cargo run parse --idl ./program.json --standard codama --event-hints SwapEvent,LiquidityEvent --output ./decoders --as-crate
```



## Generated Code Structure

```
generated-decoder/
├── Cargo.toml          # Rust crate configuration
├── src/
│   ├── lib.rs          # Main library file
│   ├── accounts/       # Account type definitions
│   ├── instructions/   # Instruction parsers
│   ├── events/         # Event definitions (if applicable)
│   └── types/          # Custom type definitions
```

## Usage Example

After generating a decoder, use it in your Rust code:

```rust
// returns data params
let only_data = RaydiumAmmV4Decoder.decode_compiled_instruction_params(&txn_ix);
// returns data params and structured accounts u8 (u8 is the index in the actual account keys of ys transaction)
let data_acc = RaydiumAmmV4Decoder.decode_compiled_instruction(&txn_ix);


//pick accounts from the txn
let mut all_accounts: Vec<Vec<u8>> = txn_message.account_keys.clone();
// rw - https://github.com/rpcpool/yellowstone-vixen/blob/5a6788ee5da28d640a002b8d63b93297907ae20d/crates/core/src/instruction.rs#L191
all_accounts.extend(txn_meta.loaded_writable_addresses.iter().cloned());
all_accounts.extend(txn_meta.loaded_readonly_addresses.iter().cloned());
let account_pubkeys: Vec<Pubkey> = all_accounts
        .iter()
        .map(|acc| Pubkey::new_from_array(acc.as_slice().try_into().unwrap()))
        .collect();
// returns data params and structured accounts but instead of u8, the accounts will be pubkey
let data_pks = RaydiumAmmV4Decoder.decode_compiled_instruction_with_accounts(&txn_ix, &account_pubkeys);
```
OR
```rust
/// .
/// RAYDIUM
///
///
pub fn raydium_amm_v4_handler(txn_ix: CompiledInstruction)  {
    if let Some(decoded_ix) = RaydiumAmmV4Decoder.decode_compiled_instruction(&txn_ix) {
        match decoded_ix.compiled_accounts {
            raydium_amm_v4_decoder::instructions::CompiledInstructionAccounts::SwapBaseIn(
                swap_base_in_compiled_instruction_accounts,
            ) => {
             
                    // pool: swap_base_in_compiled_instruction_accounts.amm,
                    // vault_a: swap_base_in_compiled_instruction_accounts.pool_coin_token_account,
                    // vault_b: swap_base_in_compiled_instruction_accounts.pool_pc_token_account,
                    // user_vault_a: swap_base_in_compiled_instruction_accounts.uer_source_token_account,
                    // user_vault_b: swap_base_in_compiled_instruction_accounts.uer_destination_token_account,
                    
            }
            raydium_amm_v4_decoder::instructions::CompiledInstructionAccounts::SwapBaseOut(
                swap_base_out_compiled_instruction_accounts,
            ) => {

                    // pool: swap_base_out_compiled_instruction_accounts.amm,
                    // vault_a: swap_base_out_compiled_instruction_accounts.pool_coin_token_account,
                    // vault_b: swap_base_out_compiled_instruction_accounts.pool_pc_token_account,
                    // user_vault_a: swap_base_out_compiled_instruction_accounts.uer_source_token_account,
                    // user_vault_b: swap_base_out_compiled_instruction_accounts.uer_destination_token_account,
            }
            _ => {}
        }
    }
    None
}
```
OR
```rust
#[derive(Debug, Clone)]
pub struct SwapEventRouter {
    pub amm: String,
    pub input_mint: String,
    pub output_mint: String,
    pub input_amount: u64,
    pub output_amount: u64,
}
/// .
/// JUPITER
///
///
pub fn jupiter_v6_handler(inner_ixs: Vec<InnerInstruction>) -> Option<Vec<SwapEventRouter>> {
    let mut swap_events: Vec<SwapEventRouter> = Vec::new();
    for ix in inner_ixs {
        if let Some(decoded_event) =
            JupiterV6Decoder.decode_compiled_instruction_params(&CompiledInstruction {
                program_id_index: ix.program_id_index,
                accounts: ix.accounts,
                data: ix.data,
            })
        {
            match decoded_event {
                jupiter_v6_decoder::instructions::JupiterV6Instruction::SwapEvent(swap_event) => {
                    swap_events.push(SwapEventRouter{
                        amm: swap_event.amm.to_string(),
                        input_mint: swap_event.input_mint.to_string(),
                        output_mint: swap_event.output_mint.to_string(),
                        input_amount: swap_event.input_amount,
                        output_amount: swap_event.output_amount,
                    });
                },
                _=> {}
            }
        }
    }

     Some(swap_events)
}
```
## Interactive Mode

Run without arguments for interactive mode:

```bash
cargo run
```

The CLI will prompt you to:
1. Choose between `parse` or `scaffold`
2. Select IDL source (file or program address)
3. Configure output options
4. Select decoders and data sources (for scaffold)

## Development

### Project Structure
- `src/main.rs` - CLI entry point and interactive mode
- `src/commands.rs` - Command definitions and argument parsing
- `src/handlers/` - Core parsing and generation logic
- `templates/` - Code generation templates
- `decoders/` - Pre-built decoder implementations

### Building
```bash
cargo build
```


# Copied idl from the site? Do the following:
1. replace `:` -> `":"`
2. in VsCode, `ctrl+f` in the file and add the following regex: `(\s[a-zA-Z]+":)`
3. replace: `"$1`
4. Some fields may have values as follows:
    ```
    active: !0,    //  true --> replace
    disabled: !1   //  false --> replace
    ```

# Common Errors:
## Eq, Hash
1. add impl instead of them:
Example:
    ```
    use std::hash::Hasher;
    use std::hash::Hash;
    impl Hash for StubOracleSet {
        fn hash<H: Hasher>(&self, state: &mut H) {
            self.price.to_bits().hash(state);
        }
    }

    impl Eq for StubOracleSet {}
    ```        
2. its its optional: `self.max_staleness_slots.hash(state);`

## Transactions not being parsed
- Well, the problem is here: https://github.com/PUMPFUN-max/carbon-cli-ys/blob/main/src/accounts.rs#L52
- So, you gotta go through each instruction in decoder>src>instructions and add the actual discrimator.
- Feel free to open a PR if you think you got a better solution thats supported by global idl's.

## Import Errors:
Just import the packages/files... 
