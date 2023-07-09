# envelope ![tests](https://github.com/ssc-hermes/envelope/actions/workflows/nodejs.yml/badge.svg)

Envelopes that have been authorized by the recipient. This hides the sender's identity, while the recipient is still visible. This is meant to hide the *metadata* of who is talking to whom via private message.

Our server needs to remember the latest sequence number it has seen. This works for a single server situation, but if there are mutliple intermediate nodes, then it would be possible to reuse the certificates by passing the same sequence number to multiple intermediate nodes.

Is that an attack vector? The recipient of the message can verify who wrote the content, so it would not be possible to spoof the content of the message. The recipient can check if the sender is someone they expect.

We want to give out our send certificates *privately*, without revealing them publicly. Although, we (the recipient) are still able to verify the identity of the message sender.

If we assume we are doing this with internet infrastructure (a server), then in the initial meeting the server would be able to see who we are giving out certificates to because the recipient must be visible. But in subsequent communication, the server would not know who we are talking to, they would just know that we are communicating with someone that we have given a certificate to.

You could also give out the certificates via other means, like on your website, or via signal message, in which case the server would not know who it is from. Meaning, the server cannot even assume that a message is from your 'friend circle' in the application. It can only see that you got a message at a particular time, we can't infer anything about who it is from.

## keystore
The envelopes and encrypted messages created by this library pair with a [keystore](https://github.com/fission-codes/keystore-idb) instance on your device.

We create a symmetric key and encrypt it to various "exchange" keys. The exchange keys are non-extractable key pairs that can only be used on the device where they were created.

That way the documents created by this library can be freely distributed without leaking any keys.

## an envelope

Just a document signed by the recipient, like
```js
// envelope
{
    seq: 0,
    expiration: 456,
    recipient: 'my-username',
    signature: '123abc',
    author: 'did:key:abc'
}
```

## a message in an envelope

```js
// the message
{ envelope, content: 'encrypted text' }
// sender ID is in the content, so that it is only readable by the recipient
```

## API

### types

#### Envelope
```ts
import { SignedRequest } from '@ssc-hermes/message'

export type Envelope = SignedRequest<{
    seq:number,
    expiration:number,
    recipient:string,  // the recipient's username
}>
```

#### EncryptedContent
When you encrypt a string, we create a record of keys. The `key` object is a map from device name to a symmetric key that has been encrypted to the device. We do it this way because each device has its own keypair. We use the symmetric key to encrypt the content.

```ts
interface EncryptedContent {
    key:Record<string, string>,  // { deviceName: 'encrypted-key' }
    content:string  // encrypted text
}
```

### create
Create an envelope.

```ts
import { SignedRequest } from '@ssc-hermes/message'

export type Envelope = SignedRequest<{
    seq:number,
    expiration:number,
    recipient:string,  // the recipient's username
}>

export async function create (crypto:Crypto.Implementation, {
    username,
    seq,
    // expire 1 year from now by default
    expiration = new Date().setFullYear(new Date().getFullYear() + 1)
}:{ username:string, seq:number, expiration:number }) => Promise<Envelope>
```

### wrapMessage
Take an envelope and some content. Encrypt the content, then put it in the envelope. This returns an array of
```js
[{ envelope, message: encryptedMessage }, { ...senderKeys }]
```

We return the sender keys as a seperate object because we *do not* want the sender's device names to be in the message that gets sent, because that would leak information about who the sender is.

You could save a map of the message hash to the returned key object. That way you can save the map to some storage, then you can look up the key by the hash of the message object.

**question**
You would need to do multi-device sync with the map of msg -> keys though. How do you sync multiple devices without leaking information about who wrote the message?

The keys for the recipient are in the encrypted message content. The premise is that the recipient will have a [keystore](https://github.com/fission-codes/keystore-idb) instance locally, and will use their local private key to decrypt the symmetric key in the message, then use that symmetric key to decrypt this message content.

We use a two step decryption like that because we are supporting multi-device use. If someone has a handful of devices, then the symmetric key will be encrypted to each device separately, so there are no shared keys.

```ts
import { Identity } from '@ssc-hermes/identity'
import { SignedRequest } from '@ssc-hermes/message'

export type Envelope = SignedRequest<{
    seq:number,
    expiration:number,
    recipient:string,  // the recipient's username
}>

export async function wrapMessage (
    me:Identity,
    recipient:Identity,  // because we need to encrypt the message to the recipient
    envelope:Envelope,
    content:Content
):Promise<[{
    envelope:Envelope,
    message:EncryptedContent
}, Keys]>
```

----------------------

Thanks to @Dominic for sketching this idea.
