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

## Creating a signature
In order to sign a JSON block, we first must convert it into its canonical string format. The procedure to generate this form is as follows:
* Iterate over all keys in alphanumeric order
  * if a key contains a hash, descend into that block and iterate over its keys in alphanumeric order
  * if a key contains an array, iterate over the array in its current order
* skip any key starting with an _underscore
* append {keyname}:{value} for each key
  * do not quote the value
 

```
openssl dgst -sha256 -sign private.pem msg.txt | base64 | tr -d '\n' > msg.sig.b64
```

## Verifying a signature
```
base64 -d msg.sig.b64 msg.sig
openssl dgst -sha256 -verify public.pem -signature msg.sig msg.txt
```