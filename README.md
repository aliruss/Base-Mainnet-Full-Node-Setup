# Base Mainnet Full Node Setup (Geth + Docker)

This guide walks you through setting up a **Base Mainnet Full Node** using `geth`, downloading a pre-synced snapshot, and monitoring synchronization progress. The node is managed using Docker and docker-compose for reliability and portability.

---

## 🔧 Hardware Recommendations

- **CPU**: 8 cores minimum (e.g., AMD EPYC or Intel Xeon)
- **RAM**: 32 GB minimum
- **Disk**: 10 TB NVMe SSD (essential for full sync speed and IOPS)
- **Network**: At least 100 Mbps (preferably 1 Gbps)
- **OS**: Ubuntu 22.04 LTS recommended

---

## 🐳 Install Docker and Docker Compose

```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove -y $pkg; done

sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test Docker
sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
```

---

## 📁 Clone Base Node Repository

```bash
git clone https://github.com/base/node.git base-node
cd base-node
```

---

## 🔐 Create `.env.mainnet` File

```bash
nano .env.mainnet
```

Paste and update the following with your preferred values:

```env
OP_NODE_L1_ETH_RPC=<your-preferred-l1-rpc>
OP_NODE_L1_BEACON=<your-preferred-l1-beacon>
OP_NODE_L1_BEACON_ARCHIVER=<your-preferred-l1-beacon-archiver>
```

---

## 📦 Download and Extract Latest Geth Snapshot

You can download a recent snapshot from either the official Base snapshot site or PublicNode, which offers smaller and faster alternatives.

### Option 1: Official Base Snapshot

```bash
mkdir -p /home/base-node/{snapshots,geth-data}

aria2c -x 16 -s 16 -k 2M -d /home/base-node/snapshots -o base-geth-full.zst \
  "https://mainnet-full-snapshots.base.org/$(curl -s https://mainnet-full-snapshots.base.org/latest)" && \
  tar -I zstd -xvf /home/base-node/snapshots/base-geth-full.zst -C /home/base-node/geth-data
```

After extraction, your Geth data will be under:

```
/home/base-node/geth-data/snapshots/mainnet/download/geth
```

### Option 2: PublicNode Snapshot (Faster & Smaller)

You can also use [https://www.publicnode.com/snapshots](https://www.publicnode.com/snapshots) to find smaller Geth snapshots for Base. Choose the latest `.tar.lz4` file listed under Base.

To download and extract:

```bash
mkdir -p /home/base-node/geth-data

aria2c -x 16 -s 16 -k 2M -o - "https://snapshots.publicnode.com/base-geth-part-33474880.tar.lz4" | \
  lz4 -d | tar -xvf - -C /home/base-node/geth-data
```

---

## 🔄 Move Data to Correct Location

```bash
mv /home/base-node/geth-data/snapshots/mainnet/download/geth /home/base-node/geth-data
```

Ensure structure is:

```
/home/base-node/geth-data/geth/chaindata
```

---

## 🧰 Start the Docker Containers

```bash
docker compose up --build -d
```

---



## 🧪 Check Geth Sync Status (Inside Container)

To monitor your running containers and check sync status, follow these steps:

1. List all active containers:

```bashbash
docker ps
```

This will show you container names (e.g., `base-node-execution-1`) that you can use in the next commands.

2. Check logs for your execution container:

```bash
docker logs -f base-node-execution-1
```

3. Quick block comparison (local vs remote):

````bash
echo "Local Block:" $(docker exec base-node-execution-1 geth attach --exec eth.blockNumber /data/geth.ipc) && echo "Remote Block:" $(curl -s https://api.basescan.org/api?module=proxy&action=eth_blockNumber | jq -r '.result' | xargs printf "%d
")
```bash
docker logs -f base-node-execution-1
````

---

## 📊 Monitoring Script with ETA

Save this as `check_sync.sh`:

```bash
#!/bin/bash

REMOTE_BLOCK=$(curl -s https://api.basescan.org/api?module=proxy&action=eth_blockNumber&apikey=YourApiKeyToken | jq -r '.result')
REMOTE_BLOCK_DEC=$((16#${REMOTE_BLOCK:2}))

START_BLOCK=$(docker exec base-node-execution-1 geth attach --exec "eth.blockNumber" /data/geth.ipc)
SLEEP_TIME=5

echo "⏳ Checking sync progress..."

while true; do
    LOCAL_BLOCK=$(docker exec base-node-execution-1 geth attach --exec "eth.blockNumber" /data/geth.ipc)
    PROGRESS=$(awk "BEGIN { printf \"%.2f\", ($LOCAL_BLOCK / $REMOTE_BLOCK_DEC) * 100 }")
    REMAINING=$((REMOTE_BLOCK_DEC - LOCAL_BLOCK))

    if [[ $REMAINING -le 0 ]]; then
        echo "✅ Sync Complete!"
        break
    fi

    if [[ -z "$BLOCKS_PER_SEC" ]]; then
        BLOCKS_PER_SEC=$(( ($LOCAL_BLOCK - $START_BLOCK) / 1 ))
        START_TIME=$(date +%s)
    else
        CURRENT_TIME=$(date +%s)
        ELAPSED=$((CURRENT_TIME - START_TIME))
        BLOCKS_PER_SEC=$((($LOCAL_BLOCK - $START_BLOCK) / ($ELAPSED > 0 ? $ELAPSED : 1)))
    fi

    ETA_SECONDS=$((REMAINING / (BLOCKS_PER_SEC > 0 ? BLOCKS_PER_SEC : 1)))
    ETA_H=$((ETA_SECONDS / 3600))
    ETA_M=$(((ETA_SECONDS % 3600) / 60))
    ETA_S=$((ETA_SECONDS % 60))

    echo "📦 Local Block:  $LOCAL_BLOCK"
    echo "🌍 Remote Block: $REMOTE_BLOCK_DEC"
    echo "📊 Progress:     $PROGRESS%  | Remaining: $REMAINING blocks | ETA: ${ETA_H}h ${ETA_M}m ${ETA_S}s"
    echo "--------------------------------------------"
    sleep $SLEEP_TIME
    REMOTE_BLOCK=$(curl -s https://api.basescan.org/api?module=proxy&action=eth_blockNumber&apikey=YourApiKeyToken | jq -r '.result')
    REMOTE_BLOCK_DEC=$((16#${REMOTE_BLOCK:2}))

done
```

Make it executable:

```bash
chmod +x check_sync.sh
./check_sync.sh
```

---

## ✅ When is the node fully synced?

Once your **Local Block** is equal to or just a few blocks behind the **Remote Block** (checked via `basescan.org`), your node is fully synced and production-ready.

---

## 🌐 Accessing Your Node Remotely via RPC

Your Base full node exposes the RPC interface (port `8545`) by default via Docker.

To access your node's RPC endpoint, simply use:

```
http://<your-server-ip>:8545
```

To test it from your local machine, you can run:

```bash
curl -X POST http://<your-server-ip>:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

You should get a response with the latest block number like:

```json
{"jsonrpc":"2.0","id":1,"result":"0x123456"}
```

Replace `<your-server-ip>` with the actual IP address of your node server.



---

## 📬 Questions / Troubleshooting

- Join Base Discord for community support
- Recheck your volume mount paths and snapshot extraction paths
- Ensure Docker has access to enough CPU/IO/space

---

Enjoy your Base Full Node 🚀

