Usage
=====

Node app uses

```javascript
var client = require('haredis').createClient(['localhost:6379', 'localhost:6380', 'localhost:6381'], options);
// Functions same way as standard redis client
```

Writes go to master, reads can be load-balanced to slaves.

orientate()
===========

Called:
- with `createClient()`
- when a node comes online
- when a node throws an error (goes offline)

Can't be called while in progress. Pauses commands (use queue).

```
call INFO on all nodes in `nodeList`.
  validate `server_info.role`
    if 'slave', take note of its master_host/master_port

once all nodes come back with responses (or errors),
  failover if
    count of master != 1
    master host/port on a slave doesn't point to central master
  set master and trigger ready
    drain queue
  use slaves in command rotation once ready
  subscribe to `haredis:gossip` channel on all nodes
```

Failover
========

```
(queue commands while failing over)
generate random id
iterate online nodes and:
  (MULTI)
    `SETNX haredis:failover (id)`
    `GET haredis:failover`
  if it's our id
    `EXPIRE haredis:failover (ttl)`
    continue.
  else, roll back other nodes (`DEL` keys) and wait (randomized timeout)

once all are iterated,
  if no master
    elect node with lowest master_last_io_seconds_ago as master
  if multiple masters
    elect node with most `connected_slaves` or `uptime_in_seconds`
  then
    `SLAVEOF NO ONE` on that node
    `SLAVEOF host port` on other nodes
    `PUBLISH haredis:gossip master:host:port` on master
    apps switch to that master
```