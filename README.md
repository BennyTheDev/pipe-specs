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
- Tickers are base26 encoded and must have an arbitrary integer with a max. value of 9999 attached (ID)
- - base26 value encoding: A=1, AA=27, Z=26, BA=53
  - ID = 0 semantically represents "No ID" (clients can leave out '0' and just render the ticker)
  - The combination ticker:ID must be unique.
- All integers passed after OP_RETURN must be unsigned
- All unisgned integers from 0 - 16 must be encoded as OP_0/OP_FALSE, OP_1/OP_TRUE to OP_16
- Only 1 OP_RETURN is allowed per TX
- No self-referenced burn events allowed. To burn tokens, they have to be assigned to a beneficiary receiver that represents a burner address. Exception is if no change address is specified as quadruple with the Transfer function. In this case, remaining tokens of a transfer will be indeed burned.
- - This means a PIPE TX is invalid if it points to the output containing OP_RETURN or any scriptPubyKey entry _not_ containing a beneficiary receiver (address/pubkey)
- Any rule-breaking TX will lead to skipping the TX entirely for inclusion of a token related result in an index
- Indexers/wallets must detect reorgs and re-index from the first reorg'ed block - 7.

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

"[BASE26 ENCODED TICKER]": human readable ticker name, decoded as described in General Rules.

"[ID]": the arbitray ID for the ticker from 0 - 9999 as unsigned integer. "0" semantically signals that there is no ID assigned (note that ticker:ID must be unique).

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary

"[DECIMALS]": the decimals for the token from 0 - 8 as unsigned integer.

"[MAX]": the max. amount of tokens as hex encoded string, ever for this token as in supply.

"[LIMIT]": the max. amount of tokens as hex encoded string that may be minted per tx.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules. This allows for applications like marketplaces sending royalties to the benificiary on trading fees.

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

"[BASE26 ENCODED TICKER]": human readable ticker name, decoded as described in General Rules.

"[ID]": the arbitray ID for the ticker from 0 - 9999 as unsigned integer. "0" semantically signals that there is no ID assigned (note that ticker:ID must be unique).

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary.

"[MINT AMOUNT]": The amount to mint as hex encoded string, between 0 and [LIMIT] (inclusive), given with the deploy function.

If the mint amount does not exceed the limit and supply that is left from the deployment, the amount of tokens must be credited to the beneficiary as assigned in [OUTPUT].

Remaining supply must be credited to the beneficiary, as long as the limit isn't exceeded. Indexers/wallets have to associate the amount of credited tokens to the resulting UTXO of the output linked with <OUTPUT> and store in its index.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules.

Deploy and Mint can happen in the same block but any Mint will be ignored if its transaction index is < the deploy transaction index.

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

## Transfer Rules

Transfer may contain a number quadruple of pushes after 'T'. Each quadruple may address different tickers, IDs, outputs and amounts.
This allows multi-sends within a single transaction. There is no limit on the amount of quadruples other than the max. allowed script size for the output.

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

"[BASE26 ENCODED TICKER]": human readable ticker name, decoded as described in General Rules.

"[ID]": the arbitray ID for the ticker from 0 - 9999 as unsigned integer. "0" semantically signals that there is no ID assigned (note that ticker:ID must be unique).

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary.

"[TRANSFER AMOUNT]": The amount to transfer as hex encoded string.

The transfer amount must be deducted from the amount of tokens associated with the UTXOs of the inputs. Remaining tokens _should_ be sent to a change address using another quadruple, unless they should be burned. If there are UTXOs with enough token balances, the transfer amount must be credited to the beneficiary as assigned in each quadruple's [OUTPUT]. The transaction will be rejected if the combined amount of UTXO balances are insufficient.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules.

Deploy, Mint and Transfer can happen in the same block but any Transfer will be ignored if its transaction index is < the deploy transaction index or < the Mint transaction that _would_ credit for the amount to transfer.

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

The operation must be atomic: If one quadruple fails, all fail.
