# Challenges Faced During Task

The most significant challenge was ensuring the StarkNet node was fully and correctly synchronized with the Sepolia testnet within a reasonable timeframe. This single requirement led to several cascading issues:

1.  **Impracticality of Syncing from Genesis:** A primary challenge was the sheer time required to sync a new node from block 0. Initial attempts with the Juno node showed a sync time of over 12–24 hours, which was not feasible given the project deadline. This made using a pre-synced database snapshot an absolute necessity.

2.  **Snapshot Availability and Integrity:** While snapshots are the solution to slow syncing, they introduced their own set of problems:
    * **Broken Links:** Official snapshot URLs were frequently outdated, leading to `404 Not Found` errors and wasted time searching for reliable, up-to-date sources.
    * **Node Health:** The initial Pathfinder node, even when started from a snapshot, was unhealthy. It reported `ContractNotFound` errors because its local database was not correctly synced, leading to a significant amount of time spent troubleshooting the wrong problem (i.e., suspecting incorrect contract addresses). This forced a switch to the Juno node client.

3.  **Server Environment and Large File Handling:** The large size of the snapshots (150GB+ compressed, ~500GB uncompressed) created further obstacles:
    * **DNS Failures:** The server initially failed to download the snapshot due to a DNS issue (`unable to resolve host address`), which required manual reconfiguration of the server's network settings.
    * **Resource Demands:** The large download and extraction process confirmed that a server with both a high-speed network connection and substantial, fast SSD storage is critical for a timely setup.

In summary, the core difficulty was not the sync process itself, but the dependency on large, third-party snapshots. The unreliability of snapshot sources and the underlying health of the node data proved to be the most complex and time-consuming part of the entire validator setup.
