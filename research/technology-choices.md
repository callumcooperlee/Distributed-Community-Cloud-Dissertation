# Technology Choices: Cryptography and P2P Networking

Decisions for O2 (encrypted chunk storage) and O3 (peer discovery and transfer), made on 4 November 2026. Each decision is traced to the requirement it serves and the evidence behind it. Items marked *proposed* still need confirming.

## Summary

- **Language:** Go
- **Encryption:** XChaCha20-Poly1305 with a key-commitment check
- **Keys:** a random key per file, held only by the owner
- **Integrity:** chunks addressed by the SHA-256 hash of their ciphertext (*proposed*)
- **P2P stack:** go-libp2p
- **Discovery:** Kademlia DHT, plus mDNS on a local network
- **Membership:** libp2p private network with a pre-shared community key (*proposed*)
- **Connectivity:** emulated network for the evaluation; any member with a public address can act as a relay for real-device demos
- **Transfer:** a simple request-response protocol over libp2p streams (*proposed*)

## Language: Go

- **Decision:** write the node in Go.
- **Why:** go-libp2p is the implementation the main IPFS client runs on, so it is the most heavily used libp2p implementation (Trautwein et al. 2022). It includes every networking feature this design needs: the Kademlia DHT, mDNS discovery, private networks, AutoNAT, Circuit Relay v2 and hole punching (DCUtR). Go's extended crypto library includes XChaCha20-Poly1305.
- **Alternatives considered:** Rust (rust-libp2p is also mature, but slower to write); Python (py-libp2p is much less complete).

## Encryption

### Cipher: XChaCha20-Poly1305

- **Decision:** encrypt every chunk on the owner's device with XChaCha20-Poly1305, a standard AEAD cipher. This meets R-02 and the objectives doc's choice of standard cryptography over a custom design.
- **Why XChaCha20 rather than AES-GCM:** XChaCha20 uses a 192-bit nonce, so a fresh random nonce can be picked for every chunk without tracking counters. AES-GCM's 96-bit nonce limits how many messages one key can safely encrypt with random nonces (NIST SP 800-38D). Random nonces suit a system where many devices may write chunks independently.
- **Evidence it's sound:** ChaCha20-Poly1305 has a proven security bound, including when many users share the system (Degabriele et al. 2021).
- **Why standard cryptography at all:** four of five end-to-end encrypted cloud storage providers analysed had serious flaws in their own cryptographic designs (Hofmann & Truong 2024).

### Key commitment

- **Decision:** store a hash of the file key alongside the ciphertext and check it before decrypting.
- **Why:** standard AEAD ciphers, including ChaCha20-Poly1305 and AES-GCM, are not "key-committing". An attacker can craft one ciphertext that decrypts validly under two different keys (Albertini et al. 2022). This has been used in real attacks to recover passwords (Len et al. 2021). Albertini et al. propose adding an explicit key commitment like this one, which works with ChaCha20-Poly1305. It costs one hash per file.
- **Relevance:** the risk is low while only the owner holds a file's key. The check matters once files can be shared between members, so it is cheap insurance now.

### Keys: random per file

- **Decision:** generate a random key for each file. Keys never leave the owner's device (R-02).
- **Why not convergent encryption:** deriving the key from the file's hash lets identical files be stored once (Douceur et al. 2002). But it reveals which members store the same file, and lets anyone who can guess a file's contents confirm that someone stores it (Bellare et al. 2013). For personal data, privacy matters more than saving space.
- **Cost:** no deduplication between users. Each member's files are stored separately.

### Integrity and addressing (*proposed*)

- **Proposal:** address each chunk by the SHA-256 hash of its ciphertext, in the content-identifier format libp2p and IPFS already use.
- **Why:** a node can check a chunk it receives against its address before storing or returning it, so corrupted or tampered chunks are rejected (O2's 100% detection measure). The AEAD tag gives a second check on decryption.

## P2P networking

### Stack: go-libp2p

- **Decision:** build on go-libp2p rather than writing networking from scratch.
- **Why:** reusing a standard stack is a scoping decision in the objectives doc. go-libp2p is deployed across the IPFS network, with peers in over 2,700 autonomous systems in 152 countries (Trautwein et al. 2022).

### Discovery: Kademlia DHT

- **Decision:** find peers through libp2p's Kademlia DHT (R-01), and use mDNS to find peers on the same local network without any setup.
- **Why Kademlia:** it is the standard, well-analysed DHT (Maymounkov & Mazières 2002). Li et al. (2004) compared DHTs, including Kademlia, under churn, and found that parameter settings matter as much as the choice of protocol. Handling churn well also depends on how a DHT recovers from failures and sets timeouts (Rhea et al. 2004).
- **At community scale:** Kademlia keeps up to 20 contacts per routing-table bucket by default. With tens of nodes, every node will usually know every other node directly, so lookups are fast and the DHT mostly acts as a membership list. The DHT still matters because it removes any need for a central directory.
- **Churn data:** peer session lengths in deployed P2P networks are highly skewed, with many short sessions (Stutzbach & Rejaie 2006). This is the behaviour the churn experiments need to model.

### Membership: private network (*proposed*)

- **Proposal:** run the network as a libp2p private network, where every connection requires a pre-shared community key.
- **Why:** it limits the network to trusted members without any central server, which fits the trusted-user scope (R-05) and keeps strangers out of the DHT.
- **Open point:** how the key is shared and changed when someone leaves the community.

### Connectivity: emulation and member relays

- **Decision:** evaluate on an emulated network of containers, where every node can reach every other. For real-device demos, any member with a public address can act as a relay.
- **The problem:** most home devices sit behind NAT routers and can't accept incoming connections. libp2p's decentralised hole punching (DCUtR) gets around this, but only through a publicly reachable relay (Seemann et al. 2022).
- **Evidence:** across 4.4 million attempts on the IPFS network, about 29% failed before hole punching started (relay or address discovery failed). Of those that reached hole punching, 70% ± 7.1% succeeded, with TCP and QUIC performing the same (Trautwein et al. 2026). End-to-end, that suggests roughly half of connections between two NAT'd devices succeed directly. This figure is my estimate; the paper doesn't report it.
- **Why a relay isn't a coordinator:** relays only pass traffic between members who can't connect directly. They hold no file metadata, make no placement or repair decisions, and any member with a public address can take the role.
- **Limitation for the dissertation:** a community where no member has a public address can't connect its NAT'd devices directly. This goes in Limitations and Future Work.

### Transfer (*proposed*)

- **Proposal:** a simple request-response protocol over libp2p streams: store chunk, fetch chunk, delete chunk.
- **Why not IPFS's Bitswap:** Bitswap is designed to find which unknown peers hold content in an open network (Trautwein et al. 2022). Here, the placement logic already records which members hold each chunk, so a direct request is simpler and easier to measure for O3's throughput benchmarks.

## Requirements covered

- **R-01** discover nodes without a central server: Kademlia DHT, mDNS
- **R-02** encrypt files before they leave the device: XChaCha20-Poly1305, random per-file keys, key commitment
- **R-05** take part without tokens or payments: private network membership by community key
- **R-07** devices go offline freely: member relays, DHT churn handling
- **R-10** store and retrieve with one command: content-addressed chunks behind a simple transfer protocol

## Open questions

- Can relays be limited to members only, so the private network stays closed?
- How is the community key shared, and changed when a member leaves?
- What chunk size should be used? Smaller chunks spread data more evenly; larger chunks mean fewer DHT entries and less overhead.

## References

Full BibTeX entries are in [`Dissertation/dissertation.bib`](../Dissertation/dissertation.bib).

- Albertini et al. (2022). How to Abuse and Fix Authenticated Encryption Without Key Commitment. *31st USENIX Security Symposium*.
- Bellare, Keelveedhi & Ristenpart (2013). Message-Locked Encryption and Secure Deduplication. *EUROCRYPT 2013*. [doi:10.1007/978-3-642-38348-9_18](https://doi.org/10.1007/978-3-642-38348-9_18)
- Degabriele et al. (2021). The Security of ChaCha20-Poly1305 in the Multi-User Setting. *ACM CCS '21*. [doi:10.1145/3460120.3484814](https://doi.org/10.1145/3460120.3484814)
- Douceur et al. (2002). Reclaiming Space from Duplicate Files in a Serverless Distributed File System. *ICDCS 2002*. [doi:10.1109/ICDCS.2002.1022312](https://doi.org/10.1109/ICDCS.2002.1022312)
- Dworkin (2007). Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode (GCM) and GMAC. *NIST SP 800-38D*. [doi:10.6028/NIST.SP.800-38D](https://doi.org/10.6028/NIST.SP.800-38D)
- Hofmann & Truong (2024). End-to-End Encrypted Cloud Storage in the Wild: A Broken Ecosystem. *ACM CCS '24*. [doi:10.1145/3658644.3690309](https://doi.org/10.1145/3658644.3690309)
- Len, Grubbs & Ristenpart (2021). Partitioning Oracle Attacks. *30th USENIX Security Symposium*.
- Li et al. (2004). Comparing the Performance of Distributed Hash Tables Under Churn. *IPTPS 2004*. [doi:10.1007/978-3-540-30183-7_9](https://doi.org/10.1007/978-3-540-30183-7_9)
- Maymounkov & Mazières (2002). Kademlia: A Peer-to-Peer Information System Based on the XOR Metric. *IPTPS 2002*. [doi:10.1007/3-540-45748-8_5](https://doi.org/10.1007/3-540-45748-8_5)
- Rhea et al. (2004). Handling Churn in a DHT. *USENIX Annual Technical Conference 2004*.
- Seemann, Inden & Vyzovitis (2022). Decentralized Hole Punching. *ICDCS Workshops 2022*. [doi:10.1109/ICDCSW56584.2022.00027](https://doi.org/10.1109/ICDCSW56584.2022.00027)
- Stutzbach & Rejaie (2006). Understanding Churn in Peer-to-Peer Networks. *IMC '06*. [doi:10.1145/1177080.1177105](https://doi.org/10.1145/1177080.1177105)
- Trautwein et al. (2022). Design and Evaluation of IPFS: A Storage Layer for the Decentralized Web. *ACM SIGCOMM '22*. [doi:10.1145/3544216.3544232](https://doi.org/10.1145/3544216.3544232)
- Trautwein et al. (2026). Large-Scale Measurement of NAT Traversal for the Decentralized Web: A Case Study of DCUtR in IPFS. *ACM IMC 2026*. [doi:10.1145/3777912.3809138](https://doi.org/10.1145/3777912.3809138)
- go-libp2p. [github.com/libp2p/go-libp2p](https://github.com/libp2p/go-libp2p)
