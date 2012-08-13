## The Why of Signing
Signing content is meant as a way to verify content without having to read it on the original feed. While an author's feed exists at a canonical (and therefore trusted) location, the system is meant to publish and distribute many copies of individual messages and especially in the case of mentions and reposts, there needs to exist a simple and reliable way to verify that the message came from the claimed origin.


## Author Meta Data
The feed's `author` meta data contains a dictionary of public keys for that user's feed indexed by unique key names. Each key contains the RSA public certificate in PEM format under `key` and optionally a key expiration date under `expired`.

The meta data itself is of course signed as well.

```javascript
{
  _sig: {
    name: 'key2',
    sig: 'xfZ4DmrcLbz8qPJoTwYg/wIqggIKBBtzqnaiUu1Wess82wKdge+UsQEqU1hY2/0OrzgtUnzgn8nSWWPJtd6qtKbOTkPQqYDf2uVk6WTHYjwpysHmMj8fzrMkpE0ZPkPD8N7kEn1Rmt85CeXMDjYDN14H3Ep4iRNc7qxeNSR7xH8='
  },
  id: 'joe@drooginstustries.com',
  name: 'Joe Smith',
  profile_image_uri: 'http://droogindustries.com/joe.jpg',
  status_uri: 'http://droogindustries.com/bob/status',
  feed_uri: 'http:/droogindustries.com/bob/status/feed',
  public_keys: {
    key2: {
      key: '-----BEGIN RSA PRIVATE KEY----- ...'
    },
    key1: {
      key: '-----BEGIN RSA PRIVATE KEY----- ...',
      expired: '2012-07-30T11:31:00Z'
    }
  },
  previous_ids: [ 'joseph@smith.com' ],
  publishers: ['http://publish.droogindustries.com/publish/RPqdvmtOHmEPbJ+kX'],
  aggregators: ['http://aggro.droogindustries.com/aggro/AgMBAAEwDQYJKoZIhvcNA']
}
```
Key names are arbitrary and used only to identify which public key goes with which signature.

## The signature
The signature block for any JSON block is always at key `_sig` and contains a `name` key to match to the public key from `author.public_keys.{keyname}` and a base-64 RSA-SHA256 signature:
```javascript
{
  _sig: {
    name: 'key2',
    sig: 'xfZ4DmrcLbz8qPJoTwYg/wIqggIKBBtzqnaiUu1Wess82wKdge+UsQEqU1hY2/0OrzgtUnzgn8nSWWPJtd6qtKbOTkPQqYDf2uVk6WTHYjwpysHmMj8fzrMkpE0ZPkPD8N7kEn1Rmt85CeXMDjYDN14H3Ep4iRNc7qxeNSR7xH8='
  },
  ...
}
```

## Key generation

The private key is an RSA key in PEM format and can be generated with:
```
openssl genrsa -out private.pem 1024
```
Whether it is generated in plain or encrypted form does not affect Happenstance, since only the public key is used by the spec. Content creation/signing workflow is left up to the implementer.

The corresponding public key is the PEM formatted certificate generated with:
```
openssl rsa -in private.pem -out public.pem -outform PEM -pubout
```

## Creating a signature
In order to sign a JSON block, we first must convert it into its canonical string format. The procedure to generate this form is as follows:
* Iterate over all keys in alphanumeric order
  * if a key contains a hash, descend into that block and iterate over its keys in alphanumeric order
  * if a key contains an array, iterate over the array in its current order
* skip any key starting with an _underscore
* append {keyname}:{value}, for each key
  * do not quote the value

**An Example**

```javascript```
{
  id: 1234,
  name: 'john doe',
  _ignore: 'not getting signed'
  address: {
    street: '123 Main St',
    _apt: '2a',
    zip: 90210
  }
}
```
The above block would be converted into the canonical form of:
```
address:street:123 Main St,zip:90210,,id:1234,name:john doe,
```
The canonical string form is then converted into a signed SHA256 hash in base64 form like this:
```
openssl dgst -sha256 -sign private.pem msg.txt | base64 -w 0 > msg.sig.b64
```
where `msg.txt` contains the canonical string form. `base64 -w 0` is used to generate a single line signature without linefeeds.

Given the signature and the key name that corresponds to the key, the `_sig` block is created and inserted into the block.

## Verifying a signature
To verify, once again the canonical string form is generated, which can then be verified via:
```
base64 -d msg.sig.b64 msg.sig;
openssl dgst -sha256 -verify public.pem -signature msg.sig msg.txt;
```
where
* `msg.sig.b64` is the `_sig.key`
* `msg.txt` is the canonical string form
* `public.pem` comes from `author.public_keys.{_sig.name}`

## Key expiration
Because access to a key may be lost for many reasons, it is possible to expire a public key with an `expired` date. This date is advisory to anyone verifying the signature that if the message was created after that date, the message is invalid.

Should the key in question actually be compromised, it is generally better to just remove the key entirely, since a bad actor with the compromised key could issue messages in the past. In this case affected messages need to be re-signed, so that someone trying to verify it, will fail and try to fetch the message again from its canonical uri (which has to be in a subpath of the feed) to ensure it exists and is valid at the source.

Expiration is better suited for lost key, lost passphrase or access to a third party that is in charge of that key being lost.