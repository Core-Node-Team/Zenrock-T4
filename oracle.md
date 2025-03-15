###  Clone zenrock-validators repository
```
cd $HOME
rm -rf zenrock-validators
git clone https://github.com/zenrocklabs/zenrock-validators
```
###  Generate keys
###  Set key password
```
read -p "Enter password for the keys: " key_pass
```
###  Create sidecar directories
```
mkdir -p $HOME/.zrchain/sidecar/bin
mkdir -p $HOME/.zrchain/sidecar/keys
```
###  Build ecdsa binary
```
cd $HOME/zenrock-validators/utils/keygen/ecdsa && go build
```
###  Build bls binary
```
cd $HOME/zenrock-validators/utils/keygen/bls && go build
```
###  Generate ecdsa key
```
ecdsa_output_file=$HOME/.zrchain/sidecar/keys/ecdsa.key.json
ecdsa_creation=$($HOME/zenrock-validators/utils/keygen/ecdsa/ecdsa --password $key_pass -output-file $ecdsa_output_file)
ecdsa_address=$(echo "$ecdsa_creation" | grep "Public address" | cut -d: -f2)
```
###  Generate bls key
```
bls_output_file=$HOME/.zrchain/sidecar/keys/bls.key.json
$HOME/zenrock-validators/utils/keygen/bls/bls --password $key_pass -output-file $bls_output_file
```
###  Output
```
echo "ecdsa address: $ecdsa_address"
```
###  Set operator configuration

Don't forget
Ensure that you have configured TESTNET_HOLESKY_ENDPOINT, MAINNET_ENDPOINT, ETH_RPC_URL, ETH_WS_URL with your specific values. You can use Quicknode.com to get api keys. I used infura.io (but ffter Metamask acquired Infura they significantly reduced their limits)

###  Declare variables

```
EIGEN_OPERATOR_CONFIG="$HOME/.zrchain/sidecar/eigen_operator_config.yaml"
TESTNET_HOLESKY_ENDPOINT="YOUR_TESTNET_HOLESKY_ENDPOINT"
MAINNET_ENDPOINT="YOUR_ETH_MAINNET_ENDPOINT"
OPERATOR_VALIDATOR_ADDRESS=$(zenrockd keys show wallet --bech val -a)
OPERATOR_ADDRESS=$ecdsa_address
ETH_RPC_URL="YOUR_TESTNET_HOLESKY_RPC"
ETH_WS_URL="YOUR_TESTNET_HOLESKY_WS"
SOLANA_MAINNET_ENDPOINT="YOUR_SOLANA_MAINNET_ENDPOINT"
ECDSA_KEY_PATH=$ecdsa_output_file
BLS_KEY_PATH=$bls_output_file
PROXY_URL="url" 
PROXY_USER="user"
PROXY_PASSWORD="password"
```
###  Create initial configuration files
```
sudo tee $HOME/.zrchain/sidecar/config.yaml > /dev/null <<EOF
enabled: true
grpc_port: 9191
zrchain_rpc: "localhost:${ZENROCK_PORT}790"
state_file: "$HOME/.zrchain/sidecar/cache.json"
operator_config: "${EIGEN_OPERATOR_CONFIG}"
network: "testnet"

eth_oracle:
  rpc:
    local: "http://127.0.0.1:8545"
    testnet: "${TESTNET_HOLESKY_ENDPOINT}"
    mainnet: "${MAINNET_ENDPOINT}"
  contract_addrs:
    service_manager: "0xa559CDb9e029fc4078170122eBf7A3e622a764E4"
    price_feeds:
      btc: "0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c"
      eth: "0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419"
    zenbtc:
      controller:
        testnet: "0xaCE3634AAd9bCC48ef6A194f360F7ACe51F7d9f1"
      token:
        ethereum:
          testnet: "0xfA32a2D7546f8C7c229F94E693422A786DaE5E18"
  network_name:
    mainnet: "Ethereum Mainnet"
    testnet: "Holešky Ethereum Testnet"
solana_rpc:
  testnet: "https://api.testnet.solana.com"
  mainnet: "${SOLANA_MAINNET_ENDPOINT}"

proxy_rpc:
  url: "${PROXY_URL}"
  user: "${PROXY_USER}"
  password: "${PROXY_PASSWORD}"
  
neutrino:
  path: "$HOME/.zrchain/neutrino"
EOF
Copy
sudo tee $HOME/.zrchain/sidecar/eigen_operator_config.yaml > /dev/null <<EOF
register_operator_on_startup: true
register_on_startup: true
production: true
kind: Secret
#To be manually updated
operator_address: ${OPERATOR_ADDRESS}
operator_validator_address: ${OPERATOR_VALIDATOR_ADDRESS}

# EigenLayer Slasher contract address
# This is the address of the contracts which are deployed in the anvil saved state
# The saved eigenlayer state is located in tests/anvil/credible_squaring_avs_deployment_output.json
avs_registry_coordinator_address: 0xdc3A1b2a44D18c6B98a1d6c8C042247d2F5AC722
operator_state_retriever_address: 0xdB55356826a16DfFBD86ba334b84fC4E37113d97

# ETH RPC URL
eth_rpc_url: "${ETH_RPC_URL}" # holesky rpc
eth_ws_url: "${ETH_WS_URL}" # holesky ws rpc

# ECDSA key
ecdsa_private_key_store_path: ${ECDSA_KEY_PATH}

# We are using bn254 curve for bls keys
bls_private_key_store_path: ${BLS_KEY_PATH}

# address which the aggregator listens on for operator signed messages
aggregator_server_ip_port_address: avs-aggregator.gardia.zenrocklabs.io:8090

# avs node spec compliance https://eigen.nethermind.io/docs/spec/intro
eigen_metrics_ip_port_address: 0.0.0.0:9292
enable_metrics: true
metrics_address: 0.0.0.0:9292
node_api_ip_port_address: 0.0.0.0:9191
enable_node_api: true

# address of token to deposit tokens into when registering on startup
token_strategy_addr: 0x80528D6e9A2BAbFc766965E0E26d5aB08D9CFaF9
service_manager_address: 0xa559CDb9e029fc4078170122eBf7A3e622a764E4
zr_chain_rpc_address: localhost:${ZENROCK_PORT}790
EOF
```
###  Download sidecar binary
```
wget -O $HOME/.zrchain/sidecar/bin/validator_sidecar https://github.com/Zenrock-Foundation/zrchain/releases/download/v5.16.9/validator_sidecar
chmod +x $HOME/.zrchain/sidecar/bin/validator_sidecar
```
###  Create and run sidecar service
```
sudo tee /etc/systemd/system/zenrock-testnet-sidecar.service > /dev/null <<EOF
[Unit]
Description=Validator Sidecar
After=network-online.target

[Service]
User=$USER
ExecStart=$HOME/.zrchain/sidecar/bin/validator_sidecar
Restart=on-failure
RestartSec=30
LimitNOFILE=65535
Environment="OPERATOR_BLS_KEY_PASSWORD=$key_pass"
Environment="OPERATOR_ECDSA_KEY_PASSWORD=$key_pass"
Environment="SIDECAR_CONFIG_FILE=$HOME/.zrchain/sidecar/config.yaml"

[Install]
WantedBy=multi-user.target
EOF
Copy
sudo systemctl daemon-reload
sudo systemctl enable zenrock-testnet-sidecar.service
sudo systemctl start zenrock-testnet-sidecar.service
```
###  Check the service logs
```
journalctl -fu zenrock-testnet-sidecar.service -o cat
```
