# DOGMI Token Conversion

This repository contains the modified code for converting old DOGMI tokens to new DOGMI tokens on the Internet Computer (IC) platform, managed through the dogmi_sneed_sns_dapp canister.

## Overview

The DOGMI token conversion process involves transferring old DOGMI tokens to the dogmi_sneed_sns_dapp canister, verifying balances, and converting them to new DOGMI tokens.

## Process Steps

1. **Token Acquisition**: Ensure you have old DOGMI tokens for conversion.

2. **Old DOGMI Token**: The old DOGMI token canister is already deployed.
   - **Old Token Canister**: `7tx3o-zyaaa-aaaak-aes6q-cai`

3. **New DOGMI Token and Indexer Deployment**: The new DOGMI token and its indexer will be deployed during the SNS launch.

4. **dogmi_sneed_sns_dapp Configuration and Modification**:
   - **DOGMI Sneed Dapp Backend Canister**: `kw5rk-raaaa-aaaai-qpfoa-cai`
   - Removed the test folder.
   - Modified the `OldTokenType` to match the standards and methods of the old DOGMI tokens with custom transactions.
   - Modified the `NewTokenType` to ICRC1.
   - Modified some init arguments according to DOGMI conversion requirements.

5. **Old Token Transfer**: Transfer the required amount of old DOGMI tokens from your account to the dogmi_sneed_sns_dapp canister.
   - Important: Transfer more than 500,000 old tokens to ensure successful conversion, as the new token fee is 50 new tokens.

6. **New Token Allocation**: Transfer the necessary amount of new DOGMI tokens to the dogmi_sneed_sns_dapp canister. This amount should be sufficient to cover all potential conversions. This will be done through a proposal after the SNS launch.

7. **Balance Verification**: Call the check balance after providing your principal ID on the DOGMI Token Converter frontend.
   - **Frontend URL**: [DOGMI Token Converter](https://kr4x6-4yaaa-aaaai-qpfoq-cai.icp0.io/)

8. **Token Conversion**: Click the convert button to initiate the conversion process. This function will transfer new DOGMI tokens to your account based on your old token balance.

9. **Final Balance Check**: Check your balance of new DOGMI tokens.

## Major Changes in Conversion Process

Due to the old DOGMI token's lack of an index canister and incomplete support for ICRC1, we modified the IndexAccount method:
- The previous method fetched all old token transactions using an indexer and then called `IndexOldBalance` to get the required old balance.
- We wrote a custom method `fetchOldBalance`, which iteratively calls the transaction methods of the old token canister, fetching 25,000 transactions in each iteration. It calculates the old balances based on fetched transactions and returns the required old balances.

- **Updated Code**:
  In lib.mo IndexAccount function :
  ```motoko
  // Fetching old balances using the custom fetchOldBalance method
  let old_balance_result = await* fetchOldBalance(context, account);
  ```
  New Method fetchOldBalance:
  ```motoko
  // Fetch and calculate the old balances using the old transactions method
  public func fetchOldBalance(context : T.ConverterContext, account : T.Account) : async* T.IndexOldBalanceResult {
      // Extract the state from the context
      let state = context.state;

      // Extract the settings from the context
      let settings = state.persistent.settings;

      // Track the sum of OLD tokens sent from the account to the dapp
      var old_sent_acct_to_dapp_d12 : T.Balance = 0;

      // Track the sum of OLD tokens sent from the dApp to the account
      var old_sent_dapp_to_acct_d12 : T.Balance = 0;

      // Get the index of the most recent OLD token transfer transaction from the dApp to the account
      var old_latest_send_txid : ?T.TxIndex = state.ephemeral.old_latest_sent_txids.get(account.owner);

      // Track if the most recent OLD token transfer transaction from the dApp to the account is found
      var old_latest_send_found = false;

      // Assign an instance of this dApp's account to a local variable for efficiency
      let sneed_converter_dapp = context.converter;

      // Initialize variables for pagination
      var offset : Nat = 0;
      let limit : Nat = 25000; // Adjust this value based on your needs
      var totalElements : Nat = 0;
      var hasMore = true;

      while (hasMore) {
          // Fetch transactions from the old token canister
          let result = await state.persistent.old_token_canister.transaction(offset, limit);
          
          totalElements := result.totalElements;
          
          for (tx in result.content.vals()) {
              // Check if the transaction involves the given account
              if (tx.from == Principal.toText(account.owner) or tx.to == Principal.toText(account.owner)) {
                  switch (tx.kind) {
                      case (#Transfer) {
                          if (tx.from == Principal.toText(account.owner) and tx.to == Principal.toText(sneed_converter_dapp.owner)) {
                              // Transaction from account to dApp
                              old_sent_acct_to_dapp_d12 := old_sent_acct_to_dapp_d12 + tx.amount;
                          } else if (tx.from == Principal.toText(sneed_converter_dapp.owner) and tx.to == Principal.toText(account.owner)) {
                              // Transaction from dApp to account
                              old_sent_dapp_to_acct_d12 := old_sent_dapp_to_acct_d12 + tx.amount;
                          }
                      };
                      case _ {} // Ignore Mint and Burn transactions
                  };
              };

              // Check if this is the latest send transaction we're looking for
              switch (old_latest_send_txid) {
                  case (?txid) {
                      if (txid == offset + result.content.size() - 1) { // Assuming transaction index is based on position
                          old_latest_send_found := true;
                      }
                  };
                  case (null) {}
              };
          };

          // Update offset and check if there are more transactions
          offset := offset + result.content.size();
          hasMore := offset < totalElements;
      };

      // Calculate the OLD token balance
      var old_balance_d12 = 0;
      var old_balance_underflow_d12 = 0;
      if (old_sent_acct_to_dapp_d12 >= old_sent_dapp_to_acct_d12) {
          old_balance_d12 := old_sent_acct_to_dapp_d12 - old_sent_dapp_to_acct_d12;
      } else {
          old_balance_underflow_d12 := old_sent_dapp_to_acct_d12 - old_sent_acct_to_dapp_d12;
      };

      // Check if the account is considered a "Burner"
      let is_burner = old_sent_acct_to_dapp_d12 >= settings.old_burner_min_amount_d12; 

      // Return the result of the indexing operation
      return {
          old_balance_d12 = old_balance_d12;
          old_balance_underflow_d12 = old_balance_underflow_d12;
          old_sent_acct_to_dapp_d12 = old_sent_acct_to_dapp_d12;
          old_sent_dapp_to_acct_d12 = old_sent_dapp_to_acct_d12;
          is_burner = is_burner;
          old_latest_send_found = old_latest_send_found;
          old_latest_send_txid = old_latest_send_txid;
      };
  };
  ```

## OldToken Type Definition

Here is the `OldToken` type definition used in this project:

```motoko
type OldToken = actor {
    icrc1_metadata : query () -> async [(Text, Value)];
    icrc1_name : query () -> async Text;
    icrc1_symbol : query () -> async Text;
    icrc1_decimals : query () -> async Nat8;
    icrc1_fee : query () -> async Nat;
    icrc1_total_supply : query () -> async Nat;
    icrc1_minting_account : query () -> async ?Account;
    icrc1_balance_of : query (Account) -> async Nat;
    icrc1_transfer : (TransferArgs) -> async { #Ok : Nat; #Err : TransferError };
    icrc1_supported_standards : query () -> async [{ name : Text; url : Text }];
    transaction : shared query (Nat, Nat) -> async {
        content : [TransactionView];
        limit : Nat;
        offset : Nat;
        totalElements : Nat;
    };
};
```

## Repository Local Setup

To install and run the project locally:

1. Clone the repository:
   ```bash
   git clone https://github.com/sagcryptoicp/dogmi_sneed_sns_dapp.git
   ```
   
2. Navigate to the project directory:
   ```bash
   cd dogmi_sneed_sns_dapp
   ```

3. Install dependencies:
   ```bash
   npm install
   ```

4. Start the Internet Computer local replica:
   ```bash
   dfx start
   ```

5. Deploy the frontend and backend canisters:
   ```bash
   dfx deploy
   ```
