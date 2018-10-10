# Mellanox Fast-Fast boot in SONiC
# High Level Design Document
### Rev 0.1
## Table of Contents
## List of Tables
###### Revision
| Rev |     Date    |       Author          | Change Description                |
|:---:|:-----------:|:---------------------:|-----------------------------------|
| 0.1 |             | Stepan Blyshchak      | Initial version                   |
## About this Manual
This document provides general information about the MLNX FFB feature implementation in SONiC.
## Scope
This document describes the high level design and flows of the MLNX FFB feature
## Definitions/Abbreviation
###### Table 2: Abbreviations
| Definitions/Abbreviation | Description                                |
|--------------------------|--------------------------------------------|
| FB                       | Fast boot                                  |
| FFB                      | Fast-Fast boot                             |
| ISSU                     | In Service Software Upgrade                |
| SDK                      | Software Development Kit                   |
| API                      | Application Programmable Interface         |
| SAI                      | Switch Abstraction Interface               |
| CLI                      | Command Line Interface                     |

So, MLNX Warm boot == MLNX FFB
# 1 Overview
MLNX FFB is an implementation of system level Warm boot in SONiC for Mellanox platforms that has a minimum data plane down time (less than 1 sec) for MSFT T0 configuration.

<b>NOTE</b>: This flow is not relevant for the 0 sec down time.

## 1.1 Components
### SAI
SAI XML profile exposes a new node that indicates wheter SDK starts in "ISSU enabled" mode.

### SYNCD
Same as for regular FB

### SWSS
Same as for regular FB

### BGP
Should invoke warm reboot flow

### TEAMD
Should invoke warm reboot flow

## Requirements
* The FFB flow is issued by Warm reboot CLI command on Mellanox platforms
  * On "ISSU enabled" devices - proceed with FFB flow
  * On "ISSU disabled" devices - return with appropriate error message
* Assumed only system level warm reboot via FFB is supported
* DP downtime should be less than 1 sec
* CP downtime should be the same as for FB flow - less than 90 sec

## Limitations
* No change in port mapping configuration (correct SDK/HW behaviour is NOT guaranteed otherwise)
* In "ISSU enabled" mode half of resources (RIFs, Routes, ACLs, etc.) are available


# 2 Components changes

## 2.1 ISSU status CLI command
In order to get whether SDK was started in "ISSU enabled" or normal mode we can add new platform show command in ```show``` commands group.

This command will be used during FFB to check whether we can FFB.

E.g:

```
admin@sonic:~ show platform mellanox issu status
ISSU ENABLED
```

The script will parse SAI XML profile to get the ISSU status node.

The same approach uses MLNX SDK Sniffer configuration.

## 2.2 syncd changes

* The syncd daemon should handle new startup command line flag ```-t fast-fast```.
* Syncd should pass to SAI ```SAI_KEY_BOOT_TYPE = 1```, so SAI will initialize SDK in FFB way.
* Also, unlike WB, syncd should NOT perform WARM start comparison logic.


## 2.3 syncd_init_common.sh changes

The syncd_init_common.sh should get that reboot cause was 'fast-fast-boot' from ```/proc/cmdline``` and pass option ```-t fast-fast``` to syncd start options.

## 2.2 Shutdown flow

The flow descibed on 'System-wide Warmboot' is describing the flow in general right now.

Specificaly, it is not clear who will do change config in ```WARM_RESTART_TABLE```. We will try to rereuse a part of it for BGP/TeamD dockers warm reboot as much as possible but the below flow of ```mlnx-ffb.sh``` will describe all that should be done.

- User issues Warm reboot CLI:
  - If Mellanox platform then invoke ```mlnx-ffb.sh```
  - Else - do regular SONiC warm reboot flow for the rest of platforms
  
### 2.2.1 Mellanox fast fast reboot flow 
#### ```mlnx-ffb.sh```
  - MLNX specific flow:
  
    - Check if FFB is supported via ```show platform mellanox issu status```, if not - reject the command
      <br>
      Example error message:
      <br>
        ```Fast fast reboot is not supported on "ISSU disabled device"```
     
    - Check if FFB is supported between two SDK version - call the script ```issu.py --check``` inside ```syncd``` and pass current SDK version and next SDK version as parameters
      <br>
      Example error message:
      <br>
        ```Fast fast reboot upgrade to SDK $VERSION_NEW from SDK $VERSION_OLD is not supported```
      <br>
      SDK version can be stored inside a file generated at image build time, e.g ```/etc/mlnx/sdk_version```
      <br>
      
      <b>NOTE:</b> This should be a very rare case. We do not plan to break the upgrade between versions, but just to have an additional check here.

    - Unset ```WARM_RESTART:system````
      
    - Burn new FW if new FW is available in next boot SONiC image: 
      <br>
      ```mlnx-fw-upgrade.sh --upgrade```
 
    - Execute ISSU start script ```issu.py --start``` inside ```syncd``` container
    
  - Dump ARP/FDB entries from APP DB - existing step in FB
  
  - Mark reboot cause file - existing step in FB
    - Similar to FB:
      ```User issued 'fast-fast-reboot' command [User: ${REBOOT_USER}, Time: ${REBOOT_TIME}]```
      
  - bgp, teamd dockers config restart in regular WARM SONiC way via State DB key
    <br>
    ```WARM_RESTART_TABLE|bgp```
    <br>
    ```WARM_RESTART_TABLE|teamd```
    <br>
    stop bgp and teamd services via systemctl
    <br>
    
    According to system level WB design doc bgp will enable GR, teamd - just kill
    
  - execute ```docker kill``` on every other container (swss, syncd, pmon, snmp, lldp)
  
  - BOOT_OPTIONS += 'fast-fast-reboot' (instead of 'fast-reboot' in FB case)
  
  - kexec $BOOT_OPTIONS

## 2.3 Startup flow
Similar to fast-reboot:
  - Start services
    - swss started normaly in non warm way
    
    - syncd:
      - ```if 'fast-fast-reboot' in $(cat /proc/cmdline); then```
        - started with with ```-t fast-fast```
        
    - bgp, teamd starts in warm way
    
  - O/A starts configuring HW
  
  - When configuration is done:
    - Execute ISSU end script ```issu.py --end``` inside ```syncd```
    
  - END

## 2.4 As a W/A for ISSU end call

A process inside syncd can be started in case of 'fast-fast-boot' that will wait for $TIMEOUT sec and call ISSU end by executing ```sx_api_issu.py end```

# Open issues

## <b> When to execute ISSU end? </b>

The possible W/A can check when the default route was added? 


