<div align="center">
  <p>
    <a href="" >
      <img src="img/nats-logo.png" width="256px"/>
    </a>
  </p>
  <!-- <p>
    <h1 align="center">
    </h1>
  </p> -->
</div>

---
>Simple guide on NATS JetStream with Leafnodes

# 🗺️ Topology diagram
![](img/nats-streams-topology-light.png)

The specific of the projects requres us to place three three-node clusters in different locations and isolate each cluster in own JetStream domain.

- `tickets-1` and `tickets-2` streams act as sources for `sales` stream
- `cinemaA-new-films` is mirrored in `cinemaA-new-films-mirror` at another facility

## 🏭 Architecture
NATS is installed on three different Kubernetes clusters via Helm charts

Kubernetes version: 1.25 \
NATS vesion: 2.9.19 \
Chart version: 0.19.16

# 🚀 NATS Cluster Configuration
## Headquarter
Headquarters NATS cluster will play the role of hub (hub and spoke model). \
`leafnodes` is enalbed here and listens on port `7422` for remote connections. \
For the purps of this guide we setup two accounts that will be shared between all NATS Clusters: \
* sys
* acc (with JetStream enabled)
```yaml
nats:
  jetstream:
    enabled: true
    domain: hub
cluster:
  enabled: true
  name: headquarter
leafnodes:
  enabled: true
  port: 7422
  no_advertise: true
auth:
  enabled: true
  basic:
    accounts:
      SYS:
        users:
          - user: sys 
            pass: sys
      ACC: 
        users:
          - user: acc
            pass: acc           # use bcrypted password instead
        jetstream: enabled
  systemAccount: SYS
```
## Cinema A
Cinama-A NATS cluster will be called `spoke-1`. \
It will initiate remote connection to the public address of `hub` and a `NodePort` opened for that purpose.
```yaml
nats:
  jetstream:
    enabled: true
    domain: cinema-a
cluster:
  enabled: true
  name: spoke-1
auth:
  enabled: true
  basic:
    accounts:
      SYS:
        users:
          - user: sys 
            pass: sys
      ACC: 
        users:
          - user: acc
            pass: acc           # use bcrypted password instead
        jetstream: enabled
leafnodes:
  remotes:
    - url: "nats://acc:acc@headquarter.nats.public.address.com:30742"
      account: ACC
    - url: "nats://sys:sys@headquarter.nats.public.address.com:30742"
      account: SYS
```
## Cinema B
Cinama-A NATS cluster will be called `spoke-2`. \
It will initiate remote connection to the public address of `hub` and a `NodePort` opened for that purpose.
```yaml
nats:
  jetstream:
    enabled: true
    domain: cinema-b
cluster:
  enabled: true
  name: spoke-2
auth:
  enabled: true
  basic:
    accounts:
      SYS:
        users:
          - user: sys 
            pass: sys
      ACC: 
        users:
          - user: acc
            pass: acc           # use bcrypted password instead
        jetstream: enabled
leafnodes:
  remotes:
    - url: "nats://acc:acc@headquarter.nats.public.address.com:30742"
      account: ACC
    - url: "nats://sys:sys@headquarter.nats.public.address.com:30742"
      account: SYS
```
# 💨 NATS Streams
Now let's create `cinama-a` stream and it's mirror on `hub` and `spoke-1`
## Mirror example
### cinema-a
`src/hub.cinema-a.json`
```json
{
    "name": "cinema-a",
    "subjects": [
      "cinema-a.new.films"
    ],
    "retention": "interest",
    "max_consumers": -1,
    "max_msgs_per_subject": -1,
    "max_msgs": -1,
    "max_bytes": -1,
    "max_age": 0,
    "max_msg_size": -1,
    "storage": "file",
    "discard": "new",
    "num_replicas": 3,
    "duplicate_window": 120000000000,
    "sealed": false,
    "deny_delete": false,
    "deny_purge": false,
    "allow_rollup_hdrs": false
}
```
### cinema-a-mirror
`src/spoke-1.cinema-a-mirror.json`
```json
{
    "name": "cinema-a",
    "subjects": [
      "cinema-a.new.films"
    ],
    "retention": "interest",
    "max_consumers": -1,
    "max_msgs_per_subject": -1,
    "max_msgs": -1,
    "max_bytes": -1,
    "max_age": 0,
    "max_msg_size": -1,
    "storage": "file",
    "discard": "new",
    "num_replicas": 3,
    "duplicate_window": 120000000000,
    "sealed": false,
    "deny_delete": false,
    "deny_purge": false,
    "allow_rollup_hdrs": false
}
```
### Apply to cluster and verify
```bash
nats --context=hub str add --config src/hub.cinema-a.json --js-domain=hub
nats --context=hub str add --config src/spoke-1.cinema-a-mirror.json --js-domain=spoke-1
```
Note that in both commands we interact with `headquarters` NATS cluster `--context=hub` and \
call the respective JetStream Domain with `--js-domain=hub` and `--js-domain=spoke-1`.


Now let's run some report commands to check the links between streams. In the last table \
we clearly notice that `cinema-a-mirror` represents a `Mirror` of `cinama-a` stream and is calling it on `$JS.hub.API`.
```bash
nats --context=hub str report --js-domain=hub

Obtaining Stream stats

╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                      Stream Report                                                       │
├────────────────────────┬─────────┬───────────┬───────────┬──────────┬─────────┬──────┬─────────┬─────────────────────────┤
│ Stream                 │ Storage │ Placement │ Consumers │ Messages │ Bytes   │ Lost │ Deleted │ Replicas                │
├────────────────────────┼─────────┼───────────┼───────────┼──────────┼─────────┼──────┼─────────┼─────────────────────────┤
│ cinema-a               │ File    │           │ 0         │ 0        │ 0 B     │ 0    │ 0       │ nats-0, nats-1*, nats-2 │
╰────────────────────────┴─────────┴───────────┴───────────┴──────────┴─────────┴──────┴─────────┴─────────────────────────╯

nats --context=hub str report --js-domain=spoke-1

Obtaining Stream stats

╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                      Stream Report                                                       │
├────────────────────────┬─────────┬───────────┬───────────┬──────────┬─────────┬──────┬─────────┬─────────────────────────┤
│ Stream                 │ Storage │ Placement │ Consumers │ Messages │ Bytes   │ Lost │ Deleted │ Replicas                │
├────────────────────────┼─────────┼───────────┼───────────┼──────────┼─────────┼──────┼─────────┼─────────────────────────┤
│ cinema-a-mirror        │ File    │           │ 0         │ 0        │ 0 B     │ 0    │ 0       │ nats-0, nats-1*, nats-2 │
╰────────────────────────┴─────────┴───────────┴───────────┴──────────┴─────────┴──────┴─────────┴─────────────────────────╯

╭───────────────────────────────────────────────────────────────────────────────────────────────╮
│                                      Replication Report                                       │
├────────────────────────┬────────┬─────────────────┬────────────────────┬────────┬─────┬───────┤
│ Stream                 │ Kind   │ API Prefix      │ Source Stream      │ Active │ Lag │ Error │
├────────────────────────┼────────┼─────────────────┼────────────────────┼────────┼─────┼───────┤
│ cinema-a-mirror        │ Mirror │ $JS.hub.API     │ cinema-a           │ 0.94s  │ 0   │       │
╰────────────────────────┴────────┴─────────────────┴────────────────────┴────────┴─────┴───────╯
```

## Source example
### tickets-1
`src/spoke-1.tickets-1.json`
```json
{
    "name": "tickets-1",
    "subjects": [
      "tickets-1.>"
    ],
    "retention": "interest",
    "max_consumers": -1,
    "max_msgs_per_subject": -1,
    "max_msgs": -1,
    "max_bytes": -1,
    "max_age": 0,
    "max_msg_size": -1,
    "storage": "file",
    "discard": "new",
    "num_replicas": 3,
    "duplicate_window": 120000000000,
    "sealed": false,
    "deny_delete": false,
    "deny_purge": false,
    "allow_rollup_hdrs": false
}
```
### tickets-2
`src/spoke-2.tickets-2.json`
```json
{
    "name": "tickets-2",
    "subjects": [
      "tickets-2.>"
    ],
    "retention": "interest",
    "max_consumers": -1,
    "max_msgs_per_subject": -1,
    "max_msgs": -1,
    "max_bytes": -1,
    "max_age": 0,
    "max_msg_size": -1,
    "storage": "file",
    "discard": "new",
    "num_replicas": 3,
    "duplicate_window": 120000000000,
    "sealed": false,
    "deny_delete": false,
    "deny_purge": false,
    "allow_rollup_hdrs": false
}
```
### sales
`src/hub.sales.json`
```json
{
    "name": "sales",
    "retention": "interest",
    "max_consumers": -1,
    "max_msgs_per_subject": 0,
    "max_msgs": -1,
    "max_bytes": -1,
    "max_age": 0,
    "max_msg_size": -1,
    "storage": "file",
    "discard": "new",
    "num_replicas": 3,
    "duplicate_window": 120000000000,
    "sources": [
      {
        "name": "tickets-1",
        "external": {
          "api": "$JS.spoke-1.API",
          "deliver": ""
        }
      },
      {
        "name": "tickets-2",
        "external": {
          "api": "$JS.spoke-2.API",
          "deliver": ""
        }
      }
    ],
    "sealed": false,
    "deny_delete": false,
    "deny_purge": false,
    "allow_rollup_hdrs": false
  }
```
### Apply to cluster and verify
```bash
nats --context=hub str add --config src/spoke-1.ticket-1.json --js-domain=spoke-1
nats --context=hub str add --config src/spoke-2.ticket-2.json --js-domain=spoke-2
nats --context=hub str add --config src/hub.sales.json --js-domain=hub
```
Again we interact with `headquarters` NATS cluster call the respective JetStream Domain with `--js-domain`.

To verify that the streams are created and linked execute `nats str report` for all `--js-domains`.
```bash
nats --context=hub str report --js-domain=spoke-1

Obtaining Stream stats

╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                      Stream Report                                                       │
├────────────────────────┬─────────┬───────────┬───────────┬──────────┬─────────┬──────┬─────────┬─────────────────────────┤
│ Stream                 │ Storage │ Placement │ Consumers │ Messages │ Bytes   │ Lost │ Deleted │ Replicas                │
├────────────────────────┼─────────┼───────────┼───────────┼──────────┼─────────┼──────┼─────────┼─────────────────────────┤
│ tickets-1              │ File    │           │ 0         │ 0        │ 0 B     │ 0    │ 0       │ nats-0, nats-1*, nats-2 │
╰────────────────────────┴─────────┴───────────┴───────────┴──────────┴─────────┴──────┴─────────┴─────────────────────────╯

nats --context=hub str report --js-domain=spoke-2

Obtaining Stream stats

╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                      Stream Report                                                       │
├────────────────────────┬─────────┬───────────┬───────────┬──────────┬─────────┬──────┬─────────┬─────────────────────────┤
│ Stream                 │ Storage │ Placement │ Consumers │ Messages │ Bytes   │ Lost │ Deleted │ Replicas                │
├────────────────────────┼─────────┼───────────┼───────────┼──────────┼─────────┼──────┼─────────┼─────────────────────────┤
│ tickets-2              │ File    │           │ 0         │ 0        │ 0 B     │ 0    │ 0       │ nats-0, nats-1*, nats-2 │
╰────────────────────────┴─────────┴───────────┴───────────┴──────────┴─────────┴──────┴─────────┴─────────────────────────╯

nats --context=hub str report --js-domain=hub

Obtaining Stream stats

╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                      Stream Report                                                       │
├────────────────────────┬─────────┬───────────┬───────────┬──────────┬─────────┬──────┬─────────┬─────────────────────────┤
│ Stream                 │ Storage │ Placement │ Consumers │ Messages │ Bytes   │ Lost │ Deleted │ Replicas                │
├────────────────────────┼─────────┼───────────┼───────────┼──────────┼─────────┼──────┼─────────┼─────────────────────────┤
│ sales                  │ File    │           │ 0         │ 0        │ 0 B     │ 0    │ 0       │ nats-0, nats-1*, nats-2 │
╰────────────────────────┴─────────┴───────────┴───────────┴──────────┴─────────┴──────┴─────────┴─────────────────────────╯

╭───────────────────────────────────────────────────────────────────────────────────────────────╮
│                                      Replication Report                                       │
├────────────────────────┬────────┬─────────────────┬────────────────────┬────────┬─────┬───────┤
│ Stream                 │ Kind   │ API Prefix      │ Source Stream      │ Active │ Lag │ Error │
├────────────────────────┼────────┼─────────────────┼────────────────────┼────────┼─────┼───────┤
│ sales                  │ Source │ $JS.spoke-1.API │ tickets-1          │ 0.94s  │ 0   │       │
│ sales                  │ Source │ $JS.spoke-2.API │ tickets-2          │ 0.44s  │ 0   │       │
╰────────────────────────┴────────┴─────────────────┴────────────────────┴────────┴─────┴───────╯
```

# 📄 Conclusion
With that example we show how to
- [x] fully connect three instances of NATS deployed on different locations
- [x] streams acting as sources for another stream or
- [x] mirror of original stream