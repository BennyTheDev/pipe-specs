# PIPE Introduction

PIPE is a Bitcoin-native token protocol inspired by Casey Rodarmor's RUNES and Ordinal's BRC-20 ideas.

Since RUNES does not allow for fair mints (an important aspect of BRC-20's) and potentially renders all of its tokens securities (due to centralized distribution), PIPE tries to combine the strengths of both ideas.

Like BRC-20, PIPE consists of 3 "functions": Deploy, Mint, Transfer. Deploy signals that a new token has been deployed, while Mint allows to mint from this token, based on the deployment's rules (supply, limits). Eventually, Transfer is being used to send tokens to selected recipients.

This document describes how each of the functions should be reflected within Bitcoin transactions and how indexers/wallets must treat those.
Further upgrades are not ruled out and may progressively be enhanced within this document.

## PLEASE NOTE

There is no indexing or wallet for PIPE yet. You can download the following simple javascript that showcases how each deploy, mint and transfer function works: https://trac.network/pipe.zip

Just download, unzip, edit and run in your browser. Initial instructions included in the html file.

## General Rules

- A PIPE TX must consist of at least 2 outputs:
- - The beneficiary receiver
  - An OP_RETURN output, followed by the 'P' and function identifier (D, M, T) and its data
- Tickers are base26 encoded and must have an arbitrary unsigned integer with a max. value of 999999 attached (ID)
- - base26 value encoding: A=1, AA=27, Z=26, BA=53
  - ID = 0 semantically represents "No ID" (clients can leave out '0' and just render the ticker)
  - The combination ticker:ID must be unique.
- All integers passed after OP_RETURN must be unsigned
- All unsigned integers from 0 - 16 must be encoded as OP_0/OP_FALSE, OP_1/OP_TRUE to OP_16
- Only 1 OP_RETURN is allowed per TX
- Burning
- - If no change address is specified as quadruple with the Transfer function, remaining tokens of a transfer will be burned.
  - Tokens associated with UTXOs in inputs will be burned if a function of a TX is rejected.
  - Therefore, clients have to apply careful validations of all rules before pushing a transaction.
- Indexers/wallets must detect reorgs and re-index from the first reorg'ed block - 7.
- Indexing starts with block 809608 (included)

## Deploy Rules

On a high level, the output of a Deploy function is structured as follows:

```
OP_RETURN
P
D
[BASE26 ENCODED TICKER]
[ID]
[OUTPUT]
[DECIMALS]
[MAX]
[LIMIT]
```

Looking closer the values as follows:

"P": shortcut for PIPE, signalling this is a PIPE function.

"D": shortcut for Deploy, signalling the following data is to define a new token.

"[BASE26 ENCODED TICKER]": human readable ticker name, encoded as described in General Rules.

"[ID]": the arbitray ID for the ticker from 0 - 999999 as unsigned integer. "0" semantically signals that there is no ID assigned (note that ticker:ID must be unique).

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary

"[DECIMALS]": the decimals for the token from 0 - 8 as unsigned integer.

"[MAX]": the max. amount of tokens as hex encoded string, ever for this token as in supply.

"[LIMIT]": the max. amount of tokens as hex encoded string that may be minted per tx.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules. This allows for applications like marketplaces sending royalties to the benificiary on trading fees.

Clients must skip UTXOs being used for inputs that already contain tokens.

[MAX] and [LIMIT] values must be a hex encoded string, representing a human readable number. Leading zeros are not allowed. Trailing zeros in decimals are not allowed. One decimal point can be used or omitted. No other characters are allowed.

Transactions with decimal points beyond [DECIMALS] are rejected. Max. number is '18446744073709551615'.

Examples:

```
2100 => ok
2100.5 => ok, if decimal length <= [DECIMALS]
 2100 => not ok
2100.50 => not ok
2,100 => not ok
18446744073709551616 => not ok (note exceeding the max number)
``` 

Indexers must transform the given [MAX] and [LIMIT] internally into bigints based on [DECIMALS] and perform calculations on those to maintain precision. No calculations or rounding on the original human readable format allowed.

NOTE: until block 809999, indexers need to detect if [MAX] and [LIMIT] are hex values. If not, then the raw values must be used (e.g. '1000' = 1000), else use the hex encoded string.

From block 810000 only hex encoded strings must be accepted and raw values lead to invalid token transactions.

## Optional: PIPE ART Rules

PIPE Art is an optional extension for PIPE that allows to store file data, file references (urls) and collection information alongside regular token deployments.

By tuning supply, limit and decimals, PIPE Art can help to achieve non-fungibility for deployed tokens. Deployed PIPE Art tokens behave in the exact same way for mints and transfers like regular tokens. In fact, there is no difference and they can be treated like non-PIPE Art associated tokens.

Unlike token information in outputs, PIPE Art data is stored as signed witness data (P2TR tapscript) using taproot-tweaked public keys. The original untweaked public key represents the collection address for PIPE Art. For each collection item, it can be specified what its number and the overall max. number are. Additionally, PIPE Art allows for traits to help describing the stored item.

Both file data and traits can be alternatively specified as references, pointing to offchain resources. Optionally, a receiver can be specified to mint directly with the deployment, bypassing the mint function requirement. This is however limited to a one-time-mint based on [LIMIT] of the deployment.

PIPE Art is added in the same TX as the main token deployment.

The following structure outlines the witness data script for PIPE Art:

```
[PUBKEY]
OP_CHECKSIG
OP_0
OP_IF
P
A
(I|R)
[MIMETYPE]
[INLINE DATA | REFERENCE STRING]
N
[NUMBER]
[MAX-NUMBER]
B
[BENEFICIARY-OUTPUT]
(empty | T | TR)
[TRAIT-CONTENT]
01
OP_ENDIF
```

"[PUBKEY]": The collection address. To be able to add more items to a collection, the deployments should occur using the same private key.

"[MIMETYPE]": The content type for inline data as hex encoded string. Must be skipped by indexers if it's a reference string instead.

"[INLINE DATA | REFERENCE STRING]": If "I" is specified, then bytes must be successively pushed. If "R" is specified, then the next push must be a hex encoded string, containing a file reference. 

"[NUMBER]": An unsigned integer representing the number of the deployed item in the collection (not to bed mixed up [TICKER] and [ID], those still need to be pushed as described)

"[MAX-NUMBER]": Unsigned integer representing the max. number currently in the collection. The max. number can be increased by the collection creator but never decrease.

"[BENEFICIARY-OUTPUT]": Unsigned integer starting from 0. Allows a shortcut to mint one-time directly with the deployment based on the [LIMIT] amount. 0 means disabled. Indexers have to substract [BENEFICIARY-OUTPUT] by 1 to assign the desired output properly.

"[TRAIT-CONTENT]": Must be either tuples of key/values with hex encoded strings (each key and value pushed separately), a reference to custom traits offchain or left out entirely (including its labels T or TR). 

Recommendations for creating non-fungible tokens

|| ERC-721 | ERC-1155 |
| -------------| ------------- | ------------- |
| Decimals | 0 | 0 |
| Supply | 1 | 1 - 999999 |
| Limit | 1 | 1

## Mint Rules

Mint structure:

```
OP_RETURN
P
M
[BASE26 ENCODED TICKER]
[ID]
[OUTPUT]
[MINT AMOUNT]
```

Values:

"P": shortcut for PIPE, signalling this is a PIPE function.

"M": shortcut for Mint, signalling the following data is to mint tokens.

"[BASE26 ENCODED TICKER]": human readable ticker name, encoded as described in General Rules.

"[ID]": the arbitray ID for the ticker from 0 - 999999 as unsigned integer. "0" semantically signals that there is no ID assigned (note that ticker:ID must be unique).

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary.

"[MINT AMOUNT]": The amount to mint as hex encoded string, between 0 and [LIMIT] (inclusive), given with the deploy function.

If the mint amount does not exceed the limit and supply that is left from the deployment, the amount of tokens must be credited to the beneficiary as assigned in [OUTPUT].

Remaining supply must be credited to the beneficiary, as long as the limit isn't exceeded. Indexers/wallets have to associate the amount of credited tokens to the resulting UTXO of the output linked with <OUTPUT> and store in its index.

Clients must skip UTXOs being used for inputs that already contain tokens.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules.

Deploy and Mint can happen in the same block but any Mint will be rejected if its transaction index is < the deploy transaction index.

[MINT AMOUNT] value must be a hex encoded string, representing a human readable number. Leading zeros are not allowed. Trailing zeros in decimals are not allowed. One decimal point can be used or omitted. No other characters are allowed.

Transactions with decimal points beyond [DECIMALS] (see Deploy Rules) are rejected. Max. number is '18446744073709551615'. 

Examples:

```
2100 => ok
2100.5 => ok, if decimal length <= [DECIMALS]
 2100 => not ok
2100.50 => not ok
2,100 => not ok
18446744073709551616 => not ok (note exceeding the max number)
``` 

Indexers must transform the given [MINT AMOUNT] internally into bigint based on [DECIMALS] (see Deploy Rules) and perform calculations on those to maintain precision. No calculations or rounding on the original human readable format allowed.

NOTE: until block 809999, indexers need to detect if [MINT AMOUNT] is a hex value. If not, then the raw value must be used (e.g. '1000' = 1000), else use the hex encoded string.

From block 810000 only hex encoded strings must be accepted and raw values lead to invalid token transactions.

## Transfer Rules

Transfer may contain a number quadruple of pushes after 'T'. Each quadruple may address different tickers, IDs, outputs and amounts.
This allows to specify change addresses (and limited multi-send) within a single transaction. There is no limit on the amount of quadruples other than the max. allowed script size for the output.

Transfer structure:

```
OP_RETURN
P
T
...begin quadruple
[BASE26 ENCODED TICKER]
[ID]
[OUTPUT]
[TRANSFER AMOUNT]
...end quadruple
...next quadruple...
```

"P": shortcut for PIPE, signalling this is a PIPE function.

"T": shortcut for Transfer, signalling the following data is to send tokens.

Quadruple:

"[BASE26 ENCODED TICKER]": human readable ticker name, encoded as described in General Rules.

"[ID]": the arbitray ID for the ticker from 0 - 999999 as unsigned integer. "0" semantically signals that there is no ID assigned (note that ticker:ID must be unique).

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary.

"[TRANSFER AMOUNT]": The amount to transfer as hex encoded string.

The transfer amount must be deducted from the amount of tokens associated with the UTXOs of the inputs. Remaining tokens _should_ be sent to a change address using another quadruple, unless they should be burned. If there are UTXOs with enough token balances, the transfer amount must be credited to the beneficiary as assigned in each quadruple's [OUTPUT]. 

It is important to note that one [OUTPUT] can only be used once to prevent multiple token types being associated with a single utxo. The transaction will be rejected if the combined amount of UTXO balances are insufficient _or_ there are duplicate [OUTPUT] associations. It's not recommended to use different token types (ticker:id) in one OP_RETURN. Due to the size limitations, there might not be enough space left for wallets to create enough quadruples to assign change for each type.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules.

Deploy, Mint and Transfer can happen in the same block but any Transfer will be rejected if its transaction index is < the deploy transaction index or < the Mint transaction that _would_ credit for the amount to transfer.

[TRANSFER AMOUNT] value must be a hex encoded string, representing a human readable number. Leading zeros are not allowed. Trailing zeros in decimals are not allowed. One decimal point can be used or omitted. No other characters are allowed.

Transactions with decimal points beyond [DECIMALS] (see Deploy Rules) are rejected. Max. number is '18446744073709551615'.

Examples:

```
2100 => ok
2100.5 => ok, if decimal length <= [DECIMALS]
 2100 => not ok
2100.50 => not ok
2,100 => not ok
``` 

Indexers must transform the given [TRANSFER AMOUNT] internally into bigint based on [DECIMALS] (see Deploy Rules) and perform calculations on those to maintain precision. No calculations or rounding on the original human readable format allowed.

The operation must be atomic: If one quadruple fails, all fail. No token balance-changing operations will be applied in this case.

NOTE: until block 809999, indexers need to detect if [TRANSFER AMOUNT] is a hex value. If not, then the raw value must be used (e.g. '1000' = 1000), else use the hex encoded string.

From block 810000 only hex encoded strings must be accepted and raw values lead to invalid token transactions.
