# skipjs

A util for signing and sending bundles through the Skip relayer.

# Usage

skipjs exposes a single default export, `SkipBundleClient`.
`SkipBundleClient` has two functions:
`signBundle` and `sendBundle`.

```
type SignedBundle = {
  transactions: Array<string>
  pubKey: string
  signature: string
}

async signBundle(transactions: Array<TxRaw>, signer: OfflineSigner, signerAddress: string): SignedBundle

async sendBundle(bundle: SignedBundle, desiredHeight: number, sync?: boolean): Promise<object>
```

## signBundle
`signBundle` is used to sign a bundle of transactions. It must be provided with an array of `TxRaw` (from cosmjs-types), an `OfflineSigner` (from cosmjs), and a `signerAddress` (the address of the searcher).
It returns a `SignedBundle`, which can be passed to `sendBundle` to send the bundle to the relayer.

## sendBundle

`sendBundle` sends a `SignedBundle` to the relayer.
`desiredHeight` is the desired height for the bundle to be included on-chain. Submitted bundles will only be considered in the auction for this height. Passing `0` as `desiredHeight` will cause the relayer to consider the bundle for the immediate next height.
`sync` is an optional parameter that defaults to `false`. If set to `true`, the promise will not resolve until the bundle has been simulated. If set to `false`, the promise resolves on bundle submission, prior to its simulation.

## Example usage:
Import `SkipBundleClient`, as well as a way to get an `OfflineSigner`, and other utils for the chain you're using. For this example, we'll use Juno.
```
import { SkipBundleClient } from '@skip-mev/skipjs'
import { getOfflineSignerProto } from 'cosmjs-utils';
import { juno, getSigningCosmosClient, cosmos } from 'juno-network';
import { chains } from 'chain-registry'
```
Create your transactions:
```
const {
    multiSend,
    send
} = cosmos.bank.v1beta1.MessageComposer.fromPartial;

var msg = send({
    amount: [
    {
        denom: 'ujuno',
        amount: '10'
    }
    ],
    toAddress: toAddress,
    fromAddress: fromAddress
});

const fee = {
    amount: [
    {
        denom: 'ujuno',
        amount: '864'
    }
    ],
    gas: '86364'
};
```

Sign your transactions:
```
const signer = await getOfflineSignerProto({
    mnemonic,
    chain: chains.find(({ chain_name }) => chain_name === 'juno')
});

const client = await getSigningCosmosClient({
  rpcEndpoint,
  signer
});

const { accountNumber, sequence } = await client.getSequence(address);
const txRaw = await client.sign(address, [msg], fee, '', {
    accountNumber: accountNumber,
    sequence: sequence,
    chainId: 'juno-skip-1'
});
```
Create your SkipBundleClient:
```
const skipBundleClient = new SkipBundleClient(RELAYER_RPC_ENDPOINT)
```

The RPC endpoint is an `ip:port` string that depends on the chain you're using. Skip relayer endpoints for each chain can be found [here](https://www.notion.so/skip-protocol/Skip-Configurations-By-Chain-a6076cfa743f4ab38194096403e62f3c).

Sign and send your bundle:
```
const signedBundle = await skipBundleClient.signBundle([txRaw], signer, BUNDLE_SENDER_ADDRESS)
const sendBundle = await skipBundleClient.sendBundle(signedBundle, DESIRED_HEIGHT_FOR_BUNDLE)
```

`BUNDLE_SENDER_ADDRESS` should be a string, the on-chain address of the account of the bundle signer.
`DESIRED_HEIGHT_FOR_BUNDLE` should be a number.
