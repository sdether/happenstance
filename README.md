happenstance
============

Happenstance is a specification and sample implementation for a _federated, decentralized status network_. The features of this network are strong and discoverable identity, status update feeds, update publish & subscribe, mentions, re-post and private messaging.

The network is defined as a number of interoperating systems, that together form the network, but which can be implemented independently. All communication in the network is done over HTTP using JSON as the message payload.

## The update workflow

Before going into details of how the network is constructed, here's a 10k ft. view of the messaging workflow for status updates.

* Sending user creates new feed entry in [Status Feed](#status-feed) system
* [Status Feed](#status-feed) pushes entry to [PubSub](#pubsub)
* [PubSub](#pubsub) pushes entry to recipients
  * entry is pushed to all its subscribers, generally [Aggregation](#aggregation) servers
  * Identity Entities in entries ( _mentions_ ) are resolved via the appropriate [Identity](#identity) server to locate the resposible Aggregation](#aggregation) servers, and pushed to those servers
* [Aggregation](#aggregation) combines and de-dupes entries and presents them as a status timeline to the recipient user

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
  _sig: '366e58644911fea255c44e7ab6468c6a5ec6b4d9d700a5ed3a810f56527b127e',
  id: 'joe@droogindustries.com',
  name: 'Joe Smith',
  status_uri: 'http://droogindustries.com/status',
  feed_uri: 'http:/droogindustries.com/status/feed.hsf'
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
  author: {metadata},
  entries: [{entry}, ...]
  subscriptions: [
    {
      id: 'bob@drooginstustries.com',
      name: 'Bob Jones',
      profile_image.uri: 'http://droogindustries.com/bob.jpg',
      status_uri: 'http://droogindustries.com/bob/status',
      feed_uri: 'http://droogindustries.com/bob/status/feed',
    },
    ...
  ]
}
```
Subscriptions is an optional way to display what users the feed owner follows, but is not required. Generally subscriptions are handled by aggregation servers and do not have to be public.

For paging, two additional keys that may exist in the feed are `previous_uri` and `next_uri`. These keys allow callers to page through the feed without needing to know the implementation of the feed's API. 

#### Author Data

The minimum requirement for the meta data is defined below. Additional information can be added as desired by implemented, although it is generally good form to use a single key and stuff the desired content into a sub-object.

```javascript
{
  _sig: '80669bd0d0bc39a062f87107de126293d85347775152328bf464908430712856',
  id: 'joe@drooginstustries.com',
  name: 'Joe Smith',
  profile_image_uri: 'http://droogindustries.com/joe.jpg',
  status_uri: 'http://droogindustries.com/bob/status',
  feed_uri: 'http:/droogindustries.com/bob/status/feed',
  public_key: 'MIIBvTCCASYCCQD55fNzc0WF7TANBgkqhkiG9w0BAQUFADAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwHhcNMTAwNTI4MDIwODUxWhcNMjAwNTI1MDIwODUxWjAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBANGEYXtfgDRlWUSDn3haY4NVVQiKI9CzThoua9+DxJuiseyzmBBe7Roh1RPqdvmtOHmEPbJ+kXZYhbozzPRbFGHCJyBfCLzQfVos9/qUQ88u83b0SFA2MGmQWQAlRtLy66EkR4rDRwTj2DzR4EEXgEKpIvo8VBs/3+sHLF3ESgAhAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAEZ6mXFFq3AzfaqWHmCy1ARjlauYAa8ZmUFnLm0emg9dkVBJ63aEqARhtok6bDQDzSJxiLpCEF6G4b/Nv/M/MLyhP+OoOTmETMegAVQMq71choVJyOFE5BtQa6M/lCHEOya5QUfoRF2HF9EjRF44K3OK+u3ivTSj3zwjtpudY5Xo='
  previous_keys: [
    'MIIBvTCCASYCCQD55fNzc0WF7TANBgkqhkiG9w0BAQUFADAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwHhcNMTAwNTI4MDIwODUxWhcNMjAwNTI1MDIwODUxWjAjMQswCQYDVQQGEwJKUDEUMBIGA1UEChMLMDAtVEVTVC1SU0EwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBANGEYXtfgDRlWUSDn3haY4NVVQiKI9CzThoua9+DxJuiseyzmBBe7Roh1RPqdvmtOHmEPbJ+kXZYhbozzPRbFGHCJyBfCLzQfVos9/qUQ88u83b0SFA2MGmQWQAlRtLy66EkR4rDRwTj2DzR4EEXgEKpIvo8VBs/3+sHLF3ESgAhAgMBAAEwDQYJKoZIhvcNAQEFBQADgYEAEZ6mXFFq3AzfaqWHmCy1ARjlauYAa8ZmUFnLm0emg9dkVBJ63aEqARhtok6bDQDzSJxiLpCEF6G4b/Nv/M/MLyhP+OoOTmETMegAVQMq71choVJyOFE5BtQa6M/lCHEOya5QUfoRF2HF9EjRF44K3OK+u3ivTSj3zwjtpudY5Xo='
    ],
  previous_ids: [ 'joseph@smith.com' ],
  publishers: ['http://publish.droogindustries.com/publish/RPqdvmtOHmEPbJ+kX'],
  aggregators: ['http://aggro.droogindustries.com/aggro/AgMBAAEwDQYJKoZIhvcNA']
}
```
#### Entry

Entries are the status updates. They should be considered write-only, since once published, copies will exist in many downstream data-stores. There is a mechanism for advisory updates and deletes using `updates_id` and `deletes_id` keys, but it is up to the downstream implementer to determine whether those entries are respected, used to create revision history or applied permanently. The minimum set (except for `deletes_id` entries) is:
```javascript
{
  _sig: 'aadefbd0d0bc39a062f87107de126293d85347775152328bf464908430712789',
  id: '4AQlP4lP0xGaDAMF6CwzAQ'
  href: 'http://droogindustries.com/joe/status/feed/4AQlP4lP0xGaDAMF6CwzAQ',
  created_at: '2012-07-30T11:31:00Z',
  author: {
    id: 'joe@drooginstustries.com',
    name: 'Joe Smith',
    profile_image.uri: 'http://droogindustries.com/joe.jpg',
    status_uri: 'http://droogindustries.com/joe/status',
    feed_uri: 'http://droogindustries.com/joe/status/feed',
  },
  text: 'Hey #{bob}, current status #{beach} #{vacation}',
  entities: {
    beach: 'http://droogindustries.com/images/beach.jpg',
    bob: 'bob@foo.com',
    vacation '#vacation'
  }
}
```

For re-posting someone else's status update, a `repost_href` key containing the original posts href can be included. In addition, it is recommended to also include the full entry under the `_repost` key (omitting it from signature requirements). The `text` of the status update can be used to add additional commentary.

Additional optional fields are specified in the separate message spec.

While there is no way to enforce content size, since the content will replicated across the network and is meant as a status network not forum, the `text` field is assumed to be limited to 1000 characters and implementers are free to drop of truncate (which invalidates the signature) long messages.

#### Authoring

While the feed is designed that it can be represented by static files, the authoring tool responsible for creating the status content bears the additional responsibility of pushing messages to the **PubSub** servers listed in `author.publishers` as described in the [PubSub](#pubsub) section.

### Aggregation

Aggregation server collect status updates for users to create their timeline. In general, aggregation servers will receive events from **PubSub** servers, but the same expected API is also used to push _mentions_ and private messages to a user.

The only required API for an aggregation server is rooted at any uri provided in the `author.aggregators` list in a user's feed. This endpoint is expected to accept POST messages with a body of `{ entries: [{entry}, ...] }`.

Since this endpoint will likely be the target of spammers, implementers are advised to include some defensive mechanisms, such as whitelisting senders to subscribed feeds only or checking the entries for a valid source feed and signature, etc.

### PubSub

The PubSub server is responsible to publishing status updates to all subscribers. It defines two REST APIs, one for creating and managing publication resources and one for creating and managing subscription resources per publication resource. Only the subscription API is required to be implemented, i.e. a pubsub server could be an integral part of the feed content management system and have no publicly exposed API for creating publications.

The subscription REST API is always rooted at the uris provided by the `author.publishers` list in the feed and. The methods supported by the endpoint are:

**POST:**
```javascript
{
  callback_uri: 'http://aggro.droogindustries.com/aggro/AgMBAAEwDQYJKoZIhvcNA',
  auth_header: 'X-Auth: wNTI1MDIwODUxWjAjMQs'
}
```
_Response: **201 Created**_
```javascript
{
  subscription_uri: 'http://publish.droogindustries.com/publish/RPqdvmtOHmEPbJ+kX/subscription/wDQYJKoZIhvcNAQEBBQADgY0AM',
}
```
The response also contains the `subscription_uri` in the location header. The optional `auth_header` in the POST will be included as a header in each publish callback, as will the location of the subscription (so that upon receiving a publication POST, the receiver can identify the caller in case the information was lost.

To delete an existing subscription the **PubSub** server must implement the DELETE call:

**DELETE:{subscrition.uri}**

PubSub may also implement an API for setting up subscription endpoints that the feed owner uses to set up the subscription. It's a REST API similar to subscriber API for creating and management a publication resource, but adds a POST endpoint at the subscription location that accepts messages with a body of `{ entries: [{entry}, ...] }`.

The responsibility of PubSub is twofold:
* deliver the entries posted to it to all its subscribers, and
* deliver the mentions enumerated in the entities object.

The former does not have to be the exact entries document posted to it. It is up the implementation to determine whether it accumulates entries and/or combines them with other entries destinated for the same callback.uri/auth.header combination.

The latter involves looking up the feed for a name entity via the nameserver and then post the mentioned entries to the aggregators listed in `author.aggregators`.

### Discovery

Discovery has no formal definition. It is comprised with organic discovery via reposts and browsing the subscriptions of the user you are subscribed out and discovery from various search indicies.

Given that anyone can get real-time feeds delivery and the feeds of anyone mentioned in an entry or can be discovered via the name-servers it is relatively simple to get access to the communications of the ecosystem for permanent or ephemeral, complete or partial indexing as a service.

Aggregators will likely want to integrate search greater than the data that passes through them, so they are the most likely customers of such services, but such integration will be custom and would not benefit for a formal definition at this stage of the ecosystem.

## Content Signing

To generate or check a SHA256 signature hash for a json body in happenstance, the key value pairs are appended in alphanumeric order. Compound values are appended as if they were simply keys in the parent object. Keys that start with underscore are ommitted from signature calculation. This is done so that downstream consumers of messages can attach meta-data without invalidating the signature and clearly separates non-trusted keys from trusted keys. The resulting string is hashed and then signed with the private key.
