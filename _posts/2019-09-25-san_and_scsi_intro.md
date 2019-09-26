---
layout: single
title:  "SAN,SCSI等介绍"
categories:
  - Written in Chinese
  - storage
tags:
  - storage
toc: true
excerpt_separator: <!--more-->
---

最近在使用ceph的iSCSI driver时，发现一些存储相关的术语让人很困惑，也不清楚背后的技术。这篇文章记录了一些调研结果。

<!--more-->

## 术语

以下是摘自wikipedia的，很多存储相关术语的解释，在试图搞清楚这些技术所处的层次，和之间的关系前，先让我们弄清楚他们的定义。

- > [SAN](<https://en.wikipedia.org/wiki/Storage_area_network>)：A **storage area network** (**SAN**) or **storage network** is a [computer network](https://en.wikipedia.org/wiki/Computer_network) which provides access to consolidated, [block-level data storage](https://en.wikipedia.org/wiki/Block_device). SANs are primarily used to enhance accessibility of storage devices, such as [disk arrays](https://en.wikipedia.org/wiki/Disk_array) and [tape libraries](https://en.wikipedia.org/wiki/Tape_library), to [servers](https://en.wikipedia.org/wiki/Server_(computing)) so that the devices appear to the [operating system](https://en.wikipedia.org/wiki/Operating_system) as [locally-attached devices](https://en.wikipedia.org/wiki/Direct-attached_storage).

  SAN是指存储区域网络，它可以提供统一的，块级别的存储。SAN主要用来增强一些存储设备对server的可用性，使这些设备在操作系统看来，就像是本地连接的设备一样。

- > [SCSI](https://en.wikipedia.org/wiki/SAN): **Small Computer System Interface** (**SCSI**, [/ˈskʌzi/](https://en.wikipedia.org/wiki/Help:IPA/English) [*SKUZ-ee*](https://en.wikipedia.org/wiki/Help:Pronunciation_respelling_key))[[1\]](https://en.wikipedia.org/wiki/SCSI#cite_note-1) is a set of standards for physically connecting and transferring data between computers and [peripheral devices](https://en.wikipedia.org/wiki/Peripheral). The SCSI standards define [commands](https://en.wikipedia.org/wiki/SCSI_command), protocols, electrical, optical and logical [interfaces](https://en.wikipedia.org/wiki/Interface_(computing)). SCSI is most commonly used for [hard disk drives](https://en.wikipedia.org/wiki/Hard_disk_drive) and [tape drives](https://en.wikipedia.org/wiki/Tape_drive), but it can connect a wide range of other devices, including scanners and [CD](https://en.wikipedia.org/wiki/CD-ROM) [drives](https://en.wikipedia.org/wiki/Optical_disc_drive), although not all controllers can handle all devices.

  SCSI是定义了计算机和一些外设之间如何互连和传输数据的一系列标准。SCSI标准集定义了命令，协议，电子、光学、以及逻辑接口。SCSI最主要用在硬盘和磁盘，但它也可以连接很多其他设备，比如CD，扫描仪等。

- > [ISCSI](<https://en.wikipedia.org/wiki/ISCSI>): In computing, **iSCSI** ([/ˈaɪskʌzi/](https://en.wikipedia.org/wiki/Help:IPA/English) ([![About this sound](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8a/Loudspeaker.svg/11px-Loudspeaker.svg.png)](https://en.wikipedia.org/wiki/File:ISCSI_pronunciation.ogg)[listen](https://upload.wikimedia.org/wikipedia/commons/6/65/ISCSI_pronunciation.ogg)) [*EYE-skuz-ee*](https://en.wikipedia.org/wiki/Help:Pronunciation_respelling_key)) is an acronym for **Internet Small Computer Systems Interface**, an [Internet Protocol](https://en.wikipedia.org/wiki/Internet_Protocol) (IP)-based storage networking standard for linking data storage facilities. It provides [block-level access](https://en.wikipedia.org/wiki/Block-level_storage) to [storage devices](https://en.wikipedia.org/wiki/Computer_data_storage) by carrying [SCSI](https://en.wikipedia.org/wiki/SCSI) commands over a [TCP/IP](https://en.wikipedia.org/wiki/TCP/IP) network. iSCSI is used to facilitate data transfers over [intranets](https://en.wikipedia.org/wiki/Intranet) and to manage storage over long distances. It can be used to transmit data over [local area networks](https://en.wikipedia.org/wiki/Local_area_network) (LANs), [wide area networks](https://en.wikipedia.org/wiki/Wide_area_network) (WANs), or the [Internet](https://en.wikipedia.org/wiki/Internet) and can enable location-independent data storage and retrieval.

  ISCSI是个基于IP协议的，定义了如何连接存储设备的网络标准，通过在TCP/IP网络上传输SCSI命令，它提供了块级别的存储。它被用来简化数据在内网的传输，和远距离的存储。它也可以被用来通过LAN、WAN，或者Internet来传输数据，同时构造了一种位置无关的数据存取系统。

- > [FC](<https://en.wikipedia.org/wiki/Fibre_Channel): **Fibre Channel** (**FC**) is a high-speed data transfer protocol (commonly running at 1, 2, 4, 8, 16, 32, and 128 [gigabit](https://en.wikipedia.org/wiki/Gigabit) per second rates) providing in-order, lossless[[1\]](https://en.wikipedia.org/wiki/Fibre_Channel#cite_note-fibrechannel.org-1) delivery of raw block data[[2\]](https://en.wikipedia.org/wiki/Fibre_Channel#cite_note-2), primarily used to connect [computer data storage](https://en.wikipedia.org/wiki/Computer_data_storage) to [servers](https://en.wikipedia.org/wiki/Server_(computing)).[[3\]](https://en.wikipedia.org/wiki/Fibre_Channel#cite_note-Preston-3)[[4\]](https://en.wikipedia.org/wiki/Fibre_Channel#cite_note-Riabov-4) Fibre Channel is mainly used in [storage area networks](https://en.wikipedia.org/wiki/Storage_area_network) (SAN) in commercial [data centers](https://en.wikipedia.org/wiki/Data_center). Fibre Channel networks form a [switched fabric](https://en.wikipedia.org/wiki/Switched_fabric) because they operate in unison as one big switch. Fibre Channel typically runs on [optical fiber](https://en.wikipedia.org/wiki/Optical_fiber) cables within and between data centers, but can also run on copper cabling.[[3\]](https://en.wikipedia.org/wiki/Fibre_Channel#cite_note-Preston-3)[[4\]](https://en.wikipedia.org/wiki/Fibre_Channel#cite_note-Riabov-4)

  FC是一种高速的数据传输协议，可提供有序的，无数据丢失的原生的块数据的分发，主要被用来连接存储设备和服务器。它主要备用在商业数据中心的SAN中。Fibre Channel网络向一个大的switch一样协同运作，形成了一个交换组（fabric：结构，组织）。FC通常运行在数据中心内部或数据中心之间的光纤网络上，但也可以运行在銅轴电缆网络之上。

- > FCP:  [Fibre Channel Protocol](https://en.wikipedia.org/wiki/Fibre_Channel_Protocol) (FCP) is a transport protocol that predominantly transports [SCSI](https://en.wikipedia.org/wiki/Small_Computer_System_Interface) commands over Fibre Channel networks.

  FCP是一个主要用在通过FC网络传输SCSI命令的协议。

- > [FCoE](https://en.wikipedia.org/wiki/Fibre_Channel_over_Ethernet): **Fibre Channel over Ethernet** (**FCoE**) is a [computer network](https://en.wikipedia.org/wiki/Computer_network) technology that encapsulates [Fibre Channel](https://en.wikipedia.org/wiki/Fibre_Channel) frames over [Ethernet](https://en.wikipedia.org/wiki/Ethernet) networks. This allows Fibre Channel to use [10 Gigabit Ethernet](https://en.wikipedia.org/wiki/10_Gigabit_Ethernet) networks (or higher speeds) while preserving the Fibre Channel protocol. 

  FCoE是一种封装了Fibre Channel帧，在以太网上传输的网络技术。通过这种技术可以允许FC协议运行在10GB或更高速的以太网络之上。

- > [FCIP](https://en.wikipedia.org/wiki/Fibre_Channel_over_IP): **Fibre Channel over IP** (**FCIP** or **FC/IP**, also known as **Fibre Channel tunneling** or **storage tunneling**) is an [Internet Protocol](https://en.wikipedia.org/wiki/Internet_Protocol) (IP) created by the [Internet Engineering Task Force](https://en.wikipedia.org/wiki/Internet_Engineering_Task_Force) (IETF) for storage technology.An FCIP entity functions to encapsulate [Fibre Channel](https://en.wikipedia.org/wiki/Fibre_Channel) frames and forward them over an IP network. FCIP entities are peers that communicate using TCP/IP. FCIP technology overcomes the distance limitations of native Fibre Channel, enabling geographically distributed [storage area networks](https://en.wikipedia.org/wiki/Storage_area_network) to be connected using existing IP infrastructure, while keeping fabric services intact. The Fibre Channel Fabric and its devices remain unaware of the presence of the IP Network.

  FCIP和FCoE有点相似，但它是将FC帧封装后走了IP网络，所以FCIP可以克服原生的FC协议的距离限制。

- > [NFS](https://en.wikipedia.org/wiki/Network_File_System): **Network File System** (**NFS**) is a [distributed file system](https://en.wikipedia.org/wiki/Distributed_file_system) protocol originally developed by [Sun Microsystems](https://en.wikipedia.org/wiki/Sun_Microsystems) (Sun) in 1984,[[1\]](https://en.wikipedia.org/wiki/Network_File_System#cite_note-sun85-1) allowing a user on a client [computer](https://en.wikipedia.org/wiki/Computer) to access files over a [computer network](https://en.wikipedia.org/wiki/Computer_network) much like local storage is accessed. NFS, like many other protocols, builds on the [Open Network Computing Remote Procedure Call](https://en.wikipedia.org/wiki/Open_Network_Computing_Remote_Procedure_Call) (ONC RPC) system.

  NFS是一个最初由Sun开发的分布式的文件系统协议，允许用户使用客户端，像访问本地存储一样，通过网络来访问文件。

- > [CIFS](<https://en.wikipedia.org/wiki/Server_Message_Block>): In [computer networking](https://en.wikipedia.org/wiki/Computer_network), **Server Message Block** (**SMB**), one version of which was also known as **Common Internet File System** (**CIFS** [/sɪfs/](https://en.wikipedia.org/wiki/Help:IPA/English)),[[1\]](https://en.wikipedia.org/wiki/Server_Message_Block#cite_note-1)[[2\]](https://en.wikipedia.org/wiki/Server_Message_Block#cite_note-2) is a network [communication protocol](https://en.wikipedia.org/wiki/Communication_protocol)[[3\]](https://en.wikipedia.org/wiki/Server_Message_Block#cite_note-3) for providing [shared access](https://en.wikipedia.org/wiki/Shared_access) to [files](https://en.wikipedia.org/wiki/Computer_file), [printers](https://en.wikipedia.org/wiki/Computer_printer), and [serial ports](https://en.wikipedia.org/wiki/Serial_port) between nodes on a network.

  CIFS是一个定义了网络中的节点如何共享文件、打印机、串口设备的网络通信协议。

- > [DAS](https://en.wikipedia.org/wiki/Direct-attached_storage): **Direct-attached storage** (**DAS**) is [digital storage](https://en.wikipedia.org/wiki/Data_storage_device) directly attached to the [computer](https://en.wikipedia.org/wiki/Computer) accessing it, as opposed to storage accessed over a computer network (i.e. [network-attached storage](https://en.wikipedia.org/wiki/Network-attached_storage)). Examples of DAS include [hard drives](https://en.wikipedia.org/wiki/Hard_drive), [solid-state drives](https://en.wikipedia.org/wiki/Solid-state_drive), [optical disc drives](https://en.wikipedia.org/wiki/Optical_disc_drive), and storage on [external drives](https://en.wikipedia.org/wiki/External_drive). The name "DAS" is a [retronym](https://en.wikipedia.org/wiki/Retronym) to contrast with [storage area network](https://en.wikipedia.org/wiki/Storage_area_network) (SAN) and network-attached storage (NAS).

  DAS是一个用来和NAS、SAN进行区分的术语，它主要指的是硬盘，固态硬盘，光驱等直连设备。

- > [NAS](https://en.wikipedia.org/wiki/Network-attached_storage): **Network-attached storage** (**NAS**) is a file-level (as opposed to [block-level](https://en.wikipedia.org/wiki/Block_device)) [computer data storage](https://en.wikipedia.org/wiki/Computer_data_storage) server connected to a [computer network](https://en.wikipedia.org/wiki/Computer_network) providing data access to a [heterogeneous](https://en.wikipedia.org/wiki/Heterogeneous_computing) group of clients. NAS is specialized for [serving files](https://en.wikipedia.org/wiki/File_server) either by its hardware, software, or configuration. It is often manufactured as a [computer appliance](https://en.wikipedia.org/wiki/Computer_appliance) – a purpose-built specialized computer.[[nb 1\]](https://en.wikipedia.org/wiki/Network-attached_storage#cite_note-1) NAS systems are networked appliances which contain one or more [storage drives](https://en.wikipedia.org/wiki/Hard_disk_drive), often arranged into logical, redundant storage containers or [RAID](https://en.wikipedia.org/wiki/RAID).

  NAS是一个文件级别的存储服务器，对异构的client提供数据访问服务。NAS不论是从硬件、软件和配置上，都是专门用来服务文件的。

## 操作系统的视角

[这张图](<https://forum.huawei.com/enterprise/en/differences-between-scsi-iscsi-fcp-fcoe-fcip-nfs-cifs-das-nas-san/thread/229549-891>)从操作系统的视角来描绘了上面提到的很多协议，结合它们的定义，他们的层级关系和互相之间的关联关系便一目了然了。

<figure>
  <img src="/assets/images/scsi-arch.png">
<figcaption>SCSI</figcaption>
</figure>

## Ceph的SAN实现

ceph实现了iSCSI协议，结合RBD，对外提供块级别存储服务。以下是ceph iSCSI的架构图。


<figure>
  <img src="/assets/images/ceph-iscsi.png">
<figcaption>ceph iSCSI</figcaption>
</figure>
