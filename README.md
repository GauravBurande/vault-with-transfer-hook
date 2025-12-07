# Vault with Transfer Hook Enforcement — Solana + Anchor + SPL Token-2022

This repository provides a complete example of building a **token custody vault** on Solana that **enforces whitelist-based transfer rules** using the **SPL Token-2022 Transfer Hook Extension**. It consists of two Anchor programs:

1. **Vault Program**

   - Secure SPL token custody using a PDA-owned vault ATA
   - Tracks per-user deposited balances on-chain
   - Uses CPI `invoke_transfer_checked` via Token-2022

2. **Whitelist Transfer Hook Program**

   - Controls which token accounts are allowed to send or receive transfers
   - Enforced **during CPI transfer execution** using the transfer hook extension
   - Adds and removes accounts from whitelist PDAs

The repository also includes **full end-to-end LiteSVM tests** that deploy both programs together locally.

---

## Features

### Vault Program

- Program Derived Accounts:

  - `Config`: global program state containing vault + mint addresses
  - `Amount`: per-user deposited token balance

- Standard operations:

  - `initialize_vault`
  - `mint`
  - `deposit`
  - `withdraw`

- Transfer enforcement:

  - Every CPI transfer requires correctness of:

    - Transfer Hook Program Account
    - Whitelist PDA for the source token account
    - ExtraAccountMetaList PDA

### Whitelist Transfer Hook Program

- Creates Account Meta list needed by Token-2022 hook
- Ensures transfer is executed **only during actual token movement**
- Restricts transfers to **whitelisted token accounts**

---

## Architecture Overview

```
src/
├── vault/
│   ├── state/
│   │   ├── Config        # authority, vault ATA, mint
│   │   └── Amount        # user-specific deposit record
│   ├── instructions/
│   │   ├── initialize_vault
│   │   ├── mint
│   │   ├── deposit
│   │   └── withdraw
│   ├── constants.rs
│   ├── error.rs
│   └── lib.rs            # program entrypoints
└── whitelist_transfer_hook/
    ├── state/
    │   └── Whitelist     # one PDA per whitelisted token account
    ├── instructions/
    │   ├── init_extra_account_meta
    │   ├── transfer_hook
    │   └── whitelist_operations
    ├── constants.rs
    ├── error.rs
    └── lib.rs
```

---

## PDA Seeds

| Account              | Program | Seed                                | Purpose                     |
| -------------------- | ------- | ----------------------------------- | --------------------------- |
| Config               | Vault   | `["config"]`                        | Global configuration        |
| Vault ATA            | Vault   | ATA derivation (`authority=config`) | Token custody vault         |
| Amount               | Vault   | `["amount", user]`                  | User’s deposit state        |
| ExtraAccountMetaList | Hook    | `["extra-account-metas", mint]`     | Transfer hook metadata      |
| Whitelist            | Hook    | `["whitelist", token_account]`      | Per-token-account whitelist |

---

## Instruction Summary

| Program        | Instruction              | Who Signs           | Purpose                                     |
| -------------- | ------------------------ | ------------------- | ------------------------------------------- |
| Vault          | initialize_vault         | Admin               | Create config PDA + mint + vault ATA        |
| Vault          | mint                     | Admin               | Mint Token-2022 into user ATA               |
| Vault          | deposit                  | User                | Move tokens into vault (whitelist required) |
| Vault          | withdraw                 | User                | Move tokens back (whitelist required)       |
| Whitelist Hook | initialize_transfer_hook | Admin               | Create extra meta list required for hook    |
| Whitelist Hook | add_to_whitelist         | Admin               | Allow token account in transfers            |
| Whitelist Hook | remove_from_whitelist    | Admin               | Revoke whitelist and close PDA              |
| Whitelist Hook | transfer_hook            | Token-2022 CPI Only | Enforce whitelist on transfer               |

---

## Transfer Hook Enforcement

During CPI token transfer, the hook program validates:

1. Transfer context indicates an official transfer (`TransferHookAccount.transferring == true`)
2. Source token account is **whitelisted**:

```rust
require!(
    whitelist.address == source_token.key(),
    WhitelistError::NotWhitelisted
);
```

Transfers that do not satisfy these conditions fail **before tokens move**.

---

## Local LiteSVM Tests

All critical interactions validated in a sandboxed SVM environment:

- Deploy both programs
- Initialize vault & transfer hook metadata
- Mint SPL Token-2022 tokens
- Whitelist token accounts
- Deposit to vault and validate internal balance
- Withdraw back from vault with whitelist checks

Run full test suite:

```
anchor test
```

Example logs:

```
Init transaction successful
Mint transaction successful
Deposit transaction successful
Withdraw transaction successful
```

---

## Build & Deploy

Build:

```
anchor build
```

Deploy:

```
solana program deploy target/deploy/vault.so
solana program deploy target/deploy/whitelist_transfer_hook.so
```

Or:

```
anchor deploy
```

---

## Security Notes

- Only admin may manage the whitelist
- PDA bump enforcement for Config, Amount, and Whitelist accounts
- Token transfers always go through Token-2022 + CPI constraints
- Vault never signs directly; PDA seeds secure authority
- Whitelist PDA must match source token account

---

## Learning Outcomes

This repository demonstrates:

- Token-2022 Transfer Hook extension usage
- Secure token vault patterns on Solana
- Combining **multiple programs** through CPI
- Using **ExtraAccountMetaList** to extend CPI account validation
- Fast and accurate testing using **LiteSVM**
