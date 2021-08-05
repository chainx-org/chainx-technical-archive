# Musig2: Why Libp2p

## Content

1. [Musig2](#Musig2)
2. [Libp2p](#Libp2p)
   1. [PeerID](#PeerID)
   2. [Swarm](#Swarm)
   3. [Transport](#Transport)
   4. [NetworkBehaviour](#NetworkBehaviour)
   5. [Pub/Sub](#Pub/Sub)
3. [Why Libp2p](#Why%20Libp2p)
4. [Summary](#Summary)

## Musig2

MuSig has an innovative feature of key aggregation. The signature under the Musig scheme is a conventional `Schnorr` signature, which can be processed by the bitcoin network once Taproot is activated. When used to create a multi-signature wallet, compared with the traditional method of n-of-n multi-signature using CHECKMULTISIG opcodes, MuSig reduces the transaction cost and increases privacy, while the traditional method requires n public keys and n ECDSA signatures on the blockchain. However, due to the need for multiple rounds of communication between signers, it may be difficult to deploy MuSig in practice. More precisely, three rounds of communication are required to create signatures under the Musig scheme, and each round of communication consists of messages passed back and forth. To improve MuSig and make the signing process easier, you can use MuSig2, a new scheme designed by Blockstream. In MuSig2's scheme, it is optimized from the original three-round interaction (MuSig) to only two rounds of interaction, and provides the same function and security as MuSig, with only a small amount of additional computation, which is an excellent scheme for aggregating signatures.

![musig2-overview](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1628063191945-musig2-overview -en.png)

Musig2 transactions on the main Bitcoin network will be similar to ordinary transactions, so the operation on the chain is similar to sending ordinary transactions, so the process of aggregating signatures needs to be completed under the chain. Therefore, to implement the offline aggregate signature of Musig2 in ChainX's Taproot BTC cross-chain upgrade scheme, you can consider using `libp2p` to implement a decentralized aggregate signature.

## Libp2p

Libp2p is a modular system composed of protocols, specifications, and codebase, which can be used to develop P2P networks and solve the problems of node discovery and data flow between nodes in P2P networks.

1. `Discover Peers`：It can discover other nodes in the P2P network and maintain the online status of nodes, and adjust the network connection according to the status of nodes, to build a stable network topology.
2. `Data Transmission`：Data transmission is responsible for the flow of data between nodes. To support the interconnection of heterogeneous network devices, one of the core requirements of libP2P is that the transport layer is unknown. Libp2p supports different transport layer protocols, such as TCP, UDP, and QUIC. After the connection is established at the transport layer, the privacy security of network data needs to be considered. Libp2p encrypts the transmission channel, and the nodes communicate through the encrypted channel. To efficiently transfer data, libP2P supports multiplexing of connections to support multiple concurrent streams between nodes.

### PeerId

In libp2p, the identity of each node is a key pair. Each node is identified by `PeerID`, a string of characters that hash the node's public key and are base58 encoded. With the node ID, it will allow anyone to authenticate the node identity by retrieving the public key, which enables secure communication between nodes and effectively solves the problem of man in middle attack.

Since each transport protocol has its address format, libp2p uses the coding scheme of `multiaddr` to unify the address format of different protocols.For instance, `/ip4/192.168.1.11/tcp/9999/p2p/12D3KooWGHPFrxdTaDiu5LTFRvnvk5jMQfzTsmop4A5qaiNhsbMH`，We can know that the IP address is 192.168.1.11, uses the TCP protocol, uses the port 9999, is a P2P node, and the node ID is 12D3KooWGHPFrxdTaDiu5LTFRvnvk5jMQfzTsmop4A5qaiNhsbMH.

### Swarm

The `Swarm` module of libp2p is responsible for combining multiple transport layers (such as TCP and UDP) into one interface, thus allowing applications to connect to remote nodes without specifying which transport layer to use. Swarm is mainly responsible for managing the creation, maintenance, and destruction of connections between nodes. It includes protocol multiplexing, stream multiplexing, NAT traversal, and connection relay, and carries on the multiplex transmission at the same time. The network interface is the interface provided by libp2p, and Swarm is the concrete implementation of the Network interface. Libp2p maintains the state of existing connections and supported transport layer protocols in Swarm. Swarm supports multiple transport protocols by:

1. When you create a new node, the transport layer protocols supported by the node are added to the structure of the transport of Swarm through AddTransport.
2. When the node turns on Listen, the Swarm module will call TransportForListening to obtain the transport layer protocol corresponding to the listening address, and call the Listen function of the corresponding transport layer protocol.
3. When the node actively dials other nodes, the Swarm module calls TransportForDialing to obtain the transport protocol used by the dialing address and calls the Dial function of the corresponding transport layer protocol.

### Transport

`Transport` defines the method (protocol) of data transmission and ensures privacy in the process of data transmission. TcpTransport is the transport layer implementation module of TCP, which combines Upgrader module. Upgrader is responsible for upgrading an original TCP connection to support encryption and multi-stream multiplexing. Secio and tls are two libraries that implement the SecureTransport interface, and the Libp2p library uses the secio encryption library by default.

Libp2p applications typically open much separate traffic flows between nodes, and may open multiple concurrent streams at the same time as a remote node. Multistream multiplexing allows you to send and receive data for the entire life cycle by establishing a connection with the remote node, and you only need to deal with the NAT operation once, because all streams from the same remote node share the same underlying transport connection. When you configure Libp2p, the stream multiplexing module is enabled, and Swarm uses them when dialing remote nodes and listening for connections. If the remote node supports the same multiplexing implementation, Swarm will select and use it when establishing the connection, and if the dial-up Swarm has established a connection to the remote node, the new stream will automatically multiplex on the existing connection.

### NetworkBehaviour

`Transport` defines how bytes are sent on the network, while `NetworkBehaviour` defines which bytes are sent on the network. More specifically, similar to the network tool ping, the local node sends the ping to the remote node and expects to receive the pong accordingly. Ping NetworkBehaviour doesn't care how ping or pong messages are sent over the network, whether they are encrypted by TCP, noise, or plain text, it only cares about what is sent on the network.

### Pub/Sub

Sending messages to other nodes in the core of P2P systems. Publishing and subscribing is a very useful mechanism to send messages of interest to specific nodes. Libp2p defines publish and subscribe interfaces to send messages to all nodes that subscribe to a particular topic. Currently, there are two stable interface implementations: floodsub is very simple to use but inefficient, it uses network flooding strategy; gossipsub defines an extensible gossip protocol. Episub is also under active development and is an extension of gossipsub, which is optimized for remote source multicast and a small number of fixed sources to broadcast to a large number of clients.

![pub/sub](https://cdn.jsdelivr.net/gh/rjman-ljm/resources@master/assets/1628065139421-1628065139418.png)

## Why Libp2p

Libp2p unifies the address format of different protocols through the coding scheme of `multiaddr`. In the Swarm module, it parses the protocol according to `multiaddr` and calls the interface of the corresponding protocol to complete the specific operation, thus achieving the goal that the application layer does not need to pay attention to the transport layer protocol, and provides a simple interface to adapt to the existing or future protocols, which enables libp2p applications to run in many different network environments.

## Summary

Under the chain implementation of the Musig2 scheme, the use of libp2p can achieve the purpose of decentralization. It can be predicted that after the BTC network activates Taproot, participating in the Musig2 scheme to aggregate signatures under the chain combined with the privacy brought by the Schnorr signature on the chain can protect the privacy and security of users in the transaction process to the greatest extent.