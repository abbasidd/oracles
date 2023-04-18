# oracles

## Summary

Oracle client written in bash that utilizes secure scuttlebutt for offchain message passing along with signed price data to validate identity and authenticity on-chain.

## Design Goals

Goals of this new architecture are:

1. Scalability,
2. Reduce costs by minimizing number of ethereum transactions and operations performed on-chain,
3. Increase reliability during periods of network congestion,
4. Reduce latency to react to price changes,
5. Make it easy to on-board price feeds for new collateral types, and
6. Make it easy to on-board new Oracles.

## Architecture

There are currently two main modules:

### Feed

Each Feed runs a Feed client which pulls prices redundantly with [Setzer]() and [Oracle Suite/Gofer](https://github.com/chronicleprotocol/oracle-suite), signs them with an ethereum private key, and broadcasts them as a message to the redundant p2p networks (e.g. scuttlebutt and libp2p).

### Relay

Relays monitor the broadcast messages, check for liveness, and homogenize the pricing data and signatures into a single ethereum transaction to be posted to the on-chain oracles.

### Configuration Files
The sample generic configurtion file are present in `./systemd/` folder.

## Install with Nix

### Tested on this enviroment:
  ```
  Ubuntu 22.04

  nix-build (Nix) 2.13.2
  ```

To build from inside this repo, clone and run:

```sh
nix-build
```

The setup scripts can also be used configure Omnia as a feed running with `systemd`:

```

sudo ./setup/run_feed.sh --gofer <PATH_OF_CONFIG> --omnia <PATH_OF_CONFIG> --spire <PATH_OF_CONFIG>

for example:

sudo ./setup/run_feed.sh --gofer /home/oracles/gofer.json --omnia /home/oracles/omnia_feed.json --spire /home/oracles/spire1.json

```

The setup scripts can also be used configure Omnia as a relay running with `systemd` but first make sure spire is running:
```

sudo ./setup/run_it_relay.sh --gofer <PATH_OF_CONFIG> --omnia <PATH_OF_CONFIG> --spire <PATH_OF_CONFIG>

for example:

sudo ./setup/run_it_relay.sh --gofer /home/oracles/gofer.json --omnia /home/oracles/omnia_relay.json --spire /home/oracles/spire1.json

```


If you want to run multiple omnia's feed instances then you have to change the name of the systemd files(`omnia.service`, `spire-agent.service`) which are generated by `./setup/run_feed.sh` or ```./setup/run_it_relay.sh``` and change the config files path for shell script for example: ```sudo ./setup/run_it_relay.sh --omnia /home/oracles/omnia2.json --spire /home/oracles/spire.json``` and run the shell script again.

you can run the omnia, spire and gofer with systemd by these command:
```
systemctl start omnia.service
systemctl start gofer-agent.service
systemctl start spire-agent.service
```

### Configuring Spire

>spire is installed through nix.

This is based on libp2p which is a peer-to-peer networking protocol designed to enable decentralized communication and file sharing over the internet. It is a modular, open-source networking protocol that allows nodes to communicate with each other directly, without the need for a central server or infrastructure.


 One of these attributes is the "transport" attribute, which uses libp2p to establish direct peer-to-peer connections between nodes without relying on a central server or infrastructure, So we have to give the peerID for example `/ip4/192.168.18.109/tcp/37705/p2p/12D3KooWPFpaE13gph8p6jdNGJv1M6fwDro8kdst53MUzVpuSJUL` i.e **"\<ip-version>/\<host>/\<protocol>/\<port>/\<type>/\<peer_id>"** w.r.t the quorum of median.To obtain the peer addresses, you can check the logs of Spire using journalctl, as Spire runs as a systemd service. Once you have the peer addresses, then ** you can add them to the "directPeersAddrs" array to connect peers in the "transport" attribute of the Spire configuration file. **

```
sudo journalctl -u <spire-agent.service> -n 100 -b -f
```


**add peerIDs in this attr to connect spire with each other**
```json 
"transport":{
      "libp2p": {
        "directPeersAddrs":[]}}
  ```

### command to run spire
```
spire agent -c <CONFIG_PATH> --log.verbosity debug
```
### command to run spire systemd
```
systemctl start <spire-agent.service>

```

### command to run ssb-server systemd
```
systemctl start <ssb-server.service>

```

The installed Scuttlebot config can be found in `~/.ssb.config`, more details
about the [Scuttlebot config](https://github.com/ssbc/ssb-config#configuration).

### Creating and Accepting SSB Invites 

Open the systemd file of ssb-server.service to lookup the binary of ssb-server.

Creating an SSB Invite

`ssb-server invite.create 1` means `<path of ssb-server> invite.create 1`

This will output a JSON object containing the invite code.

Accepting an SSB Invite

To accept an SSB invite without using Docker, open a terminal window and run the following command:

`ssb-server invite.accept <invite_code>`

Replace <invite_code> with the invite code obtained in the previous step.


### how it will work

 we should run the 3 feeds with the 3 spires 
 every feed should have their own omnia
 means the config file have the right omnia addr pasted in feed object of spire's config


**Example configuration:Relay**

```json
{
  "mode": "relay",
  "ethereum": {
    "from": "0x",
    "keystore": "",
    "password": "",
    "network": "goerli",
    "gasPrice": {
      "source": "node",
      "multiplier": 1
		}
  },
  "transports":["ssb"],
  "feeds": [
    "0xdeadbeef123"
  ],
  ...
}
```

**Example configuration:Feed**
```json
{
  "mode": "feed",
  "ethereum": {
    "from": "0x86B5B8Fe2B467F733c0624e13b9Df08867d94B96",
    "keystore": "/home/admin/.ethereum/keystore",
    "password": "/home/admin/.ethereum/keystore/.pass",
    "type": "ethereum",
    "network": "http://127.0.0.1:8545"
  },
  "options": {
    "interval": 60,
    "msgLimit": 35,
    "srcTimeout": 10,
    "setzerTimeout": 10,
    "setzerCacheExpiry": 120,
    "setzerMinMedian": 1,
    "setzerEthRpcUrl": "http://127.0.0.1:9989",
    "verbose": true,
    "logFormat": "text"
  },
  "sources": [
    "gofer","setzer"
  ],
  "transports": [
    "spire"
  ],
  "pairs": {
    "ETH/USD": {
      "msgExpiration": 1800,
      "msgSpread": 0.5
    }
  }
}
```
