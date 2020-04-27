# Persistent_Memory

## Introduction

Application performance and security are undeniable parts of high-performance systems in datacenters. Hence, there are numerous number of studies in various layers of these systems that some of them are in infancy phase \textcolor{red}{\cite{}} and rest of them are commercialized \textcolor{red}{\cite{}}. In addition, distributed high-performance systems such as database systems or key-value stores which are widely used in production datacenters, need to be improved in various aspects \textcolor{red}{\cite{}}. For example, there should be quite huge, byte-addressable, persistence and low cost memory systems in high-performance datacenters for applications like distributed databases. Also, there should be a secure, reliable and trusted mechanism to completely isolate applications' code and data from each other to prevent the occurrence possible attacks. According to Recent researches in \cite{intelsgx,empirical}, Intel proposed two products to address above mentioned issues: Intel Optane Persistent Memory (PMEM) and Intel Software Guard Extension (SGX). Intel Optane Persistent Memory (PMEM) promises byte-addressability, persistence, high capacity (128/256/512GB per DIMM), lower latency in read/write, lower cost than DRAMs and high performance, all on memory bus \cite{enclavedb, occlum}. Intel SGX enables user-level code to create private memory regions called enclaves, which code and data are protected by the CPU from any software and hardware attacks outside the enclaves. In fact, SGX provides a practical solution to the problem of secure computation on untrusted platforms such as multi tenant datacenters and public clouds \cite{occlum}.

Furthermore, with the emergence of Remote Persistent Memory (RPM), the developers have been studied to implement high-performance distributed systems on RPM with specific remote memory access, called Remote Direct Memory Access (RDMA). RDMA-based distributed systems use specific kinds of NICs, called RNIC which eliminates TCP/IP protocol stack for network connectivity. Therefore, it is not possible to implement high-performance distributed systems over commodity hardware. The problem also gets even more complex when these systems need strong security mechanisms to have confidentiality, integrity and freshness in untrusted environments. Some researches such as \cite{eRPC} and \cite{dspm} proposed mechanisms for distributed systems to access resources remotely with latency comparable to specialized hardware like RDMA. Yet, there are several works need to be added in these mechanisms such as security to prepare the whole system operable in un-trustable environments. As a result, there is a fundamental need to provide an infrastructure for developers to implement a secure and distributed high performance system over commodity hardware inside datacenters. So, the ultimate goal in proposed research is to design and build COSMOS, a Se\underline{c}ure Rem\underline{o}te Per\underline{s}istent Me\underline{mo}ry Ab\underline{s}traction, which can be used to build different set of high-performance and secure applications such as KV stores/databases accessible over the traditional TCP/IP network inside datacenters. To this end, the entire system is divided into three parts: Persistent Memory, Networking infrastructure which leverages the concept and implementation of Extended Remote Procedure Call (eRPC) and security which is provided by Intel SGX.

## Background

In this section, each part of COSMOS which is mentioned in the previous section, will be elaborated in greater detail.

### Persistent Memory

Persistent Memory or PMEM is a type of Non-Volatile memory (NVMem) which is directly connected to memory bus rather than PCI-e. PMEMs provide byte-addressability, persistence, and latency which is within an order of magnitude of DRAM\cite{dspm, empirical,snia}.As shown in Figure\ref{figure:hierarchy}, PMEMs provide a new entry in the memory-storage hierarchy which fills the perfromance/capcacity gap\cite{pmem.io}.

![picture](data/memorypyramid.png)

With PMEMs, applications have a new tier available for data placement. In addition to the memory and storage tiers, the persistent memory tier offers greater capacity than DRAM and significantly faster performance than storage. Applications can access persistent memory like they do with traditional memory, eliminating the need to page blocks of data back and forth between memory and storage.
PMEMs are built to work in two modes: Memory Mode where it acts exactly like DRAMs, and App Direct Mode where it plays the role of persistent memory. In App Direct Mode, software has a byte-addressable way to talk to the persistent memory capacity. In-memory databases restart time can also be significantly reduced because applications no longer have to reload data from storage to main memory, as they can access the persistent memory address space by using load/store accesses.

With the emergence of Intel's Optane Dual Inline Memory Module (DIMM) which is the most recent commercially available PMEM on the market, researchers in \cite{empirical} explored properties and characteristics of the product at different levels. Optane memory has several features as below:
\begin{itemize}
\item Optane DIMM has higher density compared to DRAM, therefore, it is available in bigger capacities like 128, 256 and 512 BG per DIMM.
\item Optane DIMMs like traditional DRAM DIMMs sit on the memory bus, and connects to the processor's Integrated Memory Controller (iMC). iMC is responsible for maintaining the Optane DIMMs and DRAMs' functionalities.
\item Optane DIMMs operate in two modes: Memory, which CPU and OS simply see the DIMMs as a larger Volatile portion of main memory, and Direct mode, which behaves as a persistent memory.
\item Optane DIMM is both persistent and byte-addressable. It means that it can fill the role of either a main memory or a persistent device(e.g. SSD).
\end{itemize}
Applications access the Optane DIMM's content using store instructions which are the extended instruction Set Architecture (ISA), and those stores will become persistent.
PMEMs also introduce several new programming challenges such as:
\begin{itemize}
\item CPUs have out-of-order CPU execution and cache access/flushing.  This means if two values are stored by the application, the order in which they become persistent may not be the order that the application wrote them.
\item If power outage happens during an aligned 8 bytes store to PMEM, either the old 8 bytes or the new one will be found in that location after reboot. In other words, 8 bytes stores are powerfail atomic.
\item As anything larger than 8 bytes is not powerfail atomic, it is up to the software to implement whatever Transactions are required for consistency.
\item Memory leaks to persistent storage are persistent.  Rebooting the server doesn't change the on-device contents.
\item Since applications have direct access to the persistent memory media, any errors will be returned back to the application as memory errors. So, applications may need to detect and handle hardware errors directly.
\end{itemize}

According to Figure \ref{figure:hierarchy}, To get the low latency direct access, a new software architecture is required that allows applications to access ranges of persistent memory. The Persistent Memory Development Kit (PMDK) is a collection of libraries and tools for System Administrators and Application Developers to simplify managing and accessing persistent memory devices. The libraries are built on Direct Access (DAX) feature which allows applications to access persistent memory directly as memory-mapped file\cite{snia}. Support for this feature is what differentiates a normal file system from a persistent memory-aware file system. Figure \ref{figure:model} shows the model which describes how applications can access persistent memory devices (NVDIMMs) using traditional POSIX standard APIs such as read, write, pread, and pwrite, or load/store operations such as memcpy when the data is memory mapped to the application. For persistent memory, the APIs for memory mapping files are at the heart of the persistent memory programming model published by \cite{snia}. The 'Persistent Memory' area describes the fastest possible access because the application I/O bypasses existing filesystem page caches and goes directly to/from the persistent memory media.

![picture](data/pmdkdiagram.png)

Unfortunately, most persistent memories are designed for the single-node environment. Also in an empirical observation \cite{empirical} it is mentioned that the it is unclear scalable persistent memories will evolve. So, with modern datacenters applicatoins' computation scale, we have to be able to scale out persistent memory systems\cite{dspm} and hide background complexities from application developers by providing suitable abstractions.

### High-performance Networking
Modern datacenters are very fast, as we see 100Gbps datacenters with latency of 2microseconds between two hosts connected to the same switch, and switches add around 300nanosecond per hop. The problem in high-performance datacenter network is that existing networking options are trying to either sacrifice performance or generality. On the slow hand, we have TCP, gRPC, etc. which work in commodity datacenters and provide features needed to build applications like reliability and congestion control, but they are slow. On the other hand, there are fast but specialized options like DPDK and RDMA which make simplifying assumptions, while requires special hardware. One of the drawback in this field is limited applicability, which means specialized hardware is needed to run them. But more fundamental drawback is that these systems co-design the application logic with the networking, and as a result, they lag modular networking abstraction which prevents reuse. These specialized technologies were deployed with the belief that placing their functionality in the network will yield a large performance gain. But in \cite{eRPC} it is shown that a general purpose RPC can provide state-of-the-art performance on commodity datacenter networks without additional networks support. Generally, Remote Procedure Call (RPC) is not technically a protocol, but it is better thought of as a general mechanism for structuring distributed systems. RPC is popular because it is based on the semantics of a local procedure call. An application developer can be largely unaware of whether the procedure is local or remote.

![picture](data/rpc.png)

In spite of the fact that RPC brings transparency to the applications, it has several disadvantages:
\being{itemize}
\item RPC Passes Parameters by values only and pointer values are not allowed.
\item This mechanism is highly vulnerable to failure as it involves a communication system, another machine, and another process.
\item RPC concept can be implemented in different ways, implying that there is no specific standard.
\item RPC increases the cost of the process.
\end{itemize}

eRPC is a fast and general remote procedure call for datacenters. The reason for using eRPC for communication is to implement high-performance system with minimum alteration in hardware. For example, in order to have performance, RDMA is used instead of traditional TCP/IP stack. eRPC claims that it can propose system performance comparable to RDMA-based systems only by using traditional system softwares even in lossy environment such as ethernet. eRPC is optimized for common-case scenarios, so it is considered a general purpose communication system. As a result, researchers in \cite{eRPC} believe it can also be optimized for special use cases. eRPC implements RPCs on top of Transport Layer that provides basic unreliable packet I/O, such as UDP.
eRPC breaks away from this performance generality trade-off by providing both, that is **do not give up generality for high performance**.
![picture](data/erpc_general.png)
 To do so, \cite{eRPC} have to provide functionality equivalent to hardware solutions in software. For example, if the network is lossy, there should be mechanisms to prevent packet loss, also end-to-end reliability and congestion control with low overhead, all in the software and without loosing performance.

One problem in managing packet loss is **large timeouts**. When the drain rate is smaller than receiving rate, there will be packet loss. When this happens, the switch buffer starts filling up and when it is full, it starts dropping packets. Because the sender get no feedback from the network, it does not know when it is safe to retransmit, so they use conservative approach for re-transmission which is in the order of miliseconds. Large timeouts are also bad because they increase latency. For example, we're building a distributed system where clients hold locks on a remote object, if a client's unlock packets get dropped, the locked object still is locked for several miliseconds. As a result, contending requests from other clients will be failed, which leads to low performance. The prior solution for this problem was lossless link layer(PFC, Infiniband). In a lossless datacenter, before switch buffer fills up, the switch sends a feedback to the sender in a form of pause message, telling it to stop. So, in lossless datacenters, there is no need for retransmission.

The approach which eRPC takes to solve abovementioned problems is that the researchers made an observation about normal datacenter networks that allows a simple method of preventing packet loss most of the time. Note that, there is no need to completely eliminate the packet loss. That is an extreme requirement which comes with drawbacks. what is needed for packet loss is to be rare enough not to affect the system performance, and this contribution has found that with a little help from software, existing network can satisfy this requirement.

There is a parameter which is called **Bandwidth Delay Product (BDP)** denotes the maximum size of sliding window to reach peak rate with available bandwidth:
```
BDP = Bandwidth * RTT
```
giving this information, in contrast, in datacenter switches, there are around 12MB of buffer, which is 3 orders of magnitude larger than BDP\cite{eRPC}. So, if software is limited to the amount of BDP, then switch can buffer hundreds of loads and prevents packet loss. So, this observation lets researchers to work on low latency NICs. eRPC by limiting the window size to 19KB with 12MB of intermediate switch buffer, can achieve 640 nodes in many to one traffic pattern.

Many transport layer components like end-2-end reliability, congestion control and memory management are quite expensive, but researchers have found that most of the overhead can be avoided in the common case\cite{eRPC}. in eRPC, it is assumed that the datacenter network is uncongested, therefore, eRPC implementation is optimized for uncongested networks. There, it uses Timely algorithm to control congestion that is, if RTT is high, then TX_Rate will decrease and vice versa.

### Security

The recent advent of hardware-based trusted execution environments (TEE) provides the isolated execution even from an untrusted OS and hardware-based attacks\cite{shieldstore}. Intel Software Guard Extension (SGX) is Intel's product which provides security and protection by creating an \textit{enclave} in applications's virtual address space. Once an enclave has been initialized, code and data within the enclave is isolated from the rest of the system including privileged software. Generally, there are three important features which collectively guarantee the security of a system:
\begin{itemize}
    \item \textit{confidentiality}: A feature which guarantees receiver that the received data was not recognizable by any third entity. This can be implemented by encrypting the data.
    \item \textit{Integrity}: A feature which guarantees the receiver that the received data is remained intact. This can be implemented by putting data hash near the data.
    \item \textit{Freshness}: A feature which guarantees the receiver that the received data is the latest and the newest one.
\end{itemize}
 Freshness guarantees are typically built on top of a system that already offers integrity guarantees, by adding a unique piece of information to each message. A popular solution for gaining freshness guarantees relies on nonces, single-use random numbers. Nonces are attractive because the sender does not need to maintain any state; the receiver, however, must store the nonces of all received messages.

\begin{figure}[htp]
    \centering
    \includegraphics[width=\linewidth, height = 6cm]{enclave.png}
    \caption{enclave}
    \label{figure:genperf}
\end{figure}

To understand how SGX works, some portion of the RAM is marked as PRM so this is known as a Processor Related Memory which are marked by BIOS. SGX creates Enclave Page Cache (EPC) that stores recently accessed sensitive pages. This memory area is encrypted using the Memory Encryption Engine (MEE), a new and dedicated chip near CPU. So, what this means is that the OS, any external device, or any other applications running over the processor would not be able to access the memory in the PRM. As a result, there is a region of a memory which resembles a hole inside the memory which is completely in the control of the hardware. So, the hardware could use this region of memory to create enclaves. So internally PRM is divided into several EPCs of size 4KB. EPCs are used to actually store the enclaves of the various applications present in the program. There is also some management related information which is stored in a special region known as the EPCM. While SGX provides data confidentiality and integrity by a feature called sealing, it does not provides data freshness. Generally, confidentiality is preserved by encrypting all signals that emerge from the processor. Integrity is a property that the memory that the memory system correctly returns the last-written block of data at any address\cite{intelsgxexplained}.

\begin{figure}[htp]
    \centering
    \includegraphics[width=\linewidth]{PRM.png}
    \caption{Data Structures used by SGX}
    \label{figure:DS_SGX}
\end{figure}

According Figure \ref{figure:DS_SGX}, there are multiple processes present in the system and each process has its own virtual address space and within this virtual address space each process could have its own enclave. Therefore we would now have a mapping for enclave region within the virtual address space to PRM region in the RAM. So, we could have multiple processes, and each of them having their own enclaves but all of enclaves associated with their own processes would mapped to the same region within the RAM that is the PRM region. EPCM is divided into various sub-regions and we have one entry for each EPC. For example, if there are 1024 EPC regions each of 4KB, then there would be 1024 regions in the EPCM, one region is associated with a corresponding EPC. EPCM is used by the hardware for access control. It stores various information related to the corresponding EPC, so EPCM is somewhat similar to the page tables, but difference is that the most of the page tables are managed by the OS while with the SGX the EPCM contents are completely managed by the hardware. There is another data structure inside PRM which is called SECS (SGX Enclave Control Store) which contains global information and metadata about that particular enclave. It is used for data integrity inside enclaves and also for mapping information to the various enclave regions.

An application which needs secure environment in SGX-enabled CPUs, should be divided into two parts: A secure part which is launched inside enclave and non-secure part which resides out of the enclave. Enclave code and data are placed and encrypted in EPC. And pages are only decrypted when they are inside the physical processor core. Keys are generated at boot-time and are stored within the CPU. So, when data or code from the enclave is going in or out of the processor, no attacker could actually identify the actual data or code by monitoring or snooping the DRAM bus

SGX extension provides hardware-based isolation and memory encryption to provide more code protection for developers. In other words, the security model of Intel SGX states that only the CPU needs to be trusted and all software including applications and the OS is considered untrusted. SGX-enabled processors offer two crucial properties:
\begin{itemize}
    \item Isolation which means each enclave's environment is isolated from the untrusted software outside of the enclave.
    \item Attestation: This property allows a remote party to authenticate the software running inside an enclave.
\end{itemize}

\begin{figure}[htp]
    \centering
    \includegraphics[width=\linewidth]{flowchart.png}
    \caption{SGX operation Flowchart}
    \label{figure:flowchart}
\end{figure}

Intel SGX is the newest technology to solve Secure Remote Computation problem called Attestation by leveraging trusted hardware in the remote computer. Secure Remote Computation is the problem of executing software on a remote computer
owned and maintained by an untrusted party via attestaton mechanism. The trusted hardware establishes a secure container, and the remote computation service user uploads the desired computation and data into the secure container. The trusted hardware protects the data’s confidentiality and integrity while the computation is being performed on it. For example, a cloud service that performs image processing on confidential medical images could be implemented by having users upload encrypted images. The users would send the encryption keys to software running inside an enclave. The enclave would contain the code for decrypting images, the image processing algorithm, and the code for encrypting the results. The code that receives the uploaded encrypted images and stores them would be left outside the enclave. While SGX provides confidentiality and integrity of the data and computation inside an enclave by isolating it from outside environment, it remains compatible with traditional software layering in the Intel architecture.
