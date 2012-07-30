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
  status.uri: 'http://droogindustries.com/status',
  feed.uri: 'http:/droogindustries.com/status/feed.hsf',
  public.key: 'MIIBvTCCASYCCQD55fNzc0WF7TANBgkqhkiG9w0BAQUFADAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwHhcNMTAwNTI4MDIwODUxWhcNMjAwNTI1MDIwODUxWjAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBANGEYXtfgDRlWUSDn3haY4NVVQiKI9CzThoua9+DxJuiseyzmBBe7Roh1RPqdvmtOHmEPbJ+kXZYhbozzPRbFGHCJyBfCLzQfVos9/qUQ88u83b0SFA2MGmQWQAlRtLy66EkR4rDRwTj2DzR4EEXgEKpIvo8VBs/3+sHLF3ESgAhAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAEZ6mXFFq3AzfaqWHmCy1ARjlauYAa8ZmUFnLm0emg9dkVBJ63aEqARhtok6bDQDzSJxiLpCEF6G4b/Nv/M/MLyhP+OoOTmETMegAVQMq71choVJyOFE5BtQa6M/lCHEOya5QUfoRF2HF9EjRF44K3OK+u3ivTSj3zwjtpudY5Xo='
  previous.keys: [
    'MIIBvTCCASYCCQD55fNzc0WF7TANBgkqhkiG9w0BAQUFADAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwHhcNMTAwNTI4MDIwODUxWhcNMjAwNTI1MDIwODUxWjAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBANGEYXtfgDRlWUSDn3haY4NVVQiKI9CzThoua9+DxJuiseyzmBBe7Roh1RPqdvmtOHmEPbJ+kXZYhbozzPRbFGHCJyBfCLzQfVos9/qUQ88u83b0SFA2MGmQWQAlRtLy66EkR4rDRwTj2DzR4EEXgEKpIvo8VBs/3+sHLF3ESgAhAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAEZ6mXFFq3AzfaqWHmCy1ARjlauYAa8ZmUFnLm0emg9dkVBJ63aEqARhtok6bDQDzSJxiLpCEF6G4b/Nv/M/MLyhP+OoOTmETMegAVQMq71choVJyOFE5BtQa6M/lCHEOya5QUfoRF2HF9EjRF44K3OK+u3ivTSj3zwjtpudY5Xo='
    ],
  previous.ids: [ 'joseph@smith.com' ],
  publishers: ['http://publish.droogindustries.com/publish/RPqdvmtOHmEPbJ+kX']
  aggregators: ['http://aggro.droogindustries.com/aggre'
}
```
#### Entry
```javascript
{
  
}
```

### Aggregation

### PubSub

### Discovery

### Content Signing

To generate or check a SHA256 signature hash for a json body in happenstance, the key value pairs are appended in alphanumeric order, skipping the `sig` key. Compound values are appended as if they were simply keys in the parent object. The resulting string is hashed and then signed with the private key.
