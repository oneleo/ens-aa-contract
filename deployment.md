## 1. Deploy and verify ENSRegistry

### Deploy ENSRegistry contract

- Edit the `.env.deployment` file by copying from `.env.deployment.example` and then run the following command.

```shell
source .env.deployment

ENS_REGISTRY_ADDRESS=$(forge create --rpc-url ${NODE_RPC_URL} --private-key ${DEPLOYER_PRIVATE_KEY} --use ${COMPILER_VERSION} "lib/ens-contracts/contracts/registry/ENSRegistry.sol:ENSRegistry" | sed -nr 's/^Deployed to: (0x[0-9a-zA-Z]{40})[.]*$/\1/p')
```

### Verify ENSRegistry contract

```shell
forge verify-contract --watch --chain ${CHAIN_ID} --verifier "etherscan" --etherscan-api-key ${ETHERSCAN_API_KEY} --compiler-version ${COMPILER_VERSION} ${ENS_REGISTRY_ADDRESS} "lib/ens-contracts/contracts/registry/ENSRegistry.sol:ENSRegistry"
```

- Output sample

```shell
Start verifying contract `0x0f43676C2ca314BE4863629170f94E5F2A9E1220` deployed on sepolia

Submitting verification for [lib/ens-contracts/contracts/registry/ENSRegistry.sol:ENSRegistry] 0x0f43676C2ca314BE4863629170f94E5F2A9E1220.
Submitted contract for verification:
        Response: `OK`
        GUID: `rpbu8gmqabxvmqpmf3c1vmjuqfgpqwc9egp4h4cmmfl4htuxhb`
        URL: https://sepolia.etherscan.io/address/0x0f43676c2ca314be4863629170f94e5f2a9e1220
Contract verification status:
Response: `NOTOK`
Details: `Pending in queue`
Contract verification status:
Response: `OK`
Details: `Pass - Verified`
Contract successfully verified
```

## 2. Deploy and verify OffchainResolver

### Deploy OffchainResolver contract

```shell
OFFCHAIN_RESOLVER_ADDRESS=$(forge create --rpc-url ${NODE_RPC_URL} --private-key ${DEPLOYER_PRIVATE_KEY} --use ${COMPILER_VERSION} "lib/offchain-resolver/packages/contracts/contracts/OffchainResolver.sol:OffchainResolver" --constructor-args ${SECOND_LEVEL_DOMAIN_RESOLVER_URL} "[${SECOND_LEVEL_DOMAIN_OWNER}]" | sed -nr 's/^Deployed to: (0x[0-9a-zA-Z]{40})[.]*$/\1/p')
```

### Verify OffchainResolver contract

```shell
forge verify-contract --watch --chain ${CHAIN_ID} --verifier "etherscan" --etherscan-api-key ${ETHERSCAN_API_KEY} --compiler-version ${COMPILER_VERSION} --constructor-args $(cast abi-encode "constructor(string memory _url, address[] memory _signers)" ${SECOND_LEVEL_DOMAIN_RESOLVER} "[${SECOND_LEVEL_DOMAIN_OWNER}]") ${OFFCHAIN_RESOLVER_ADDRESS} "lib/offchain-resolver/packages/contracts/contracts/OffchainResolver.sol:OffchainResolver"
```

- Output sample

```shell
Start verifying contract `0xfd84a4804Fd8BA30a8b26Ee526D08827d1d272C0` deployed on sepolia

Submitting verification for [lib/offchain-resolver/packages/contracts/contracts/OffchainResolver.sol:OffchainResolver] 0xfd84a4804Fd8BA30a8b26Ee526D08827d1d272C0.
Submitted contract for verification:
        Response: `OK`
        GUID: `wfqnchepttnrs6sht7pv33gshd3n52qsxqychbxhpmttdmwjrp`
        URL: https://sepolia.etherscan.io/address/0xfd84a4804fd8ba30a8b26ee526d08827d1d272c0
Contract verification status:
Response: `NOTOK`
Details: `Pending in queue`
Contract verification status:
Response: `OK`
Details: `Pass - Verified`
Contract successfully verified
```

## 3. Set subnode owner and resolver

### Set subnode owner

```shell
# Set `eth` node owner
cast send --rpc-url ${NODE_RPC_URL} --private-key ${DEPLOYER_PRIVATE_KEY} ${ENS_REGISTRY_ADDRESS} "setSubnodeOwner(bytes32 node, bytes32 label, address owner)" ${ROOT_DOMAIN} $(cast keccak256 ${TOP_LEVEL_DOMAIN}) ${TOP_LEVEL_DOMAIN_OWNER}

# Set `token.eth` node owner
cast send --rpc-url ${NODE_RPC_URL} --private-key ${DEPLOYER_PRIVATE_KEY} ${ENS_REGISTRY_ADDRESS} "setSubnodeOwner(bytes32 node, bytes32 label, address owner)" $(cast namehash ${TOP_LEVEL_DOMAIN}) $(cast keccak256 ${SECOND_LEVEL_DOMAIN}) ${SECOND_LEVEL_DOMAIN_OWNER}
```

### Set subnode resolver

```shell
# Set `token.eth` node resolver
cast send --rpc-url ${NODE_RPC_URL} --private-key ${DEPLOYER_PRIVATE_KEY} ${ENS_REGISTRY_ADDRESS} "setResolver(bytes32 node, address resolver)" $(cast namehash "${SECOND_LEVEL_DOMAIN}.${TOP_LEVEL_DOMAIN}") ${OFFCHAIN_RESOLVER_ADDRESS}
```

## 4. Get subnode owner and resolver

### Get subnode owner

```shell
# Calculate `eth` node
TOP_LEVEL_DOMAIN_NODE=$(cast keccak256 $(cast abi-encode "bytes32(bytes32,bytes32)" ${ROOT_DOMAIN} $(cast keccak256 ${TOP_LEVEL_DOMAIN})))

# Calculate `token.eth` node
SECOND_LEVEL_DOMAIN_NODE=$(cast keccak256 $(cast abi-encode "bytes32(bytes32,bytes32)" $(cast namehash ${TOP_LEVEL_DOMAIN}) $(cast keccak256 ${SECOND_LEVEL_DOMAIN})))

# Get `eth` node owner
TOP_LEVEL_DOMAIN_OWNER_ADDRESS=$(cast parse-bytes32-address $(cast call --rpc-url ${NODE_RPC_URL} ${ENS_REGISTRY_ADDRESS} "owner(bytes32 node)" ${TOP_LEVEL_DOMAIN_NODE}))

echo ${TOP_LEVEL_DOMAIN_OWNER_ADDRESS}

# Get `token.eth` node owner
SECOND_LEVEL_DOMAIN_OWNER_ADDRESS=$(cast parse-bytes32-address $(cast call --rpc-url ${NODE_RPC_URL} ${ENS_REGISTRY_ADDRESS} "owner(bytes32 node)" ${SECOND_LEVEL_DOMAIN_NODE}))

echo ${SECOND_LEVEL_DOMAIN_OWNER_ADDRESS}
```

### Get subnode resolver

```shell
# Get `token.eth` node resolver
SECOND_LEVEL_DOMAIN_RESOLVER_ADDRESS=$(cast parse-bytes32-address $(cast call --rpc-url ${NODE_RPC_URL} ${ENS_REGISTRY_ADDRESS} "resolver(bytes32 node)" ${SECOND_LEVEL_DOMAIN_NODE}))

echo ${SECOND_LEVEL_DOMAIN_RESOLVER_ADDRESS}

# Get `token.eth` node resolver url
SECOND_LEVEL_DOMAIN_RESOLVER_URL_BYTES=$(cast call --rpc-url ${NODE_RPC_URL} ${SECOND_LEVEL_DOMAIN_RESOLVER_ADDRESS} "url()")

# Trim prefix and convert to ASCII
SECOND_LEVEL_DOMAIN_RESOLVER_URL=$(cast to-ascii $(echo ${SECOND_LEVEL_DOMAIN_RESOLVER_URL_BYTES} | awk '{print substr($0, 131)}'))

echo ${SECOND_LEVEL_DOMAIN_RESOLVER_URL}
```
