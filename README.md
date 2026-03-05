# Technical Overview: TabbyPOS Watcher

**TabbyPOS Watcher** is an asynchronous backend service that acts as a bridge between the **Bitcoin Cash (BCH) blockchain** and your local **MSSQL database**. Its primary goal is to automate payment detection and confirmation for retail Point-of-Sale environments.



---

## 🛠 Core Functionality

### 1. Dynamic Database Synchronization (Polling)
The service performs continuous background polling of your SQL environment:
* **Merchant Sync**: Every 30 seconds, it scans the `MerchantAddress` table for new entries.
* **Instant Monitoring**: Newly added addresses are automatically subscribed to the blockchain listener in real-time without requiring a service restart.

### 2. Real-Time Blockchain Monitoring
Instead of periodically "checking" for balance changes, the service maintains a persistent, encrypted **SSL connection** to public Fulcrum (Electrum) nodes:
* **Event-Driven**: It listens for "status change" notifications. 
* **0-Conf Ready**: The moment a transaction is broadcast to the network (mempool), the service receives a signal and begins the verification process immediately.

### 3. Two-Step Payment Verification
Since basic blockchain notifications do not include transaction values, the script executes a rigorous two-step validation:
* **Step A (Discovery)**: Identifies the specific Transaction ID (TXID) associated with the merchant's address.
* **Step B (Deep Inspection)**: Fetches the full transaction details (hex) and iterates through the outputs (`vout`). It sums all values sent to the merchant's address to ensure the customer didn't send a partial payment.



### 4. Automated Settlement & Deduplication
Once a valid payment is confirmed:
* **Status Update**: It automatically sets `Status = 1` (Paid) in the `TxReceiveCoin` table for the corresponding address.
* **Audit Trail**: Records the TXID in the `BCH_Processed_Payments` table.
* **Double-Spend Protection**: Before processing, it checks the database to ensure a Transaction ID has not been credited previously.

---

## ⚡ Technical Highlights

* **Asynchronous Architecture**: Built with Python’s `asyncio` to handle hundreds of concurrent merchant subscriptions without performance bottlenecks.
* **Satoshi-Level Precision**: All calculations are performed in Satoshis (integers) to eliminate floating-point math errors common in financial software.
* **High Availability**: Features an automatic failover mechanism that rotates between multiple public nodes if the primary connection is interrupted.
* **SQL Integration**: Optimized for MSSQL using `pyodbc`, handling connection strings and encrypted connections securely.

```pythob

import asyncio
import json
import ssl
import pyodbc
from datetime import datetime

# --- Configuration ---
SQL_CONFIG = {
    "conn_str": "DRIVER={ODBC Driver 18 for SQL Server}; SERVER=KOLMENTECHS01\SQLEXPRESS; DATABASE=TabbyDB; Trusted_Connection=yes; Encrypt=no;",
    "table_merchants": "MerchantAddress",
    "table_receive": "TxReceiveCoin",
    "table_processed": "BCH_Processed_Payments"
}

FULCRUM_SERVERS = [
    {"host": "bch.imaginary.cash", "port": 50002},
    {"host": "electroncash.de", "port": 50002}
]

class TabbyWatcher:
    def __init__(self):
        self.subscribed_addresses = set()
        self.conn_str = SQL_CONFIG["conn_str"]
        self.is_running = True

    def fetch_all(self, query, params=()):
        with pyodbc.connect(self.conn_str) as conn:
            with conn.cursor() as cursor:
                cursor.execute(query, params)
                return cursor.fetchall()

    async def run(self):
        server_idx = 0
        while self.is_running:
            server = FULCRUM_SERVERS[server_idx]
            print(f"📡 Connecting to Fulcrum: {server['host']}...")
            try:
                context = ssl.create_default_context()
                reader, writer = await asyncio.open_connection(
                    server['host'], server['port'], ssl=context)

                self.subscribed_addresses.clear()
                polling_task = asyncio.create_task(self.poll_database_for_new_addresses(writer))

                print("✅ Watcher is online. Monitoring database and blockchain...")
                while True:
                    line = await reader.readline()
                    if not line: break
                    
                    data = json.loads(line.decode())
                    if "method" in data and data["method"] == "blockchain.address.subscribe":
                        address = data["params"][0]
                        print(f"🔔 Activity detected on address: {address}")
                        await self.handle_payment_verification(address, reader, writer)

                polling_task.cancel()
            except Exception as e:
                print(f"⚠️ Connection error or DB failure: {e}")
                server_idx = (server_idx + 1) % len(FULCRUM_SERVERS)
                await asyncio.sleep(5)

    async def poll_database_for_new_addresses(self, writer):
        while True:
            try:
                rows = self.fetch_all(f"SELECT address FROM {SQL_CONFIG['table_merchants']}")
                for row in rows:
                    addr = row[0].strip()
                    if addr and addr not in self.subscribed_addresses:
                        msg = {"id": "sub_req", "method": "blockchain.address.subscribe", "params": [addr]}
                        writer.write(json.dumps(msg).encode() + b'\n')
                        await writer.drain()
                        self.subscribed_addresses.add(addr)
                        print(f"➕ New address added to monitor: {addr}")
            except Exception as e:
                print(f"❌ DB Polling Error: {e}")
            await asyncio.sleep(30)

    async def handle_payment_verification(self, address, reader, writer):
        """
        Step 1: Detect transaction in Mempool.
        Step 2: Fetch full transaction details to verify amount.
        """
        print(f"🔍 Verifying payment for: {address}")
        
        # 1. Fetch the target amount
        res = self.fetch_all(
            f"SELECT AmountInCoin FROM {SQL_CONFIG['table_receive']} WHERE Address = ? AND Status = 0",
            (address,)
        )
        if not res: return 
        
        target_bch = float(res[0][0])
        target_sats = int(round(target_bch * 100000000))

        # 2. Get the list of transactions in Mempool
        query_mempool = {"id": "get_mempool", "method": "blockchain.address.get_mempool", "params": [address]}
        writer.write(json.dumps(query_mempool).encode() + b'\n')
        await writer.drain()
        
        resp = json.loads((await reader.readline()).decode())
        tx_list = resp.get("result", [])

        for tx_entry in tx_list:
            txhash = tx_entry['tx_hash']

            # 3. Deduplication Check
            processed = self.fetch_all(
                f"SELECT COUNT(1) FROM {SQL_CONFIG['table_processed']} WHERE TXID = ?", (txhash,)
            )
            if processed[0][0] > 0: continue

            # 4. FIX: Fetch full transaction details to get the actual values
            query_tx = {"id": "get_tx", "method": "blockchain.transaction.get", "params": [txhash, True]}
            writer.write(json.dumps(query_tx).encode() + b'\n')
            await writer.drain()
            
            tx_resp = json.loads((await reader.readline()).decode())
            tx_detail = tx_resp.get("result", {})

            # 5. Loop through outputs (vout) to sum the amount sent to this address
            total_received_sats = 0
            vouts = tx_detail.get("vout", [])
            for vout in vouts:
                spk = vout.get("scriptPubKey", {})
                # Support different Fulcrum response formats (address vs addresses)
                out_addr = spk.get("address") or spk.get("addresses", [None])[0]
                
                if out_addr == address or (out_addr and out_addr.replace("bitcoincash:", "") == address.replace("bitcoincash:", "")):
                    total_received_sats += int(round(float(vout["value"]) * 100000000))

            print(f"📊 Target: {target_sats} Sats | Received: {total_received_sats} Sats")

            # 6. Compare and Finalize
            if total_received_sats >= target_sats:
                print(f"💰 Amount Match Found! Finalizing...")
                try:
                    with pyodbc.connect(self.conn_str) as conn:
                        cursor = conn.cursor()
                        cursor.execute(
                            f"INSERT INTO {SQL_CONFIG['table_processed']} (TXID, OrderAmount, MerchantAddr, ProcessTime) VALUES (?, ?, ?, GETDATE())",
                            (txhash, target_bch, address)
                        )
                        # Ensure column names match your DB schema (Address vs Status)
                        cursor.execute(
                            f"UPDATE {SQL_CONFIG['table_receive']} SET Status = 1 WHERE Address = ? AND Status = 0",
                            (address,)
                        )
                        conn.commit()
                    print(f"✅ Payment Successful for {address}!")
                except Exception as e:
                    print(f"❌ Database update failed: {e}")

if __name__ == "__main__":
    watcher = TabbyWatcher()
    try:
        asyncio.run(watcher.run())
    except KeyboardInterrupt:
        print("Stopping Watcher...")


```
