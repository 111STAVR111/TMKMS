<h1 align="center">TMKMS</h1>

![tmkms](https://github.com/111STAVR111/TMKMS/assets/77785195/3cac0560-034e-4ba8-bcb6-82c1975eeccf)


### The Tendermint Key Management System (or TMKMS) should be used by any validator currently or intending to be in the active validator set. This application mitigates the risk of double-signing and provides high-availability to validator keys while keeping these keys on a separate physical host. While TMKMS can be used on the same machine as the validator, it is recommended to be on a separate host.
[DOCS](https://docs.osmosis.zone/osmosis-core/keys/tmkms/)

## Let's look at an example - `Canto`

## Create new user (from root)
```python
adduser tmkms
usermod -aG sudo tmkms
su tmkms
cd $HOME
```

## Install RUST
```python
curl --proto '=https' --tlsv1.3 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
#Install GCC
sudo apt update & sudo apt install build-essential curl jq  --yes
```
## Compile and sort TMKMS binaries
```python
cd $HOME
cargo install tmkms --features=softsign
sudo mv $HOME/.cargo/bin/tmkms /usr/local/bin/
```

## Create and Init TKMS working directory
```python
mkdir -p $HOME/tmkms/canto
tmkms init $HOME/tmkms/canto
```

## Import Private key
- Upload your validator `priv_validator_key.json` to directory `/home/tmkms/priv_validator_key.json`
### Then check availablity 
```python
cat $HOME/priv_validator_key.json
```
## If right output is appeared, follow next step below
```python
tmkms softsign import $HOME/priv_validator_key.json $HOME/tmkms/canto/secrets/canto-consensus.key
```
### Now we can erase copy of original file
```python
sudo shred -uvz $HOME/priv_validator_key.json
```

- Swap `tmkms.toml` to the one below. The only `"addr ="` field edit need to be done, replace it with your validator node `IP + port(26658 default)`
```python
rm -rf ~/tmkms/canto/tmkms.toml
tee ~/tmkms/canto/tmkms.toml << EOF
#Tendermint KMS configuration file
[[chain]]
id = "canto_7700-1"
key_format = { type = "bech32", account_key_prefix = "cantopub", consensus_key_prefix = "cantovalcons" }
state_file = "$HOME/tmkms/canto/state/canto_7700-1_priv_validator_state.json"
#Software-based Signer Configuration
[[providers.softsign]]
chain_ids = ["canto_7700-1"]
key_type = "consensus"
path = "$HOME/tmkms/canto/secrets/canto-consensus.key"
#Validator Configuration
[[validator]]
chain_id = "canto_7700-1"
addr = "tcp://60.19.92.21:10218" #Set here IP and port of the canto node U will be using for signing blocks (port can be custom)   
secret_key = "$HOME/tmkms/canto/secrets/kms-identity.key"
protocol_version = "v0.34"
reconnect = true
EOF
```
### Create service file and run TMKMS
```python
sudo tee /etc/systemd/system/tmkmsd-canto.service << EOF
[Unit]
Description=TMKMS-canto
After=network.target
StartLimitIntervalSec=0
[Service]
Type=simple
Restart=always
RestartSec=10
User=$USER
ExecStart=$(which tmkms) start -c $HOME/tmkms/canto/tmkms.toml
LimitNOFILE=1024
[Install]
WantedBy=multi-user.target
EOF
```
## Start 
```python
sudo systemctl daemon-reload
sudo systemctl enable tmkmsd-canto.service
sudo systemctl restart tmkmsd-canto.service
sudo systemctl status tmkmsd-canto.service
sudo journalctl -fu tmkmsd-canto.service -o cat
```

- #ERROR `tmkms::client: [canto_7700-1@tcp://91.19.90.20:21218] I/O error: Connection refused (os error 111)`
<h2 align="center">Its NORMAL</h2>

![error](https://github.com/111STAVR111/TMKMS/assets/77785195/1c39f6de-0fa7-48a5-b2c8-af7da0397935)

- LAST STEPS. Activate signing from `canto node` side
- Find field `priv_validator_laddr = ""` at dir `$HOME/.cantod/config/config.toml` and edit to your Validator `IP + port`
- Make sure your firewall open only for KMS server IP to allow connect to port 26658 (or any custom port u set)
- Example : `priv_validator_laddr = "tcp://0.0.0.0:26658"`  (Line 68 +-)

### Restarting the node Validator Node
```python
sudo systemctl restart cantod && sudo journalctl -fu cantod -o cat
```

<h2 align="center">make sure that the logs are good  </h2>

![Good](https://github.com/111STAVR111/TMKMS/assets/77785195/d0fe10a7-8db0-473f-926e-188aa9ef7137)

- delete `priv_validator_key.json` from the validator node and restart again. Everything should work


<h2 align="center">Helpful commands</h2>

`su tmkms && cd`
## Logs
`sudo journalctl -fu tmkmsd-canto -o cat`
### Restart
`sudo systemctl restart tmkmsd-canto && sudo journalctl -fu tmkmsd-canto -o cat`


