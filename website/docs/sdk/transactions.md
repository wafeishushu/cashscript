---
title: Sending Transactions
---

When calling a contract function on a Contract object, an incomplete Transaction object is returned. This transaction can be completed by providing a number of outputs using the [`to()`][to()] or [`withOpReturn()`][withOpReturn()] functions. Other chained functions are included to set other transaction parameters.

Most of the available transaction options are only useful in very specific use cases, but the functions [`to()`][to()], [`withOpReturn()`][withOpReturn()] and [`send()`][send()] are commonly used. [`withHardcodedFee()`][withHardcodedFee()] is also commonly used with covenant contracts.

## Transaction options

### to()
```ts
transaction.to(to: string, amount: bigint, token?: TokenDetails): this
transaction.to(outputs: Array<Recipient>): this
```

The `to()` function allows you to add outputs to the transaction. Either a single pair `to/amount` pair can be provided, or a list of them. This function can be called any number of times, and the provided outputs will be added to the list of earlier added outputs. Tokens can be sent by providing a `TokenDetails` object as the third parameter, or including it in your array of outputs with the `.token` property.

```ts
interface Recipient {
  to: string;
  amount: bigint;
  token?: TokenDetails;
}

interface TokenDetails {
  amount: bigint;
  category: string;
  nft?: {
    capability: 'none' | 'mutable' | 'minting';
    commitment: string;
  };
}
```

:::note
The CashScript SDK supports automatic UTXO selection for BCH and fungible CashTokens. However, if you want to send Non-Fungible CashTokens, you will need to do manual UTXO selection using `from()`.
:::

#### Example
```ts
.to('bitcoincash:qrhea03074073ff3zv9whh0nggxc7k03ssh8jv9mkx', 500000n)
```

### withOpReturn()
```ts
transaction.withOpReturn(chunks: string[]): this
```

The `withOpReturn()` function allows you to add `OP_RETURN` outputs to the transaction. The `chunks` parameter can include regular UTF-8 encoded strings, or hex strings prefixed with `0x`. This function can be called any number of times, and the provided outputs will be added to the list of earlier added outputs.

#### Example
```ts
.withOpReturn(['0x6d02', 'Hello World!'])
```

### from()
```ts
transaction.from(inputs: Utxo[]): this
```

The `from()` function allows you to provide a hardcoded list of contract UTXOs to be used in the transaction. This overrides the regular UTXO selection performed by the CashScript SDK, so **no further selection will be performed** on the provided UTXOs. This function can be called any number of times, and the provided UTXOs will be added to the list of earlier added UTXOs.

:::tip
The built-in UTXO selection is generally sufficient. But there are specific use cases for which it makes sense to use a custom selection algorithm.
:::

#### Example
```ts
.from(await instance.getUtxos())
```

### fromP2PKH()
```ts
fromP2PKH(input: Utxo, template: SignatureTemplate): this;
fromP2PKH(inputs: Utxo[], template: SignatureTemplate): this;
```

The `fromP2PKH()` function allows you to provide a list of P2PKH UTXOs to be used in the transaction. The passed `SignatureTemplate` is used to sign these UTXOs. This function can be called any number of times, and the provided UTXOs will be added to the list of earlier added UTXOs.

:::note
If you are using meep to debug a `fromP2PKH()` transaction, meep will always use the first input for the debugging. So if you want to debug the smart contract bytecode, make sure that the first input is not a P2PKH input.
:::

#### Example
```ts
import { bobAddress, bobPrivateKey } from 'somewhere';
import { ElectrumNetworkProvider, SignatureTemplate } from 'cashscript';

const provider = new ElectrumNetworkProvider();
const bobUtxos = await provider.getUtxos(bobAddress);

.fromP2PKH(bobUtxos, new SignatureTemplate(bobPrivateKey))
```


### withFeePerByte()
```ts
transaction.withFeePerByte(feePerByte: number): this
```

The `withFeePerByte()` function allows you to specify the fee per per bytes for the transaction. By default the fee per bytes is set to 1.0 satoshis, which is nearly always enough to be included in the next block. So it's generally not necessary to change this.

#### Example
```ts
.withFeePerByte(2.3)
```

### withHardcodedFee()
```ts
transaction.withHardcodedFee(hardcodedFee: bigint): this
```

The `withHardcodedFee()` function allows you to specify a hardcoded fee to the transaction. By default the transaction fee is automatically calculated by the CashScript SDK, but there are certain use cases where the smart contract relies on a hardcoded fee.

:::tip
If you're not building a covenant contract, you probably do not need a hardcoded transaction fee.
:::

#### Example
```ts
.withHardcodedFee(1000n)
```

### withMinChange()
```ts
transaction.withMinChange(minChange: bigint): this
```

The `withMinChange()` function allows you to set a threshold for including a change output. Any remaining amount under this threshold will be added to the transaction fee instead.

:::tip
This is generally only useful in specific covenant use cases.
:::

#### Example
```ts
.withMinChange(1000n)
```

### withoutChange()
```ts
transaction.withoutChange(): this
```

The `withoutChange()` function allows you to disable the change output. The remaining amount will be added to the transaction fee instead. This is equivalent to `withMinChange(Number.MAX_VALUE)`.

:::caution
Be sure to check that the remaining amount (sum of inputs - sum of outputs) is not too high. The difference will be added to the transaction fee and cannot be reclaimed.
:::

#### Example
```ts
.withoutChange()
```

### withoutTokenChange()
```ts
transaction.withoutTokenChange(): this
```

The `withoutTokenChange()` function allows you to disable the change output for tokens.

:::caution
Be sure to check that the remaining amount (sum of inputs - sum of outputs) is not too high. The difference will be burned and cannot be reclaimed.
:::

### withAge()
```ts
transaction.withAge(age: number): this
```

The `withAge()` function allows you to specify the minimum age of the transaction inputs. This is necessary if you want to to use the `tx.age` CashScript functionality, and the `age` parameter passed into this function will be the value of `tx.age` inside the smart contract. For more information, refer to [BIP68][bip68].

#### Example
```ts
.withAge(10)
```

### withTime()
```ts
transaction.withTime(time: number): this
```

The `withTime()` function allows you to specify the minimum block number that the transaction can be included in. The `time` parameter will be the value of `tx.time` inside the smart contract.

:::tip
By default, the transaction's `time` variable is set to the most recent block number, which is the most common use case. So you should only override this in specific use cases.
:::

#### Example
```ts
.withTime(700000)
```

## Transaction building
### send()
```ts
async transaction.send(): Promise<TransactionDetails>
```

After completing a transaction, the `send()` function can be used to send the transaction to the BCH network. An incomplete transaction cannot be sent.

```ts
interface TransactionDetails {
  inputs: Uint8Array[];
  locktime: number;
  outputs: Uint8Array[];
  version: number;
  txid: string;
  hex: string;
}
```

:::tip
If the transaction fails, a meep command is automatically returned. This command can be used to debug the transaction using the [meep debugger][meep]
:::

#### Example
```ts
import { alice } from './somewhere';

const txDetails = await instance.functions
  .transfer(new SignatureTemplate(alice))
  .withOpReturn(['0x6d02', 'Hello World!'])
  .to('bitcoincash:qrhea03074073ff3zv9whh0nggxc7k03ssh8jv9mkx', 200000n)
  .to('bitcoincash:qqeht8vnwag20yv8dvtcrd4ujx09fwxwsqqqw93w88', 100000n)
  .withHardcodedFee(1000n)
  .send()
```

### build()
```ts
async transaction.build(): Promise<string>
```

After completing a transaction, the `build()` function can be used to build the entire transaction and return the signed transaction hex string. This can then be imported into other libraries or applications as necessary.

#### Example
```ts
const txHex = await instance.functions
  .transfer(new SignatureTemplate(alice))
  .to('bitcoincash:qrhea03074073ff3zv9whh0nggxc7k03ssh8jv9mkx', 500000n)
  .withAge(10)
  .withFeePerByte(10)
  .build()
```

### meep()
```ts
async transaction.meep(): Promise<string>
```

After completing a transaction, the `meep()` function can be used to return the required debugging command for the [meep debugger][meep]. This command string can then be used to debug the transaction.

#### Example
```ts
const meepStr = await instance.functions
  .transfer(new SignatureTemplate(alice))
  .to('bitcoincash:qrhea03074073ff3zv9whh0nggxc7k03ssh8jv9mkx', 500000n)
  .withTime(700000)
  .meep()
```

:::note
Meep does not work very well with contracts that use modern CashScript / BCH features, like native introspection, P2SH32 or CashTokens.
:::


## Transaction errors
Transactions can fail for a number of reasons. Most of these are related to the execution of the smart contract (e.g. wrong parameters or a bug in the contract code). But errors can also occur because of other reasons (e.g. a fee that's too low or the same transaction already exists in the mempool). To facilitate error handling in your applications, the CashScript SDK provides an enum of different *reasons* for a failure.

This `Reason` enum only includes errors that are related to smart contract execution, so other reasons have to be caught separately. Besides the `Reason` enum, there are also several error classes that can be caught and acted on:

* **`FailedRequireError`**, signifies a failed require statement. This includes the following reasons:
  * `Reason.EVAL_FALSE`
  * `Reason.VERIFY`
  * `Reason.EQUALVERIFY`
  * `Reason.CHECKMULTISIGVERIFY`
  * `Reason.CHECKSIGVERIFY`
  * `Reason.CHECKDATASIGVERIFY`
  * `Reason.NUMEQUALVERIFY`
* **`FailedTimeCheckError`**, signifies a failed time check using `tx.time` or `tx.age`. This includes the following reasons:
  * `Reason.NEGATIVE_LOCKTIME`
  * `Reason.UNSATISFIED_LOCKTIME`
* **`FailedSigCHeckError`**, signifies a failed signature check. This includes the following reasons:
  * `Reason.SIG_COUNT`
  * `Reason.PUBKEY_COUNT`
  * `Reason.SIG_HASHTYPE`
  * `Reason.SIG_DER`
  * `Reason.SIG_HIGH_S`
  * `Reason.SIG_NULLFAIL`
  * `Reason.SIG_BADLENGTH`
  * `Reason.SIG_NONSCHNORR`
* **`FailedTransactionError`**, signifies a general fallback error. This includes all remaining reasons listed in the `Reason` enum as well as any other reasons unrelated to the smart contract execution.

```ts
enum Reason {
  EVAL_FALSE = 'Script evaluated without error but finished with a false/empty top stack element',
  VERIFY = 'Script failed an OP_VERIFY operation',
  EQUALVERIFY = 'Script failed an OP_EQUALVERIFY operation',
  CHECKMULTISIGVERIFY = 'Script failed an OP_CHECKMULTISIGVERIFY operation',
  CHECKSIGVERIFY = 'Script failed an OP_CHECKSIGVERIFY operation',
  CHECKDATASIGVERIFY = 'Script failed an OP_CHECKDATASIGVERIFY operation',
  NUMEQUALVERIFY = 'Script failed an OP_NUMEQUALVERIFY operation',
  SCRIPT_SIZE = 'Script is too big',
  PUSH_SIZE = 'Push value size limit exceeded',
  OP_COUNT = 'Operation limit exceeded',
  STACK_SIZE = 'Stack size limit exceeded',
  SIG_COUNT = 'Signature count negative or greater than pubkey count',
  PUBKEY_COUNT = 'Pubkey count negative or limit exceeded',
  INVALID_OPERAND_SIZE = 'Invalid operand size',
  INVALID_NUMBER_RANGE = 'Given operand is not a number within the valid range',
  IMPOSSIBLE_ENCODING = 'The requested encoding is impossible to satisfy',
  INVALID_SPLIT_RANGE = 'Invalid OP_SPLIT range',
  INVALID_BIT_COUNT = 'Invalid number of bit set in OP_CHECKMULTISIG',
  BAD_OPCODE = 'Opcode missing or not understood',
  DISABLED_OPCODE = 'Attempted to use a disabled opcode',
  INVALID_STACK_OPERATION = 'Operation not valid with the current stack size',
  INVALID_ALTSTACK_OPERATION = 'Operation not valid with the current altstack size',
  OP_RETURN = 'OP_RETURN was encountered',
  UNBALANCED_CONDITIONAL = 'Invalid OP_IF construction',
  DIV_BY_ZERO = 'Division by zero error',
  MOD_BY_ZERO = 'Modulo by zero error',
  INVALID_BITFIELD_SIZE = 'Bitfield of unexpected size error',
  INVALID_BIT_RANGE = 'Bitfield\'s bit out of the expected range',
  NEGATIVE_LOCKTIME = 'Negative locktime',
  UNSATISFIED_LOCKTIME = 'Locktime requirement not satisfied',
  SIG_HASHTYPE = 'Signature hash type missing or not understood',
  SIG_DER = 'Non-canonical DER signature',
  MINIMALDATA = 'Data push larger than necessary',
  SIG_PUSHONLY = 'Only push operators allowed in signature scripts',
  SIG_HIGH_S = 'Non-canonical signature: S value is unnecessarily high',
  MINIMALIF = 'OP_IF/NOTIF argument must be minimal',
  SIG_NULLFAIL = 'Signature must be zero for failed CHECK(MULTI)SIG operation',
  SIG_BADLENGTH = 'Signature cannot be 65 bytes in CHECKMULTISIG',
  SIG_NONSCHNORR = 'Only Schnorr signatures allowed in this operation',
  DISCOURAGE_UPGRADABLE_NOPS = 'NOPx reserved for soft-fork upgrades',
  PUBKEYTYPE = 'Public key is neither compressed or uncompressed',
  CLEANSTACK = 'Script did not clean its stack',
  NONCOMPRESSED_PUBKEY = 'Using non-compressed public key',
  ILLEGAL_FORKID = 'Illegal use of SIGHASH_FORKID',
  MUST_USE_FORKID = 'Signature must use SIGHASH_FORKID',
  UNKNOWN = 'unknown error',
}
```

[fetch-api]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
[meep]: https://github.com/gcash/meep
[bip68]: https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki

[to()]: /docs/sdk/transactions#to
[withOpReturn()]: /docs/sdk/transactions#withopreturn
[from()]: /docs/sdk/transactions#from
[withFeePerByte()]: /docs/sdk/transactions#withfeeperbyte
[withHardcodedFee()]: /docs/sdk/transactions#withhardcodedfee
[withMinChange()]: /docs/sdk/transactions#withminchange
[withAge()]: /docs/sdk/transactions#withage
[withTime()]: /docs/sdk/transactions#withtime

[send()]: /docs/sdk/transactions#send
[build()]: /docs/sdk/transactions#build
[meep()]: /docs/sdk/transactions#meep
