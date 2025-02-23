Dependencies Installation

# Install dependencies for building from source
sudo apt update
sudo apt install -y curl git jq lz4 build-essential

# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
Node Installation

Node Name

Your Node Name
Port prefix

266
# Download binary
cd $HOME && mkdir -p go/bin/
wget https://github.com/airchains-network/junction/releases/download/v0.1.0/junctiond
chmod +x junctiond
mv junctiond $HOME/go/bin/

# Prepare cosmovisor directories
mkdir -p $HOME/.junction/cosmovisor/genesis/bin
ln -s $HOME/.junction/cosmovisor/genesis $HOME/.junction/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which junctiond) $HOME/.junction/cosmovisor/genesis/bin

# Set node CLI configuration
junctiond config chain-id junction
junctiond config keyring-backend test
junctiond config node tcp://localhost:26657

# Initialize the node
junctiond init "Your Node Name" --chain-id junction

# Download genesis and addrbook files
curl -L https://snapshots-testnet.nodejumper.io/airchains/genesis.json > $HOME/.junction/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/airchains/addrbook.json > $HOME/.junction/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "575e98598e9813a26576759c7ef70fd38d2516a4@junction-testnet-rpc.synergynodes.com:15656,04e2fdd6ec8f23729f24245171eaceae5219aa91@airchains-testnet-seed.itrocket.net:19656,aeaf101d54d47f6c99b4755983b64e8504f6132d@airchain-testnet-peer.dashnode.org:28656,bb26fc8cef05cee75d4cae3f25e17d74c7913967@airchains-t.seed.stavr.tech:4476,df949a46ae6529ae1e09b034b49716468d5cc7e9@testnet-seeds.stakerhouse.com:13756,48887cbb310bb854d7f9da8d5687cbfca02b9968@35.200.245.190:26656,60133849b4c83531eb2d835970035a0f08868658@65.109.93.124:28156,df2a56a208821492bd3d04dd2e91672657c79325@airchain-testnet-peer.cryptonode.id:27656,04e2fdd6ec8f23729f24245171eaceae5219aa91@airchains-testnet-seed.itrocket.net:19656,3dc2f101876e1a26730f99c06a5a2eb6e2cc2349@65.21.69.53:33656"|' $HOME/.junction/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.00025amf"|' $HOME/.junction/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.junction/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.junction/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots-testnet.nodejumper.io/airchains/airchains_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.junction"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/airchains.service > /dev/null << EOF
[Unit]
Description=Airchains node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.junction
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.junction"
Environment="DAEMON_NAME=junctiond"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable airchains.service

# Start the service and check the logs
sudo systemctl start airchains.service
sudo journalctl -u airchains.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
