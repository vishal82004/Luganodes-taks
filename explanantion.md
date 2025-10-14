# StarkNet Validator Setup Guide (concise)

This guide walks you through deploying a StarkNet Sepolia testnet validator: server prep, account setup, Juno node, staking, attestation service, and a public RPC proxy.

---

## 1. Environment Preparation
- VPS: Ubuntu 22.04, >=4GB RAM, 2 CPU, 100GB SSD.
- Install Docker:
```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
sudo usermod -aG docker ${USER}  
```
- Install starkli:
```bash
curl https://get.starkli.sh | sh
starkliup
source ~/.bashrc
```

---

## 2. Accounts & Funding
Create three accounts (Braavos / Argent X):
- Staking Account (holds STRK)
- Operator Account (signing)
- Rewards Account
Fund with Sepolia ETH for gas and at least 27 STRK to the Staking Account.

---

## 3. Server Configuration
Export addresses / keys (temporary environment variables):
```bash
export STAKING_ADDRESS="0x032c979ea0468c2fbf03f50629dde3acf40453c5af3232108c3a296068afa8e2"
export OPERATIONAL_ADDRESS="0x022c6f5787b91332fcbc44e42210ff992aa81ff497c0ffe2574f4ce94c4b3cb8"
export REWARDS_ADDRESS="0x052d0786ed8ea0963c5fe5c6cc687ecb695d266a4c0bbf6b00335c71f73f14fe"
export OPERATIONAL_PRIVATE_KEY="<OPERATOR_PRIVATE_KEY>"
```
Prepare starkli:
```bash
mkdir -p ~/.starkli/accounts/ ~/.starkli/keys/
starkli account fetch --output ~/.starkli/accounts/staker.json $STAKING_ADDRESS
starkli signer keystore from-key ~/.starkli/keys/staker.json
# follow prompts to encrypt keystore
```

---

## 4. Juno Node (fast sync via snapshot)
Set ETH websocket URL:
```bash
export ETHEREUM_URL="<YOUR_ETHEREUM_SEPOLIA_WEBSOCKET_URL>"
```
Download & extract snapshot (requires lz4):
```bash
sudo apt-get update && sudo apt-get install -y lz4
mkdir -p $HOME/juno && cd $HOME/juno
wget -O latest.tar.lz4 "https://snapshots.gizatech.xyz/juno/sepolia/latest.tar.lz4"
tar -I lz4 -xf latest.tar.lz4
```
Run Juno in Docker:
```bash
docker run -d \
  --name juno \
  --restart unless-stopped \
  -p 6060:6060 \
  -v $HOME/juno:/var/lib/juno \
  nethermind/juno:latest \
  --network sepolia \
  --eth-node $ETHEREUM_URL
```
Verify logs:
```bash
docker logs -f juno
```

---

## 5. Staking (example commands)
Approve STRK then stake (replace contract addresses and amounts as needed):
```bash
starkli invoke --account ~/.starkli/accounts/staker.json --keystore ~/.starkli/keys/staker.json --network sepolia \
  0x04718f5a0fc34cc1af16a1cdee98ffb20c31f5cd61d6ab07201858f4287c938d approve 0x03745ab04a431fc02871a139be6b93d9260b0ff3e779ad9c8b377183b23109f1 27000000000000000000 0

starkli invoke --account ~/.starkli/accounts/staker.json --keystore ~/.starkli/keys/staker.json --network sepolia \
  0x03745ab04a431fc02871a139be6b93d9260b0ff3e779ad9c8b377183b23109f1 stake 0x052d0786ed8ea0963c5fe5c6cc687ecb695d266a4c0bbf6b00335c71f73f14fe 0x022c6f5787b91332fcbc44e42210ff992aa81ff497c0ffe2574f4ce94c4b3cb8 27000000000000000000

# Set 13% commission (1300 / 10000)
starkli invoke --account ~/.starkli/accounts/staker.json --keystore ~/.starkli/keys/staker.json --network sepolia \
  0x03745ab04a431fc02871a139be6b93d9260b0ff3e779ad9c8b377183b23109f1 set_commission 1300
```

---

## 6. Attestation Service
Run attestation, pointing to your Juno node (use internal IP or localhost if running on same host):
```bash
docker run -it --rm --network host \
  -e VALIDATOR_ATTESTATION_OPERATIONAL_PRIVATE_KEY=$OPERATIONAL_PRIVATE_KEY \
  ghcr.io/eqlabs/starknet-validator-attestation \
  --staking-contract-address 0x03745ab04a431fc02871a139be6b93d9260b0ff3e779ad9c8b377183b23109f1 \
  --attestation-contract-address 0x03f32e152b9637c31bfcf73e434f78591067a01ba070505ff6ee195642c9acfb \
  --staker-operational-address 0x022c6f5787b91332fcbc44e42210ff992aa81ff497c0ffe2574f4ce94c4b3cb8 \
  --node-url http://127.0.0.1:6060 \
  --local-signer
```

---

## 7. Public RPC (Nginx) — example
Install nginx and add a site:
```nginx
server {
    listen 80;
    server_name 65.0.3.77;

    location /rpc {
        proxy_pass http://127.0.0.1:6060;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/starknet /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
# Public RPC: http://65.0.3.77/rpc
```

---

## 8. Monitoring & Deliverables (brief)
- Optional: Uptime Kuma monitor (JSON query) to check syncing via /rpc.
- Consider Prometheus/Grafana (scrape Juno metrics) for dashboards.
- Deliverables checklist: addresses, transaction hash, public RPC, uptime screenshots, Grafana screenshot, attestation logs, brief notes on issues encountered.


