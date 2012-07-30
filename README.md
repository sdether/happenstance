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

**GET:/who/{user}**
```javascript
{
  id: '{user}@{host}',
  name: 'Joe Smith',
  status.uri: 'http://droogindustries.com/status',
  feed.uri: 'http:/droogindustries.com/status/feed.hsf',
  sig: '4AQlP4lP0xGaDAMF6CwzAQ'
}
```

A domain's status nameserver is expected to live at **status.{host}**, unless a DNS SRV record exists for the {host}:
```
_happenstance-ns._tcp.{host}. TTL IN SRV 5 0 {port} {status host}
```

### Status Feed

### Aggregation

### PubSub

### Discovery
