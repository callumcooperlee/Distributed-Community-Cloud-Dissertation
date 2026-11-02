# Existing Storage Systems Review

Survey of existing storage systems for objective O1, written for the supervisor meeting on 2 November 2026. Tracks issue callumcooperlee/Distributed-Community-Cloud#21.

## Purpose

- No existing system stores **personal data durably on a trusted community's own devices without a central coordinator or a token economy**. Each solution gives up at least one of these. 
- Advantages and disadvantages of the systems are turned into user stories for the finished product.

## Existing systems

### IPFS and Filecoin

- **Strengths:** The most widely used P2P storage network, with peers in over 2,700 autonomous systems across 152 countries (Trautwein et al. 2022). Files are addressed by a hash of their content, so a downloaded file can be checked for tampering. Peers and content are found using a Kademlia DHT.
- **Weaknesses:** IPFS doesn’t guarantee storage. A file stays available only while at least one node “pins” it, and files aren’t encrypted (Daniel & Tschorsch 2022, Table III). Filecoin adds paid storage on top, but needs a blockchain, a token and storage proofs.
- **Relevance:** I can reuse libp2p and Kademlia, but I need my own replication and repair (R3, R5).
- **Churn:** Nothing replaces copies when nodes leave. If every node pinning a file goes offline, the file is unavailable (Daniel & Tschorsch 2022, §V-C).

### Storj

- **Strengths:** Splits files with Reed-Solomon erasure coding and stores extra pieces to cover nodes leaving and to leave room for repair. Nodes are audited and given a reputation score, which is used to choose where data goes (Daniel & Tschorsch 2022, §IV-D).
- **Weaknesses:** Relies on “satellite” servers that store file metadata, choose storage nodes and trigger repair. Data stored through one satellite can’t be reached through another, and a denial-of-service attack on a satellite has been demonstrated on a test network (Daniel & Tschorsch 2022, p. 42).
- **Relevance:** Choosing nodes by past reliability is worth keeping, but without a central satellite (R1, R6).
- **Churn:** Handled well, but centrally. Spare pieces absorb nodes leaving, and the satellite notices lost pieces and pays for repair (Daniel & Tschorsch 2022, §IV-D).

### Sia

- **Strengths:** Encrypts files on the client and uses erasure coding. Hosts sign storage contracts and put up collateral, so they lose money if they lose data (Vorick & Champine 2014).
- **Weaknesses:** Needs a blockchain and its own currency. The data owner remains “responsible for file health” (Daniel & Tschorsch 2022, p. 43), so their device has to come online regularly to check and repair files.
- **Relevance:** Repair should be done by the community, not left to the owner (R3, R5).
- **Churn:** Repair depends on the owner's device being online. If the owner is offline while hosts drop out, nobody repairs the file.

### Tahoe-LAFS

- **Strengths:** The closest existing design to mine. Storage servers are run by friends or an organisation, files are encrypted and erasure-coded on the client, and the "least authority" model means servers can't read or modify what they store (Wilcox-O'Hearn & Warner 2008).
- **Weaknesses:** New clients find storage servers through an "introducer", which the project's own documentation calls a single point of failure. Repair isn't automatic: someone has to run the checker and then the repairer. Files are split 3-of-10 by default (Tahoe-LAFS architecture docs).
- **Relevance:** Its security model is a good one to follow, but discovery should go through a DHT and repair should run automatically (R1, R2, R3, R7).
- **Churn:** The documentation says small private grids "should work well", but grids where nodes come and go often "may require more repair/verification bandwidth than actual upload/download traffic" (Tahoe-LAFS architecture docs). This is the cost my evaluation measures.

### Hyphanet (formerly Freenet)

- **Strengths:** Users volunteer disk space. Stored data is encrypted, so a host can't tell what it holds. There's no coordinator, and a "darknet" mode connects only to people you know (Clarke et al. 2001).
- **Weaknesses:** Each node's store works like a cache. When it fills up, data that hasn't been requested recently is deleted (Clarke et al. 2001). That's intended for censorship resistance, but it means there's no guarantee a personal file will survive.
- **Relevance:** Shows that coordinator-free storage can work, but only by giving up durability, which my project can't do (R3).
- **Churn:** Copies spread through caching as data is requested, so popular data survives nodes leaving. Unpopular data, like most personal files, does not.

### Hypercore (Dat)

- **Strengths:** No coordinator and no payments. Data is versioned and traffic is encrypted (Ogden 2017).
- **Weaknesses:** Replication is passive: data survives only while peers choose to keep hosting it (Daniel & Tschorsch 2022, p. 40).
- **Relevance:** Without incentives or active repair, nothing ensures data survives. My system needs active repair (R3).
- **Churn:** When the peers hosting a dataset go offline, it becomes unavailable, and nothing makes new copies.

### Syncthing

- **Strengths:** A widely used way to keep files in sync across your own devices without a central server, often adopted as a private alternative to Dropbox, OneDrive and Google Drive (Quinn et al. 2015).
- **Weaknesses:** Every device holds a full copy of every file, so storage isn't pooled. Encryption for devices you don't trust is a separate, optional feature (Syncthing documentation).
- **Relevance:** Sets the bar for how easy personal file storage should be, but my system needs to pool storage across a community rather than copy everything everywhere (R4).
- **Churn:** Copes by brute force: every device holds everything, so any one survivor keeps the data. Changes only sync when two devices are online at the same time.

### Friend-to-friend storage (F2F, Friendstore)

- **Strengths:** Picks storage partners from trusted users. This gives more stable partners and cheaper maintenance (Li & Dabek 2006). Friendstore limits how much each node stores to what it could re-upload with its own bandwidth if data had to be repaired (Tran et al. 2008).
- **Weaknesses:** These were prototypes and simulations from 2006–08, designed for desktop PCs. The only empirical follow-up (Sharma et al. 2011) used two weeks of online/offline logs from 848 users of an Italian instant-messaging service. It measured availability only, with no working system and no measurement of data loss or repair.
- **Relevance:** Trusted-community idea and the bandwidth limit on repair (R6, R9).
- **Churn:** Sharma et al. found about half of users could keep their data available at least 90% of the time by replicating to all their friends, and random placement came close to their best placement algorithm. No one measured what happens over months of churn with repair running.

## Commercial cloud: Google Drive, OneDrive, iCloud

Commercial services are the benchmark for availability and ease of use. They store data in always-on data centres, so they never face device churn. Where they fall short is control: the provider holds the keys, sets the terms, and is a single point of failure.

### Google Drive

- **Strengths:** Always available, seamless sync and a large free tier.
- **Weaknesses:** Google can read files by default. A sync bug in Drive for desktop v84 (November 2023) made some users' files revert by up to six months, and recovery needed a patched client (The Register 2023).

### OneDrive

- **Strengths:** Built into Windows and Office, so it works with no setup for most users.
- **Weaknesses:** Each file is encrypted with its own AES-256 key, but those keys are protected by master keys that Microsoft holds in Azure Key Vault, so Microsoft can decrypt files. There is no end-to-end option. Personal Vault adds stronger sign-in, not end-to-end encryption (Microsoft 2026).

### iCloud

- **Strengths:** End-to-end encryption is available through Advanced Data Protection (ADP).
- **Weaknesses:** Without ADP, iCloud Drive, Photos and Backup use keys that Apple holds. Apple withdrew ADP for UK users in February 2025 after the Home Office demanded access (Computer Weekly 2025; MacRumors 2025).

### Points that apply to all three

1. **Clients differ a lot in performance.** Benchmarks of Dropbox, Google Drive, SkyDrive (now OneDrive), Amazon Cloud Drive and Wuala found large differences in sync speed and bandwidth use (Drago et al. 2013; Bocchi et al. 2017).
2. **The terms of service decide who can access your data.** A survey of 35 consumer cloud services' terms found broad provider rights over content and accounts (Michels et al. 2019).
3. **Switching provider is hard.** Moving data and workflows between providers is costly, so users tend to stay even when they stop trusting them (Opara-Martins et al. 2016).
4. **End-to-end encryption is easy to get wrong.** Four of five end-to-end encrypted storage providers analysed had serious cryptographic flaws (Hofmann & Truong 2024). This is why I'm using standard AEAD encryption rather than designing my own.

**Requirements this gives:** encryption keys held only by the user (R2); storing and retrieving a file should be as simple as with a sync folder (R10); no single party whose failure or policy change loses data (R1, R3); owners can export and delete their data (R8).

**Sources still to improve:** the benchmark papers date from 2013–17. iCloud and Google Drive rely on news coverage, and Apple's Platform Security Guide would be a better primary source. I haven't confirmed which services Bocchi et al. (2017) tested.

## Research gaps

Daniel & Tschorsch (2022) list personal data and churn as open problems. Earlier F2F work looked at trusted storage, but nobody has built and tested a coordinator-free version under churn. That's the gap this project addresses.

### Behaviour under churn

- **Evidence it's open:** Churn resistance is listed as future work, with only one paper cited on it (Daniel & Tschorsch 2022, p. 48). Availability, scalability and changing membership work against each other (Blake & Rodrigues 2003).
- **What my project does:** Runs emulated churn experiments that measure how many files can still be recovered over time.

### Durability without a coordinator

- **Evidence it's open:** Active replication and erasure coding "require coordination", while passive replication relies on volunteers (Daniel & Tschorsch 2022, p. 46). Every durable system reviewed above uses either a coordinator or a currency.
- **What my project does:** Repairs data automatically through agreement between peers, with no satellite, introducer or blockchain.

### Personal rather than public data

- **Evidence it's open:** P2P data networks are "rather designed for public data than private data", and better confidentiality "can open these systems for storing private and personal data" (Daniel & Tschorsch 2022, pp. 48–49).
- **What my project does:** Stores personal files only, encrypted on the owner's device.

### Reliability-aware placement at community scale

- **Evidence it's open:** Placement based on node availability has been studied for large open networks and in simulation (Skowron & Rządca 2016; Huang et al. 2025), not for tens of trusted nodes.
- **What my project does:** Compares reliability-based and random placement on durability, storage overhead and repair traffic.

### Modern re-evaluation of F2F storage

- **Evidence it's open:** F2F and Friendstore were 2006–08 desktop prototypes (Li & Dabek 2006; Tran et al. 2008). Sharma et al. (2011) simulated availability from instant-messaging logs, but built no system, didn't measure durability or repair, and list "more extensive traces" as future work.
- **What my project does:** Builds a working libp2p implementation and evaluates it under emulated churn.

**Risk from Sharma et al. (2011):** under a limit on the number of copies, random placement came close to their best placement algorithm. Reliability-aware placement may therefore give only a small gain at community scale. That's a risk to the O5 hypothesis, and the evaluation has to report the result either way.

## Draft requirements as user stories

Twelve in-scope stories plus two marked Won't, each traced to a gap above or a lesson from an existing system. Priorities are a first pass for discussion.

### Must

- **R1:** As a community member, I want my node to find other nodes without any central server, so the network keeps working if anyone leaves.
	- *From:* Storj satellites, Tahoe-LAFS introducer
	- *Objective:* O3
- **R2:** As a data owner, I want files encrypted on my device before they leave it, so no host can read them.
	- *From:* Tahoe-LAFS; Hofmann & Truong 2024; commercial providers holding keys
	- *Objective:* O2
- **R3:** As a data owner, I want lost copies recreated automatically when a device drops out, so my files survive without me managing them.
	- *From:* IPFS, Hyphanet, Hypercore (no repair); Sia (owner repairs)
	- *Objective:* O5
- **R4:** As a community member, I want to contribute spare disk to a shared pool, so together we store more than any one device holds.
	- *From:* Syncthing (full copies only)
	- *Objective:* O2
- **R5:** As a community member, I want to take part without tokens or payments, so joining only requires being trusted.
	- *From:* Sia, Filecoin, Storj
	- *Objective:* scope
- **R7:** As a laptop or phone owner, I want my device to go offline whenever it needs to without putting anyone's data at risk.
	- *From:* the churn gap; Tahoe-LAFS assumes always-on servers
	- *Objective:* O6
- **R10:** As a data owner, I want to store and retrieve a file with one CLI command, so I can use the system without understanding how it works.
	- *From:* commercial cloud usability
	- *Objective:* O2

### Should

- **R6:** As a node, I want to place data on peers according to their observed reliability, so fewer copies are needed for the same durability.
	- *From:* Storj reputation; Li & Dabek 2006
	- *Objective:* O5, O6
- **R8:** As a data owner, I want to delete or export my data at any time, so I'm never locked in.
	- *From:* Daniel & Tschorsch 2022, p. 49 (deleting personal data); Opara-Martins et al. 2016
	- *Objective:* O2
- **R9:** As a host, I want repair traffic limited to what my upload can handle, so contributing doesn't slow my connection.
	- *From:* Friendstore; Chun et al. 2006
	- *Objective:* O5
- **R11:** As a community member, I want to see what each node has contributed and how reliable it's been, so contribution is fair and visible.
	- *From:* Volunteer Currency, limited to accounting
	- *Objective:* O4

### Could

- **R12:** As a data owner, I want erasure coding instead of full copies, so my data takes up less of the community's space.
	- *From:* Storj, Tahoe-LAFS
	- *Objective:* O5 stretch
	- *Note:* First to cut if January slips

### Won't (this time)

- **R13:** Protection against malicious or Sybil nodes.
	- *Reason:* Out of scope because users are trusted (agreed 5 Oct)
	- *Objective:* Future work
- **R14:** A graphical app.
	- *Reason:* Out of scope because a CLI is enough to demonstrate the product
	- *Objective:* Future work

## Questions

- [ ] Is reviewing these eight systems the right basis for O1, or should it cover fewer in more depth?
- [ ] Is "a modern, coordinator-free re-evaluation of F2F storage under churn" a strong enough contribution, given F2F and Friendstore exist and Sharma et al. (2011) only simulated availability?
- [ ] Is it acceptable to cite news coverage for current commercial encryption defaults (iCloud ADP in the UK, the Google Drive incident), or should I find primary sources?
- [ ] Are the Must stories (R1–R5, R7, R10) achievable in the hours left, with R12 as the first cut?
- [ ] Sharma et al. found random placement came close to their best algorithm. Should O5 still test reliability-aware against random placement, knowing the gain may be small?

## References

Full BibTeX entries are in [`Dissertation/dissertation.bib`](../Dissertation/dissertation.bib). Verify each on the Glasgow library or dblp before citing it in the dissertation.

- Blake & Rodrigues (2003). High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two. *Proceedings of the 9th Workshop on Hot Topics in Operating Systems (HotOS IX)*.
- Bocchi et al. (2017). Personal Cloud Storage Benchmarks and Comparison. *IEEE Transactions on Cloud Computing*. [doi:10.1109/TCC.2015.2427191](https://doi.org/10.1109/TCC.2015.2427191)
- Chun et al. (2006). Efficient Replica Maintenance for Distributed Storage Systems. *Proceedings of the 3rd USENIX Symposium on Networked Systems Design and Implementation (NSDI '06)*.
- Clarke et al. (2001). Freenet: A Distributed Anonymous Information Storage and Retrieval System. *Designing Privacy Enhancing Technologies*. [doi:10.1007/3-540-44702-4_4](https://doi.org/10.1007/3-540-44702-4_4)
- Computer Weekly (2025). Apple Withdraws Encrypted iCloud Storage from UK After Government Demands Back Door Access. *Accessed 2 November 2026*. [link](https://www.computerweekly.com/news/366619614/Apple-withdraws-encrypted-iCloud-storage-from-UK-after-government-demands-back-door-access)
- Daniel & Tschorsch (2022). IPFS and Friends: A Qualitative Comparison of Next Generation Peer-to-Peer Data Networks. *IEEE Communications Surveys & Tutorials*. [doi:10.1109/COMST.2022.3143147](https://doi.org/10.1109/COMST.2022.3143147)
- Drago et al. (2013). Benchmarking Personal Cloud Storage. *Proceedings of the 2013 Internet Measurement Conference (IMC '13)*. [doi:10.1145/2504730.2504762](https://doi.org/10.1145/2504730.2504762)
- Hofmann & Truong (2024). End-to-End Encrypted Cloud Storage in the Wild: A Broken Ecosystem. *Proceedings of the 2024 ACM SIGSAC Conference on Computer and Communications Security (CCS '24)*. [doi:10.1145/3658644.3690309](https://doi.org/10.1145/3658644.3690309)
- Huang et al. (2025). Improving Overall Data Availability in Decentralized Storage via an Availability-Balanced Replica Placement Strategy. *2025 IEEE 31st International Conference on Parallel and Distributed Systems (ICPADS)*. [doi:10.1109/ICPADS67057.2025.11323126](https://doi.org/10.1109/ICPADS67057.2025.11323126)
- Li & Dabek (2006). F2F: Reliable Storage in Open Networks. *Proceedings of the 5th International Workshop on Peer-to-Peer Systems (IPTPS '06)*.
- MacRumors (2025). Apple Pulls Encrypted iCloud Security Feature in UK Amid Government Backdoor Demands. *21 February 2025. Accessed 2 November 2026*. [link](https://www.macrumors.com/2025/02/21/apple-pulls-encrypted-icloud-security-feature-uk/)
- Marinos & Briscoe (2009). Community Cloud Computing. *Cloud Computing (CloudCom 2009)*. [doi:10.1007/978-3-642-10665-1_43](https://doi.org/10.1007/978-3-642-10665-1_43)
- Michels et al. (2019). Beyond the Clouds, Part 1: What Cloud Contracts Say About Who Owns and Can Access Your Content. *Queen Mary University of London, School of Law*.
- Microsoft (n.d.). How OneDrive Safeguards Your Data in the Cloud. *Accessed 2 November 2026*. [link](https://support.microsoft.com/onedrive/how-onedrive-safeguards-your-data-in-the-cloud)
- Ogden (2017). Dat -- Distributed Dataset Synchronization and Versioning. *OSF Preprints*. [doi:10.31219/osf.io/nsv2c](https://doi.org/10.31219/osf.io/nsv2c)
- Opara-Martins et al. (2016). Critical Analysis of Vendor Lock-in and Its Impact on Cloud Computing Migration: A Business Perspective. *Journal of Cloud Computing*. [doi:10.1186/s13677-016-0054-z](https://doi.org/10.1186/s13677-016-0054-z)
- Quinn et al. (2015). Forensic Analysis and Remote Evidence Recovery from Syncthing: An Open Source Decentralised File Synchronisation Utility. *Digital Forensics and Cyber Crime (ICDF2C 2015)*. [doi:10.1007/978-3-319-25512-5_7](https://doi.org/10.1007/978-3-319-25512-5_7)
- Sharma et al. (2011). An Empirical Study of Availability in Friend-to-Friend Storage Systems. *2011 IEEE International Conference on Peer-to-Peer Computing (P2P)*. [doi:10.1109/P2P.2011.6038754](https://doi.org/10.1109/P2P.2011.6038754)
- Skowron & Rz\kadca (2016). Flexible Replica Placement for Optimized P2P Backup on Heterogeneous, Unreliable Machines. *Concurrency and Computation: Practice and Experience*. [doi:10.1002/cpe.3491](https://doi.org/10.1002/cpe.3491)
- Syncthing Contributors (n.d.). Untrusted (Encrypted) Devices. *Syncthing documentation. Accessed 2 November 2026. Check whether the feature is still marked beta*. [link](https://docs.syncthing.net/users/untrusted.html)
- Tahoe-LAFS Developers (n.d.). Tahoe-LAFS Architecture. *Accessed 2 November 2026*. [link](https://github.com/tahoe-lafs/tahoe-lafs/blob/master/docs/architecture.rst)
- The Register (2023). Google Drive Missing Files Fix. *7 December 2023. Accessed 2 November 2026*. [link](https://www.theregister.com/2023/12/07/google_drive_missing_files_fix/)
- Tran et al. (2008). Friendstore: Cooperative Online Backup Using Trusted Nodes. *Proceedings of the 1st Workshop on Social Network Systems (SocialNets '08)*.
- Trautwein et al. (2022). Design and Evaluation of IPFS: A Storage Layer for the Decentralized Web. *Proceedings of the ACM SIGCOMM 2022 Conference*. [doi:10.1145/3544216.3544232](https://doi.org/10.1145/3544216.3544232)
- Vorick & Champine (2014). Sia: Simple Decentralized Storage. *Whitepaper, Nebulous Inc*.
- Wilcox-O'Hearn & Warner (2008). Tahoe: The Least-Authority Filesystem. *Proceedings of the 4th ACM International Workshop on Storage Security and Survivability (StorageSS '08)*. [doi:10.1145/1456469.1456474](https://doi.org/10.1145/1456469.1456474)
