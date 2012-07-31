happenstance
============

Happenstance is a specification and sample implementation for a _federated, decentralized status network_. The features of this network are strong and discoverable identity, status update feeds, update publish & subscribe, mentions, re-post and private messaging.

The network is defined as a number of interoperating systems, that together form the network, but which can be implemented independently. All communication in the network is done over HTTP using JSON as the message payload.

## The components

* [Identity](#identity)
* [Status Feed](#status-feed)
* [Aggregation](#aggregation)
* [PubSub](#pubsub)
* [Discovery](#discovery)

### Identity

The Identity server provides the status feed lookup for **{user}@{host}** identities. It MUST provide a single API endpoint for looking up users, although there is no prohibition against alternative lookup endpoints and methods.

**GET:/who/joe**
```javascript
{
  sig: '366e58644911fea255c44e7ab6468c6a5ec6b4d9d700a5ed3a810f56527b127e',
  id: 'joe@droogindustries.com',
  name: 'Joe Smith',
  status.uri: 'http://droogindustries.com/status',
  feed.uri: 'http:/droogindustries.com/status/feed.hsf'
}
```
The above specifies the minimum required information, although the entire feed metadata section may be served by this call.

A domain's status nameserver is expected to live at **status.{host}**, unless a DNS SRV record exists for the {host}:
```
_happenstance-ns._tcp.{host}. TTL IN SRV 5 0 {port} {status host}
```

### Status Feed

The canonical location of the status is always an HTML document including a `link` element identifying the status feed document. This abstraction serves two purposes:

* The canonical location can stay the same while the feed management provider can be changed independently
* The HTML document should serve as a human readable rendering of the feed document

The link takes the form of:
```html
<link rel="alternate" type="application/hsf+json" href="http://droogindustries.com/status/feed.hsf"/>
```

The document itself contains two sets of information, the feed meta data and a subset of the feed entries.
```javascript
{
  meta: {metadata},
  entries: [{entry}, ...]
}
```

#### Meta Data

The minimum requirement for the meta data is defined below. Additional information can be added as desired by implemented, although it is generally good form to use a single key and stuff the desired content into a sub-object.

```javascript
{
  sig: '80669bd0d0bc39a062f87107de126293d85347775152328bf464908430712856',
  id: 'joe@drooginstustries.com',
  name: 'Joe Smith',
  profile_image.uri: 'http://droogindustries.com/joe.jpg',
  status.uri: 'http://droogindustries.com/status',
  feed.uri: 'http:/droogindustries.com/status/feed',
  public.key: 'MIIBvTCCASYCCQD55fNzc0WF7TANBgkqhkiG9w0BAQUFADAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwHhcNMTAwNTI4MDIwODUxWhcNMjAwNTI1MDIwODUxWjAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBANGEYXtfgDRlWUSDn3haY4NVVQiKI9CzThoua9+DxJuiseyzmBBe7Roh1RPqdvmtOHmEPbJ+kXZYhbozzPRbFGHCJyBfCLzQfVos9/qUQ88u83b0SFA2MGmQWQAlRtLy66EkR4rDRwTj2DzR4EEXgEKpIvo8VBs/3+sHLF3ESgAhAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAEZ6mXFFq3AzfaqWHmCy1ARjlauYAa8ZmUFnLm0emg9dkVBJ63aEqARhtok6bDQDzSJxiLpCEF6G4b/Nv/M/MLyhP+OoOTmETMegAVQMq71choVJyOFE5BtQa6M/lCHEOya5QUfoRF2HF9EjRF44K3OK+u3ivTSj3zwjtpudY5Xo='
  previous.keys: [
    'MIIBvTCCASYCCQD55fNzc0WF7TANBgkqhkiG9w0BAQUFADAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwHhcNMTAwNTI4MDIwODUxWhcNMjAwNTI1MDIwODUxWjAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBANGEYXtfgDRlWUSDn3haY4NVVQiKI9CzThoua9+DxJuiseyzmBBe7Roh1RPqdvmtOHmEPbJ+kXZYhbozzPRbFGHCJyBfCLzQfVos9/qUQ88u83b0SFA2MGmQWQAlRtLy66EkR4rDRwTj2DzR4EEXgEKpIvo8VBs/3+sHLF3ESgAhAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAEZ6mXFFq3AzfaqWHmCy1ARjlauYAa8ZmUFnLm0emg9dkVBJ63aEqARhtok6bDQDzSJxiLpCEF6G4b/Nv/M/MLyhP+OoOTmETMegAVQMq71choVJyOFE5BtQa6M/lCHEOya5QUfoRF2HF9EjRF44K3OK+u3ivTSj3zwjtpudY5Xo='
    ],
  previous.ids: [ 'joseph@smith.com' ],
  publishers: ['http://publish.droogindustries.com/publish/RPqdvmtOHmEPbJ+kX']
  aggregators: ['http://aggro.droogindustries.com/aggro'
}
```
#### Entry

Entries are the status updates. They should be considered write-only, since once published, copies will exist in many downstream data-stores. There is a mechanism for advisory updates and deletes using `updates.id` and `deletes.id` keys, but it is up to the downstream implementer to determine whether those entries are respected, used to create revision history or applied permanently. The minimum set (except for `deletes.id` entries) is:
```javascript
{
  sig: 'aadefbd0d0bc39a062f87107de126293d85347775152328bf464908430712789',
  id: '4AQlP4lP0xGaDAMF6CwzAQ'
  href: 'http://droogindustries.com/status/feed/4AQlP4lP0xGaDAMF6CwzAQ',
  created_at: ''2012-07-30T11:31:00Z',
  author: {
    id: 'joe@drooginstustries.com',
    name: 'Joe Smith',
    profile_image.uri: 'http://droogindustries.com/joe.jpg',
    status.uri: 'http://droogindustries.com/status',
    feed.uri: 'http://droogindustries.com/status/feed',
  },
  text: 'Hey #{bob}, current status #{beach} #{vacation}',
  entities: {
    beach: 'http://droogindustries.com/images/beach.jpg',
    bob: 'bob@foo.com',
    vacation '#vacation'
  }
}
```

For re-posting someone else's status update, a `repost` key containing their full entry is included, while the `text` can be used to add additional commentary. Additional optional fields are specified in the separate message spec.

While there is no way to enforce content size, since the content will replicated across the network and is meant as a status network not forum, the `text` field is assumed to be limited to 1000 characters and implementers are free to drop of truncate (which invalidates the signature) long messages.

### Aggregation

### PubSub

### Discovery

### Content Signing

To generate or check a SHA256 signature hash for a json body in happenstance, the key value pairs are appended in alphanumeric order, skipping the `sig` key. Compound values are appended as if they were simply keys in the parent object. The resulting string is hashed and then signed with the private key.

Keys that start with underscore are ommitted from signature calculation. This is done so that downstream consumers of messages can attach meta-data without invalidating the signature and clearly separates non-trusted keys from trusted keys.
