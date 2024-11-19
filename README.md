# Warden Node Kurulum Rehberi

Bu rehber, Warden protokolü için bir node kurulumunun nasıl yapılacağını adım adım açıklar.

---

## Manuel Kurulum

### 1. Bağımlılıkların Kurulumu

Sistemi güncelleyin ve gerekli araçları yükleyin:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```

```bash
cd $HOME
VER="1.23.2"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

WALLET yerine cüzdan isminizi, MONIKER yerine validator adınızı yazmayı unutmayın:
```bash
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export WARDEN_CHAIN_ID="chiado_10010-1"" >> $HOME/.bash_profile
echo "export WARDEN_PORT="18"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

Binary İndirme ve Derleme
```bash
cd $HOME
rm -rf bin
mkdir bin && cd bin
wget https://github.com/warden-protocol/wardenprotocol/releases/download/v0.5.2/wardend_Linux_x86_64.zip
unzip wardend_Linux_x86_64.zip
chmod +x wardend
mv $HOME/bin/wardend $HOME/go/bin
```

 Node'u Başlatma ve Yapılandırma
```bash
wardend init $MONIKER
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${WARDEN_PORT}657\"|" $HOME/.warden/config/client.toml
```

Genesis ve Addrbook Dosyalarını İndirme
```bash
wget -O $HOME/.warden/config/genesis.json https://raw.githubusercontent.com/CoinHuntersTR/props/refs/heads/main/warden/Chiado/genesis.json
wget -O $HOME/.warden/config/addrbook.json  https://raw.githubusercontent.com/CoinHuntersTR/props/refs/heads/main/warden/Chiado/addrbook.json
```

Seeds ve Peers Ayarları
```bash
URL="https://warden-testnet-rpc.itrocket.net/net_info"
response=$(curl -s $URL)
PEERS=$(echo $response | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):" + (.node_info.listen_addr | capture("(?<ip>.+):(?<port>[0-9]+)$").port)' | paste -sd "," -)
echo "PEERS=\"$PEERS\""

sed -i -e "s|^seeds *=.*|seeds = \"$SEEDS\"|; s|^persistent_peers *=.*|persistent_peers = \"$PEERS\"|" $HOME/.warden/config/config.toml
```

Pruning Ayarları
```bash
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.warden/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.warden/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.warden/config/app.toml
```

Minimum Gas Price ve Diğer Ayarlar
```bash
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "25000000award"|g' $HOME/.warden/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.warden/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.warden/config/config.toml
```

Servis Dosyasını Oluşturma
```bash
sudo tee /etc/systemd/system/wardend.service > /dev/null <<EOF
[Unit]
Description=Warden node
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/.warden
ExecStart=$(which wardend) start --home $HOME/.warden
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

Servisi Başlatma
```bash
sudo systemctl daemon-reload
sudo systemctl enable wardend
sudo systemctl restart wardend && sudo journalctl -u wardend -f
```

Snapshot İndirme
```bash
sudo systemctl stop wardend
cp $HOME/.warden/data/priv_validator_state.json $HOME/.warden/priv_validator_state.json.backup
rm -rf $HOME/.warden/data
```

Snapshot'i indirin:
```bash
curl -L https://server-4.itrocket.net/testnet/warden/warden_2024-10-25_32787_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.warden
mv $HOME/.warden/priv_validator_state.json.backup $HOME/.warden/data/priv_validator_state.json
```

Servisi yeniden başlatıp logları kontrol edin:
```bash
sudo systemctl restart wardend && sudo journalctl -u wardend -f
```

