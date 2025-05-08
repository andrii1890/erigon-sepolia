# Erigon Sepolia Archive Node   ERIGON+CAPLIN
  **Erigon by default is "all in one binary" solution Consensus Layer + Execution Layer**
  
  - Execution Layer - Erigon
  - Consensus Layer - Caplin
  
    Caplin is a full-fledged validating Consensus Client like Prysm, Lighthouse, Teku, Nimbus and Lodestar. Its goal is:
     - provide better stability
     - validation of the chain
     - stay in sync
     - keep the execution of blocks on chain tip
     - serve the Beacon API using a fast and compact data model alongside low CPU and memory usage.

    Caplin's Usage
    
       Caplin is be enabled by default. to disable it and enable the Engine API, use the --externalcl flag. From that point on, an external Consensus Layer will be need.

       Caplin also has an archivial mode for historical states and blocks. it can be enabled through the --caplin.archive flag. In order to enable the caplin's Beacon API, the flag --beacon.api=<namespaces> must be added. e.g: -- 
       beacon.api=beacon,builder,config,debug,node,validator,lighthouse will enable all endpoints.
    
    **NOTE: Caplin is not staking-ready so aggregation endpoints are still to be implemented. Additionally enabling the Beacon API will lead to a 6 GB higher RAM usage.
  
    Archive Node Ethereum Sepolia: 1.1TB (May 2025) and FullNode: 0.6TB (May 2025)
  
  
## Hardware Requirements
- At least 4 cores, 8 threads
- 32-GB RAM or more
- 1.92Tb - NVMe SSD
- SSD or NVMe. Do not recommend HDD - on HDD Erigon will always stay N blocks behind chain tip, but not fall behind. Bear in mind that SSD performance deteriorates when close to capacity
- Golang >= 1.22; GCC 10+ or Clang; On Linux: kernel > v4. 64-bit architecture
- Internet Connection: A stable, high-speed internet connection and uninterrupted power supply is crucial!

## How much RAM do I need
- Baseline (ext4 SSD): 16Gb RAM sync takes 2 days, 32Gb - ~16 hours, 64Gb - ~6 hours

## Default Ports and Firewalls##
  Erigon ports
  - engine	9090	TCP	gRPC Server	Private
  - engine	42069	TCP & UDP	Snap sync (Bittorrent)	Public
  - engine	8551	TCP	Engine API (JWT auth)	Private
  - sentry	30403	TCP & UDP	eth/68 peering	Public
  - sentry	30404	TCP & UDP	eth/67 peering	Public
  - sentry	9091	TCP	incoming gRPC Connections	Private
  - rpcdaemon	8545	TCP	HTTP & WebSockets & GraphQL	Private
    
 Caplin ports
  - sentinel	4000	UDP	Peering	Public
  - sentinel	4001	TCP	Peering	Public
  - rest	5555	tcp	rest	Public
  
## First step
- **Update packages**
    ```
    sudo apt update && sudo apt upgrade -y
    ```
- **Install dependencies**
     ```
     sudo apt install curl build-essential git wget jq make gcc nano htop ncdu lz4  -y
     ```
- **Install GO**
    ```
    sudo rm -rf /usr/local/go
    curl -Ls https://go.dev/dl/go1.22.8.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
    eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
    eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
    ```

- **Install Erigon**
  
  Scripts will download and build latest version of Erigon
    ```
    wget https://goo.su/M7mbZ -O erigon_install.sh && chmod +x erigon_install.sh && ./erigon_install.sh
    ```
       
- **Create folders**
     ```
     cd $HOME && mkdir -p ethereum/sepolia/eth1/data ethereum/sepolia/eth1/config ethereum/sepolia/eth1/logs
     ```
- **Create service file**
     ```
     sudo tee /etc/systemd/system/sepolia.service > /dev/null << EOF
     [Unit]
     Description=Ethereum Sepolia Node Service
     After=network-online.target

     [Service]
     User=$USER
     ExecStart=$(which erigon) --config $HOME/ethereum/sepolia/eth1/config/config.yaml
     Restart=on-failure
     RestartSec=10
     LimitNOFILE=65535

     StandardOutput=append:$HOME/ethereum/sepolia/eth1/logs/sepolia.log
     StandardError=append:$HOME/ethereum/sepolia/eth1/logs/sepolia.log

    [Install]
    WantedBy=multi-user.target
    EOF
    sudo systemctl daemon-reload
    sudo systemctl enable sepolia.service
    ```
- **Create config file**
   ```
   tee $HOME/ethereum/sepolia/eth1/config/config.yaml > /dev/null << EOF
   datadir : '$HOME/ethereum/sepolia/eth1/data/'
   port : "30403"
   chain : "sepolia"
   identity : "rpc ethereum sepolia"
   prune.mode : "archive"
   nat : "any"
   http : "true"
   http.addr : "0.0.0.0"
   http.port : "8545"
   http.vhosts : "*"
   http.api : ["eth","engine","debug","net","trace","web3","erigon","txpool","admin","ots"]
   ws : "true"
   ws.port : "8546"
   torrent.port : "42069"
   torrent.download.rate : "900mb"
   metrics : "true"
   metrics.addr : "127.0.0.1"
   metrics.port : "6061"
   beacon.api.addr : "0.0.0.0"
   beacon.api.port : "5555"
   beacon.api.cors.allow-methods : ["GET","POST","PUT","OPTIONS"]
   beacon.api.cors.allow-origins : "*"
   beacon.api : ["beacon","builder","config","debug","node","lighthouse"]
   private.api.addr : "localhost:9090"
   caplin.blobs-no-pruning : "true"
   caplin.states-archive : "true"
   caplin.blocks-archive : "true"
   caplin.blobs-archive : "true"
   caplin.blobs-immediate-backfill : "true"
   state.cache : "1024"
   netrestrict : ["10.0.0.0/8","172.16.0.0/12","100.64.0.0/10","198.18.0.0/15","169.254.0.0/16","172.16.0.0/12","192.0.2.0/24","192.88.99.0/24","192.168.0.0/16","198.18.0.0/15","198.51.100.0/24","203.0.113.0/24","224.0.0.0/4","240.0.0.0/4","192.0.0.0/24","0.0.0.0/8","255.255.255.255/32"]
   #txpool.globalbasefeeslots : "100000"
   #txpool.globalqueue : "100000"
   #txpool.globalslots : "100000"
   #txpool.accountslots : "32"
   #rpc.gascap : "50000000"
   #rpc.batch.concurrency : "4"
   EOF
   ```   
    
- **Enable Erigon Service**
   ```
   sudo systemctl start sepolia.service && sudo systemctl status sepolia.service
   ```
   
## Third step
- **Add alias for erigon logs**
    ```
    echo "#Erigon Sepolia Logs" >> $HOME/.profile
    echo 'alias sepolia.log="tail -f $HOME/ethereum/sepolia/eth1/logs/sepolia.log"' >> $HOME/.profile
    source $HOME/.profile
    ```
    now you can simply find logs: sepolia.log

***HTTP*** request will be available on: ***http://<YOUR_IP>:8545***
  
***WS*** request will be available on: ***ws://<YOUR_IP>:8546***
  
**Beacon Client*** request will be available on: ***http://<YOUR_IP>:5555***

## Don't forget to install ufw and protect your endpoint

