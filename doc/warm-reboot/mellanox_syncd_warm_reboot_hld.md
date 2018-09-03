# Mellanox SYNCD Warm-Reboot
# High Level Design Document
### Rev 0.1

# Table of Contents

  * [List of Tables](#list-of-tables)
  * [Revision](#revision)
  * [About this Manual](#about-this-manual)
  * [Scope](#scope)vi
  * [Definitions/Abbreviation](#definitionsabbreviation)
  * [1. Subsystem Requirements](#1-subsystem-requirements)
    * [1.1. Functional Requirements](#11-functional-requirements)
    * [1.1. Non-Functional Requirements](#11-non-functional-requirements)
  * [2. Modules Design](#2-modules-design)
    * [2.1. SYNCD](#21-syncd)
  * [3. Flows](#3-flows)
	* [3.1. SYNCD Warm-Reboot](#31-syncd-warm-reboot)
  * [4. Open Questions](#4-open-questions)

# List of Tables
* [Table 1: Revision](#revision)
* [Table 2: Abbreviations](#definitionsabbreviation)
###### Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 |             | Volodymyr Samotiy  | Initial version                   |

# About this Manual
This document provides general information about the SYNCD warm-reboot implementation in SONiC for Mellanox platforms.
# Scope
This document describes high level design of the warm-reboot feature only for SYNCD sub-component on Mellanox platforms. All other warm-reboot parts are out of the scope of this document.
# Definitions/Abbreviation
###### Table 2: Abbreviations
Definitions/Abbreviation | Description                                
-------------------------|--------------------------------------------
SONiC                    | Software for Open Networking in the Cloud
SYNCD                    | SONiC component that takes the SAI objects from the Redis DB (```ASIC_DB```) and puts them to the ASIC
API                      | Application Programmable Interface         |
SAI                      | Switch Abstraction Interface                |
CRIU                     | Checkpoint/Restore In Userspace                |
# 1. Subsystem Requirements
## 1.1. Functional Requirements
This section describes the SONiC functional requirements for SYNCD warm-reboot on Mellanox platforms.
- SYNCD should be aware of performing warm-reboot.
- When warm-reboot is performed for SYNCD it must ensure the state of ASIC and SAI restores to the same state as before reboot.
- CRIU tool should be used to create dump of SYNCD processes when warm-reboot is performed:
	- ```syncd``` process should be dumped
	- ```sx_sdk``` process should be dumped
- SYNCD should copy shared SDK files (``` /var/run/shm/*```) to persistent location when warm-reboot is performed.
- SYNCD should copy shared SDK files back (``` /var/run/shm/*```) from persistent location when we boot after warm-reboot.
- CRIU tool should be used to restore SYNCD processes when it boots after warm-reboot:
	- ```syncd``` process should be restored
	- ```sx_sdk``` process should be restored
## 1.2. Non-Functional Requirements
This section describes the SONiC non-functional requirements for SYNCD warm-reboot on Mellanox platforms.
- ```syncd``` docker container should be updated in order to have CRIU tool installed
- SYNCD should be represented as a separate systemd service

# 2. Modules Design
## 2.1. SYNCD
New script should be provided in order to be executed on SYNCD shutdown and start.
On shutdown it should check whether SYNCD goes down because of warm-reboot, if yes it should perform actions to dump SYNCD processes using CRIU tool:
- SAI attribute ```SAI_SWITCH_ATTR_RESTART_WARM``` should be set to ```true```
- Create checkpoints for SYNCD processes:
	- ```syncd``` process should be dumped to persistant location
	- ```sx_sdk``` process should be dumped to persistant location
- Copy shared SDK files (``` /var/run/shm/*```) to persistent location when warm-reboot is performed.

On start SYNCD should know whether it is booting after warm-reboot, it yes it should perform actions to restore SYNCD processes using CRIU tool:
- Copy shared SDK files back (``` /var/run/shm/*```) from persistent location when we boot after warm-reboot.
- Restore SYNCD processes when it boots after warm-reboot:
	- ```syncd``` process should be restored from persistant location
	- ```sx_sdk``` process should be restored from persistant location
 - SAI attribute ```SAI_SWITCH_ATTR_WARM_RECOVER``` should be set to ```true```

**Note:** Netdevs will be restored by SAI after warm-reboot. However SAI isn't saving/restoring the netdev attributes such as IP, MTU, admin mode, which aren't managed by SAI. SONiC should restore this config.

# 3. Flows
## 3.1. SYNCD warm-reboot
![](https://github.com/volodymyrsamotiy/SONiC/raw/warm-reboot/images/mellanox_syncd-warm-reboot_hld.png)

# 4. Open Questions
1. How ```SYNCD``` should know that warm-reboot is executed, will it be available in  ```WARM_RESTART_TABLE``` from ```STATE_DB```?
2. Who will set SAI warm-reboot attribute before actual warm-reboot procedure?
3. What is relation between SWSS and SYNCD, can be SYNCD rebooted separately once it is provided as a systemd service?
4. How netdev config will be reapplied once SYNCD is booted after warm-reboot?
5. What persistant location can be used for storing processes dumps in ```SYNCD```?

