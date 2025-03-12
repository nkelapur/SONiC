# fpmsyncd NHG Warm-reboot

## High Level Design document

## Table of contents

- [Revision](#revision)

- [About this manual](#about-this-manual)

- [Scope](#scope)

- [Abbreviations](#abbreviations)

- [1 Introduction](#1-introduction)
    - [1.1 Feature overview](#11-feature-overview)
    - [1.2 Requirements](#12-requirements)
        - [1.2.1 Functionality](#121-functionality)
        - [1.2.2 Command interface](#122-command-interface)
    
- [2 Design](#2-design)
    - [2.1 Overview](#21-overview)
    - [2.2 Datastructures](#22-data-structure)
    - [2.2 Procedures](#23-procedures)
    - [2.3 DB schema](#23-db-schema)
        - [2.3.1 Config DB](#231-config-db)
        - [2.3.2 Data sample](#232-data-sample)
    - [2.4 Flows](#24-flows)
      - [2.4.1 Config flow](#241-config-flow)
        - [2.4.2 Route Programming flow](#242-route-programming-flow)
    - [2.5 Config Commands](#25-config-commands)
    - [2.6 Usage examples](#26-usage-examples)
    
- [3 Test plan](#3-test-plan)
  
    

## Revision

| Rev  |    Date    |     Author      | Description     |
| :--: | :--------: | :-------------: | :-------------- |
| 0.1  | 03/12/2025 | Nikhil Kelapure | Initial version |

## About this manual

This document provides general information about fpmsyncd NHG warm-reboot implementation in SONiC

## Scope

This document describes the high level design of fpmsyncd NHG warm-reboot nhid reconciliation in SONiC

**In scope:**  

1. NHG warm-reboot

## Abbreviations

| Term  | Meaning                                   |
| :---- | :---------------------------------------- |
| SONiC | Software for Open Networking in the Cloud |
| ECMP  | Equal-Cost Multi-Path                     |
| EVPN  | Ethernet Virtual Private Network          |
| FRR   | Free Range Routing                        |
| BGP   | Border Gateway Protocol                   |
| IP    | Internet Protocol                         |
| L3    | Layer 3                                   |
| OA    | Orchestration Agent                       |
| DB    | Database                                  |
| NH    | Next Hop                                  |
| NHG   | Next Hop Group                            |
| NHID  | Next Hop Group  ID                          |
| ToR   | Top-Of-Rack                               |
| API   | Application Programming Interface         |
| SAI   | Switch Abstraction Interface              |
| ASIC  | Application-Specific Integrated Circuit   |
| SWSS  | Switch State Service                      |
| CLI   | Сommand-line Interface                    |
| WB    | Warm Reboot                               |

## List of figures


## List of tables





# 1 Introduction

## 1.1 Feature overview

With the next-hop group feature in fpmsyncd, nexthops and routes are separated from each other. Nexthops are installed in app-db with unique identifiers as the key and routes are installed referring to a nexthop id.

FRR (zebra) allocates the nexthop ids and installs them to FPMsyncd. Currently, the next-hop ID allocated by zebra is used while installing next-hop group entries in app-db. In the bgp docker restart and warm-boot scenarios, when fpmsyncd comes back online, the app-db already has old next-hop group entries and route entries and zebra doesn’t have any awareness of the NHG entries which were already installed in app-db.

The two sets of NHG entries, i.e. new entries provided by zebra and old entries already present in app-db have to be reconciled in order to avoid disruption of traffic for already installed routes and nexthops.

This document explains the approach taken for the reconciliation of NHGs in fpmsyncd.


## 1.2 Requirements

### 1.2.1 Functionality

**This feature will support the following functionality:**
- Reconcile the old and new NHG id before and after warm reboot or bgp docker restart

### 1.2.2 Command line interface

No New command is added for this feature. 



# 2 Design

## 2.1 Overview

On warm reboot or docker restart Fpmsyncd restores the old nhid from app-db and reconciles them with the new nhid it receives from the control plain ( frr/zebra)

Fpmsyncd maintains following tables:
1. zebra nhg entries in form of map with key as zebra nhid
2. Free pool of fpm nhid values
3. fpm nhg entries in form of map with key as fpm nhid
4. app-db nhg entries retrieved at the startup of fpmsyncd

For recursive NHGs zebra creates connected NHGs with the same nexthop information. However the nexthop type is different. Hence to uniquely lookup the nhid from the given nexthop information, we will pass the nexthop type to fpm and write that value in  app-db.



## 2.2 Data structure
#### Zebra nhg entry (m_zebraNhgTable)
	key: int (zebra nhid)
	value:
		-  ifname, nexthop, vni_label, nexthop_type, fpm_nhid_value
		Or
		-   {member-fpmnh1, member-fpmnh2 .. member-fpmnhN} , fpm_nhid_value

#### Free pool of fpm nhid values (m_fpmNhidPool)
	This is maintained in the form of a bitmap.

#### Fpm nhg entry (m_fpmNhgTable)
	key: int (fpm nhid)
	value:
	zebra_nhid value

#### Appdb nhg entry (m_appNhgTable)
	key: 
		{ifname, nexthop, vni_label,nexthop_type}
		Or
		{member-fpmnh1, member-fpmnh2 .. member-fpmnhN} 
	value:
		fpm nhid
		list of zebra nhids mapped to this nhg entry



## 2.3 Procedures
#### Startup():
	Read app-db nhg entries and populate m_appNhgTable.
	Loop for each entry in m_appNhgTable:
		Mark corresponding nhid as occupied in m_fpmNhidPool.
		Mark zebra_nhid = 0
	Start NHG_RECONCOLE timer

#### nhgAdd():
	Lookup zebra_nhid entry in m_zebraNhgTable
	Not found:
		Look up key/contents in m_appNhgTable
		For recursive/ecmp NHGs, translate the member zebra nhid to fpm_nhid by looking up m_zebraNhgTable. 
		Skip and log error If member zebra nhid not found for translation
			Found:
				zebra_nhid list in m_appNhgTable entry empty?
				Append zebra_nhid in the m_appNhgTable entry list.
				Call addFpmNhgEntry(fpm_nhid) with given fpm_nhid
			Not found:
				Goto procedure: addFpmNhgEntry() 
	Found:
		Update contents of m_zebraNhgTable[zebra_nhid] entry
		Get the fpm_nhid and update app-db entry with this key

#### nhgDel():
	Lookup zebra_nhid entry in m_zebraNhgTable:
	Found:
		Call delFpmNhgEntry(m_zebraNhgTable[zebra_nhid].fpm_nhid)
		Delete app-db entry
		Delete m_zebraNhgTable[zebra_nhid] entry

#### addFpmNhgEntry():
	get free fpm_nhid from m_fpmNhidPool
	Add entry m_fpmNhgTable[fpm_nhid]
		m_fpmNhgTable[fpm_nhid].zebra_nhid = zebra_nhid
	Add entry m_zebraNhgTable[zebra_nhid]
		m_zebraNhgTable[zebra_nhid].fpm_nhid = fpm_nhid

#### addFpmNhgEntry(fpm_nhid):
	Alloc given fpm_nhid from m_fpmNhidPool
	Add entry m_fpmNhgTable[fpm_nhid]
		m_fpmNhgTable[fpm_nhid].zebra_nhid = zebra_nhid
	Add entry m_zebraNhgTable[zebra_nhid]
		m_zebraNhgTable[zebra_nhid].fpm_nhid = fpm_nhid

#### delFpmNhgEntry():
	Delete m_fpmNhgTable[fpm_nhid] entry
	Delete m_zebraNhgTable[zebra_nhid] entry
	Free fpm_nhid from m_fpmNhidPool

#### getFpmNhgId( zebra_nhid):
	Get fpm_nhid for given zebra_nhid for programming the routes to APP_DB
	If zebra_nhid is not already present, route programming will be skipped with ERROR  log

#### nhgReconcileTimeout():
	Loop through m_appNhgTable entries:
	Is zebra_nhid list is empty?
			Yes:
				Delete fpm nhid entry from app-db
				Delete entry from m_appNhgTable
			No:
				Delete entry from m_appNhgTable


## 2.2 configuration

### 2.2.1 L3 network

**Sample config:**


## 2.3 DB schema

### 2.3.1 Config DB

### 2.3.2 Data sample

**Config DB:**
```bash


```



## 2.4 Flows


## 2.5 Config Commands


## 2.6 Sample Tables Output


```bash
Route Table before warm-reboot

S>*  111.1.1.1/32 [1/0] (87) (0) via 120.1.1.2, Ethernet120, weight 1, 00:03:43
  *                              via 121.1.1.2, Ethernet121, weight 1, 00:03:43
  *                              via 122.1.1.2, Ethernet122, weight 1, 00:03:43
  *                              via 123.1.1.2, Ethernet123, weight 1, 00:03:43
C>*  120.1.1.0/24 (62) (0) is directly connected, Ethernet120, 00:04:55
C>*  121.1.1.0/24 (64) (0) is directly connected, Ethernet121, 00:04:43
C>*  122.1.1.0/24 (66) (0) is directly connected, Ethernet122, 00:04:34
C>*  123.1.1.0/24 (68) (0) is directly connected, Ethernet123, 00:04:22
S>*  222.1.1.1/32 [1/0] (93) (0) via 120.1.1.2, Ethernet120, weight 1, 00:00:04
  *                              via 121.1.1.2, Ethernet121, weight 1, 00:00:04

sonic# show nexthop-group rib 93
ID: 93 (zebra)
     RefCnt: 1
     Uptime: 00:00:42
     VRF: default
     Valid, Installed
     Depends: (70) (75)
           via 120.1.1.2, Ethernet120 (vrf default), weight 1
           via 121.1.1.2, Ethernet121 (vrf default), weight 1


sonic# show nexthop-group rib 87
ID: 87 (zebra)
     RefCnt: 1
     Uptime: 00:00:25
     VRF: default
     Valid, Installed
     Depends: (70) (75) (81) (88)
           via 120.1.1.2, Ethernet120 (vrf default), weight 1
           via 121.1.1.2, Ethernet121 (vrf default), weight 1
           via 122.1.1.2, Ethernet122 (vrf default), weight 1
           via 123.1.1.2, Ethernet123 (vrf default), weight 1
sonic# show nexthop-group rib 70
ID: 70 (zebra)
     RefCnt: 3
     Uptime: 00:00:29
     VRF: default
     Valid, Installed
     Interface Index: 79
           via 120.1.1.2, Ethernet120 (vrf default), weight 1
     Dependents: (87)
sonic# show nexthop-group rib 75
ID: 75 (zebra)
     RefCnt: 3
     Uptime: 00:00:32
     VRF: default
     Valid, Installed
     Interface Index: 80
           via 121.1.1.2, Ethernet121 (vrf default), weight 1
     Dependents: (87)
sonic# show nexthop-group rib 81
ID: 81 (zebra)
     RefCnt: 2
     Uptime: 00:00:36
     VRF: default
     Valid, Installed
     Interface Index: 81
           via 122.1.1.2, Ethernet122 (vrf default), weight 1
     Dependents: (87)
sonic# show nexthop-group rib 88
ID: 88 (zebra)
     RefCnt: 2
     Uptime: 00:00:38
     VRF: default
     Valid, Installed
     Interface Index: 82
           via 123.1.1.2, Ethernet123 (vrf default), weight 1
     Dependents: (87)
sonic#

Different NHG tables in fpmsyncd

Mar 05 04:13:42.639636+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: APP NHG TABLE:
Mar 05 04:13:42.639636+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: ZEBRA NHG TABLE:
Mar 05 04:13:42.639636+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 62 fpm_nhid: 24, NH:0.0.0.0:Ethernet120 VNI:, RMAC:, TYPE:1
Mar 05 04:13:42.639676+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 64 fpm_nhid: 25, NH:0.0.0.0:Ethernet121 VNI:, RMAC:, TYPE:1
Mar 05 04:13:42.639706+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 66 fpm_nhid: 26, NH:0.0.0.0:Ethernet122 VNI:, RMAC:, TYPE:1
Mar 05 04:13:42.639706+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 68 fpm_nhid: 27, NH:0.0.0.0:Ethernet123 VNI:, RMAC:, TYPE:1
Mar 05 04:13:42.639706+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 70 fpm_nhid: 28, NH:120.1.1.2:Ethernet120 VNI:, RMAC:, TYPE:2
Mar 05 04:13:42.639706+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 75 fpm_nhid: 29, NH:121.1.1.2:Ethernet121 VNI:, RMAC:, TYPE:2
Mar 05 04:13:42.639706+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 81 fpm_nhid: 30, NH:122.1.1.2:Ethernet122 VNI:, RMAC:, TYPE:2
Mar 05 04:13:42.639752+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 87 fpm_nhid: 24002, NHGRP:28,29,30,31
Mar 05 04:13:42.639752+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 88 fpm_nhid: 31, NH:123.1.1.2:Ethernet123 VNI:, RMAC:, TYPE:2
Mar 05 04:13:42.639752+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 93 fpm_nhid: 24003, NHGRP:28,29
Mar 05 04:13:42.639752+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: FPM NHG TABLE:
Mar 05 04:13:42.639752+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 24 zebra_nhid: 62
Mar 05 04:13:42.639752+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 25 zebra_nhid: 64
Mar 05 04:13:42.639752+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 26 zebra_nhid: 66
Mar 05 04:13:42.639850+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 27 zebra_nhid: 68
Mar 05 04:13:42.639850+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 28 zebra_nhid: 70
Mar 05 04:13:42.639850+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 29 zebra_nhid: 75
Mar 05 04:13:42.639850+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 30 zebra_nhid: 81
Mar 05 04:13:42.639850+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 31 zebra_nhid: 88
Mar 05 04:13:42.639850+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 24002 zebra_nhid: 87
Mar 05 04:13:42.639850+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 24003 zebra_nhid: 93


After Warm-reboot and before reconcillation :
One of the route prefix "222.1.1.1/32" is not learnt after warm-reboot incontrol plane.
The expectation is that at reconcillation the route and the corresponding NHG will be cleaned up.

S>*  111.1.1.1/32 [1/0] (117) (0) via 120.1.1.2, Ethernet120, weight 1, 00:02:15
  *                               via 121.1.1.2, Ethernet121, weight 1, 00:02:15
  *                               via 122.1.1.2, Ethernet122, weight 1, 00:02:15
  *                               via 123.1.1.2, Ethernet123, weight 1, 00:02:15
C>*  120.1.1.0/24 (103) (0) is directly connected, Ethernet120, 00:02:15
C>*  121.1.1.0/24 (104) (0) is directly connected, Ethernet121, 00:02:15
C>*  122.1.1.0/24 (105) (0) is directly connected, Ethernet122, 00:02:15
C>*  123.1.1.0/24 (106) (0) is directly connected, Ethernet123, 00:02:15


Mar 05 04:16:32.599786+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: APP NHG TABLE:
Mar 05 04:16:32.599817+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 0.0.0.0:Ethernet120:::1 zebra_nhid: 103 fpm_nhid: 24
Mar 05 04:16:32.599841+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 0.0.0.0:Ethernet121:::1 zebra_nhid: 104 fpm_nhid: 25
Mar 05 04:16:32.599863+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 0.0.0.0:Ethernet122:::1 zebra_nhid: 105 fpm_nhid: 26
Mar 05 04:16:32.599886+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 0.0.0.0:Ethernet123:::1 zebra_nhid: 106 fpm_nhid: 27
Mar 05 04:16:32.599910+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 120.1.1.2:Ethernet120:::2 zebra_nhid: 118 fpm_nhid: 28
Mar 05 04:16:32.599933+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 121.1.1.2:Ethernet121:::2 zebra_nhid: 119 fpm_nhid: 29
Mar 05 04:16:32.599956+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 122.1.1.2:Ethernet122:::2 zebra_nhid: 120 fpm_nhid: 30
Mar 05 04:16:32.599979+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 123.1.1.2:Ethernet123:::2 zebra_nhid: 121 fpm_nhid: 31
Mar 05 04:16:32.599998+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 28,29 zebra_nhid: 0 fpm_nhid: 24003
Mar 05 04:16:32.600024+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: NHG: 28,29,30,31 zebra_nhid: 117 fpm_nhid: 24002
Mar 05 04:16:32.600024+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: ZEBRA NHG TABLE:
Mar 05 04:16:32.600064+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 103 fpm_nhid: 24, NH:0.0.0.0:Ethernet120 VNI:, RMAC:, TYPE:1
Mar 05 04:16:32.600064+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 104 fpm_nhid: 25, NH:0.0.0.0:Ethernet121 VNI:, RMAC:, TYPE:1
Mar 05 04:16:32.600208+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 105 fpm_nhid: 26, NH:0.0.0.0:Ethernet122 VNI:, RMAC:, TYPE:1
Mar 05 04:16:32.600232+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 106 fpm_nhid: 27, NH:0.0.0.0:Ethernet123 VNI:, RMAC:, TYPE:1
Mar 05 04:16:32.600255+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 117 fpm_nhid: 24002, NHGRP:28,29,30,31
Mar 05 04:16:32.600279+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 118 fpm_nhid: 28, NH:120.1.1.2:Ethernet120 VNI:, RMAC:, TYPE:2
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 119 fpm_nhid: 29, NH:121.1.1.2:Ethernet121 VNI:, RMAC:, TYPE:2
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 120 fpm_nhid: 30, NH:122.1.1.2:Ethernet122 VNI:, RMAC:, TYPE:2
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: zebra_nhid: 121 fpm_nhid: 31, NH:123.1.1.2:Ethernet123 VNI:, RMAC:, TYPE:2
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: FPM NHG TABLE:
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 24 zebra_nhid: 103
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 25 zebra_nhid: 104
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 26 zebra_nhid: 105
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 27 zebra_nhid: 106
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 28 zebra_nhid: 118
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 29 zebra_nhid: 119
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 30 zebra_nhid: 120
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 31 zebra_nhid: 121
Mar 05 04:16:32.600436+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- debugDump: fpm_nhid: 24002 zebra_nhid: 117



At reconcillation :

Mar 05 04:18:05.547460+00:00 2025 sonic NOTICE bgp#fpmsyncd: :- reconcile: Warm-Restart reconciliation: deleting stale entry 24003 { nexthop_group: 28,29 }


```



# 3 Test plan

## 3.1 Unit tests 





