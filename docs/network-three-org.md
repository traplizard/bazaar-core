# Create a local network of 3 organizations

## Quick start

Use script [network-create-local.sh](./network-create-local.sh) to create a network with an arbitrary 
number of member organizations running in docker containers on the local machine.
```bash
./network-create-local.sh org1 org2 org3
```

This will create a network with `bazaar.com` domain and container names like `peer0.org1.bazaar.com`, 
`api.org2.bazaar.com` and API ports 4000, 4001, 4002; will create a channel named *common* 
and nodejs chaincode *reference* with its source in this repo [./chaincode/node/reference](./chaincode/node/reference). 

Member organization's docker containers are started with default docker-compose config files 
`-f docker-compose.yaml -f docker-compose-couchdb.yaml -f docker-compose-ports.yaml`. 
You can override them by setting env variable `DOCKER_COMPOSE_ARGS`; for
example to start without a CouchDb container to use LevelDb for storage:
```bash
DOCKER_COMPOSE_ARGS="-f docker-compose.yaml -f docker-compose-ports.yaml" ./network-create-local.sh
```
 
You can give your network and channel names, set starting API port, override chaincode location, 
install and instantiate arguments.
```bash
DOMAIN=mynetwork.org \
CHANNEL=a-b \
WEBAPP_HOME=/home/oleg/webapp \
CHAINCODE_HOME=/home/oleg/chaincode \
CHAINCODE_INSTALL_ARGS='example02 1.0 chaincode_example02 golang' \
CHAINCODE_INSTANTIATE_ARGS="a-b example02 [\"init\",\"a\",\"10\",\"b\",\"0\"] 1.0 collections.json AND('a.member','b.member')" \
./network-create-local.sh a b
```

To understand the script please read the below step by step instructions for the network 
of three member organizations org1, org2, org3.

You can also extend this example by manually adding more than 3 organizations and any number of channels 
with various membership.

## Create orderer and member organizations and add them to the consortium

Clean up. Remove all containers, delete local crypto material:
```bash
./clean.sh
```

Start docker containers of the *orderer* organization (crypto-materials, certificates and keys will be auto-generated inside the containers):
```bash
docker-compose -f docker-compose-orderer.yaml up
```

Open another shell. Start *org1* (crypto-materials, certificates and keys will be auto-generated inside the containers):
```bash
docker-compose -f docker-compose.yaml -f docker-compose-api-port.yaml up
```

Open another shell. Note since we're reusing the same `docker-compose.yaml` file we need to redefine `COMPOSE_PROJECT_NAME`.
Redefine the api port mapped to the host to avoid collision with api.org1.

Generate and start *org2*.
```bash
export COMPOSE_PROJECT_NAME=org2 ORG=org2 
export API_PORT=4001
```

Then start the peer
```bash
docker-compose -f docker-compose.yaml -f docker-compose-api-port.yaml up -d
```

Generate and start *org3* in another shell:
```bash
export COMPOSE_PROJECT_NAME=org3 ORG=org3 
export API_PORT=4002

docker-compose -f docker-compose.yaml -f docker-compose-api-port.yaml up -d
```

Now you should have 4 console windows running containers of *orderer*, *org1*, *org2*, *org3* organizations.

Open another console where we'll become an *Admin* user of the *orderer* organization. We'll add orgs to the consortium:
```bash
./consortium-add-org.sh org1
./consortium-add-org.sh org2
./consortium-add-org.sh org3
``` 

Now all 3 orgs are known in the consortium and can create and join channels.

## Create channels, install and instantiate chaincodes

Open another console where we'll become *org1* again. We'll create channel *common*, add other orgs to it, 
and join our peers to the channel:
```bash
./channel-create.sh common
./channel-join.sh common
./channel-add-org.sh common org2
./channel-add-org.sh common org3 
``` 

Let's create a bilateral channel between *org1* and *org2* and join to it:
```bash
./channel-create.sh org1-org2
./channel-join.sh org1-org2
./channel-add-org.sh org1-org2 org2
```

Install and instantiate chaincode *reference* on channel *common*. Note the path to the source code is inside `cli` 
docker container and is mapped to the local  `./chaincode/node/reference`
```bash
./chaincode-install.sh reference
./chaincode-instantiate.sh common reference 
```

Install and instantiate chaincode *relationship* on channel *org1-org2*:
```bash
./chaincode-install.sh relationship
./chaincode-instantiate.sh org1-org2 relationship '["init","a","10","b","0"]'
```

Open another console where we'll become *org2* to install chaincodes *reference* and  *relationship* 
and to join channels *common* and *org1-org2*:
```bash
export COMPOSE_PROJECT_NAME=org2 ORG=org2

./chaincode-install.sh reference
./chaincode-install.sh relationship
./channel-join.sh common
./channel-join.sh org1-org2
``` 

Now become *org3* to install chaincode *reference* and join channel *common*:
```bash
export COMPOSE_PROJECT_NAME=org3 ORG=org3

./chaincode-install.sh reference
./channel-join.sh common
``` 
