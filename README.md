# hive-auth-wrapper

The hive-auth-wrapper library relieves you from managing a WebSocket connection and the events it generates. It allows you to use the functionality of the HAS infrastructure in the same way as you would with a traditional API.

## Installation

```
npm install hive-auth-wrapper
```

### Usage

```
import HAS from 'hive-auth-wrapper'

const APP_META = {name:"myapp", description:"My HAS compatible application", icon:undefined}

// Create an authentication object
const auth = {
  username: "username"  // (required)
  token: undefined
  expire: undefined
  key: undefined
}

// Retrieving connection status
//
const status = HAS.status()
console.log(status)
```

#### Configuration

The HAS wrapper should work with its default configuration. However, you can change it by calling `setOptions(options)`. The `options` object has the following structure:

```
{
  host: string = undefined,
  auth_key_secret: string = undefined
}
```

* host: _(optional)_ HAS server to connect to (default to wss://hive-auth.arcange.eu)
* auth_key_secret: _(optional)_ the PKSA pre-shared encryption key to use to encrypt any auth_key passed with an auth_req payload.

NOTE: `auth_key_secret` should be defined only if you are running your own PKSA in service mode and the app sends the auth_key online with the auth_req payload!

#### Authentication

When the app performs its first authentication, it can use a `auth` object with undefined `token` and `expire` properties.
The `auth.token` and `auth.expire` will be updated if the authentication succeeds.

If the app already own an `auth` object with a `token` which has not expired, it can be reused without calling `authenticate()` again.

When authenticating a user, the app can optionaly request the PKSA to sign a challenge using one of the posting, active or memo keys.

```
if(auth.token && auth.expire > Date.now()) {
    // token exists and is still valid - no need to login again
    resolve(true)
} else {
    let challenge_data = undefined
    // optional - create a challenge to be signed with the posting key
    challenge_data = {
        key_type: "posting",
        challenge: JSON.stringify({
            login: auth.username,
            ts: Date.now(),
        })
    }

    HAS.authenticate(auth, APP_META, challenge_data, (evt) => {
        console.log(evt)    // process auth_wait
    })
    .then(res => resolve(res))  // Authentication request approved
    .catch(err => reject(err))  // Authentication request rejected or error occured
}
```

#### Broadcasting transactions

The APP can request the PKSA to sign and/or broadcast a transaction.

```
const op = [ "vote", { voter:auth.username, author:"author", permlink:"permlink", weight:10000 } ]
HAS.broadcast(auth, "posting", [op], (evt)=> {
    console.log(evt)    // process sign_wait
}) )
.then(res => resolve(res) ) // transaction approved and successfully broadcasted
.catch(err => reject(err) ) // transaction rejected or failed 
```

### Signing a challenge

Apps may want to validate an account by asking it to sign a predefined text string (challenge) with one of its keys.

```
try {
    const challenge_data = {
        key_type: "posting",
        challenge: JSON.stringify({
            login: auth.username,
            ts: Date.now(),
        })
    }
    const res = await HAS.challenge(auth, challenge_data)
    // Validate signature against account public key
    const sig = ecc.Signature.fromHex(resC.data.challenge)
    const buf = ecc.hash.sha256(challenge, null, 0)
    const verified = sig.verifyHash(buf, ecc.PublicKey.fromString(resC.data.pubkey));
    if(verified) {
        console.log("challenge succeeded")
    } else {
        console.error("challenge failed")
    }
} catch(e) {
    console.error("challenge failed")
}
```