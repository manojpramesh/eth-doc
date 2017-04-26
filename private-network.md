# Create private network

### 1. create a genesis file

Every blockchain starts with a genesis block. For a private network, we have to create a custom genesis file.

Here's an example of a custom genesis file.

``` json
{
  "alloc": {
    "1d9924e3efbc0398d12f94146a3004f734662f9e": {
      "balance": "1000000000000000000000000000000"
    }
  },

  "nonce": "0x0000000000000042",
  "difficulty": "0x6666",
  "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "timestamp": "0x00",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "extraData": "0x11bbe8db4e347b4e8c937c1c8370e4b5ed33adb3db69cbdb7a38e1e50b1b82fa",
  "gasLimit": "0x4c4b40"
}
```

### 2. Choose a network id

Every network has its own network id. The main network has id 1. Use the `--networkid` command line option for this.


### 3. Initialize genesis file

Initialize the genesis file using the below command.

```
geth --datadir path/to/data/directory init path/to/genesis.json
```

### 4. start ethereum node

start a node using the given command

```
geth --datadir path/to/data/directory --networkid "1233232" --identity "identity-name" --port "30303" --nat "any" --rpccorsdomain "*" --rpc console
```
<br>

#### todo

1. command explanation
2. add peers
3. boot nodes
4. static nodes
5. ports and firewall