# Use Concordium Testnet Node Service

Fortunately, Concordium provides a testnet node meaning you don't need to run your own node in your system for lots of things. It can allow you to do almost everything that you can do with your own local node other than the functions below.&#x20;

```
shutdown = false
peer_connect = false
peer_disconnect = false
get_banned_peers = false
ban_peer = false
unban_peer = false
dump_start = false
dump_stop = false
get_peers_info = false
get_node_info = false
```

You need to add _--grpc-port 1000 --grpc-ip testnet.node.concordium.com_ at the end of your command like below.

```
concordium-client my_command --grpc-port 1000 --grpc-ip testnet.node.concordium.com
```
