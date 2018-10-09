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
SAI XML profile exposes a new node that indicates wheter SDK starts in "ISSU enabled" mode:
SAI will start SDK in ```boot_mode=ISSU_ENABLE``` if "ISSU enabled" or SAI will start SDK in ```boot_mode=NORMAL``` if "ISSU disabled".

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
    - Check if FFB is supported via ```show platform mellanox issu status```, if not - return error
    - Burn new FW if new FW is available in next boot SONiC image - ```mlnx-fw-upgrade.sh --upgrade```
    - Execute ISSU start script inside ```syncd``` container via ```sx_api_issu.py``` from sx_examples or some custom script (```sx_api_issu_start_set()```)
  - Dump ARP/FDB entries from APP DB - existing step in FB
  - Mark reboot cause file - existing step in FB
    - Similar to FB:
      ```User issued 'fast-reboot' command [User: ${REBOOT_USER}, Time: ${REBOOT_TIME}]```
  - bgp, teamd dockers config restart in regular WARM SONiC way via CONFIG DB key
    (```WARM_RESTART_TABLE|bgp``` and ```WARM_RESTART_TABLE|teamd```).
  - stop bgp and teamd services via systemctl
    - According to system level WB design doc bgp will enable GR, teamd - just kill
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
    - Execute ISSU end script inside ```syncd``` container via ```sx_api_issu.py``` from sx_examples or some custom script  (```sx_api_issu_end_set()```)
  - END

## 2.4 As a W/A for ISSU end call

A process inside syncd can be started in case of 'fast-fast-boot' that will wait for $TIMEOUT sec and call ISSU end by executing ```sx_api_issu.py end```

# Open issues

## <b> When to execute ISSU end? </b>

What configuration is enough to call ISSU end?
<p>When is configuration done to call ISSU end? There is no good way to understand configuration from config_db was applied on HW.
<p>Also there is no good way to understand that all BGP routes were applied on HW.
<p>The possible W/A can include a script with timer started in syncd docker that after $TIMEOUT sec will call ISSU end


## <b> What would be the DP downtime? </b>

During the configure phase the DP will be disrupted.
It takes time for BGP routes to be advertised by a VM peer and inserted in HW.
So the downtime will depend on how quickly routes are received by BGP.

## <b> Who should set ```WARM_RESTART_TABLE```? CLI? User? </b>

This is not clear from 'System-wide Warmreboot' doc. Need to clarify

## <b> Is BGP/TeamD dockers Warm restart needed? </b>

The fpmsyncd will expect that there is stale route data in APPL DB to do reconcilation logic after gr timer.
Since APPL DB will be empty on start, do we need to start bgp docker in warm way? Except graceful restart is needed.

TeamD - unknown

# Approach 2 (more like WB flow)

Assume O/A will restart in Warm way. It will restore APP DB (includes routes, neighbors, etc), push config down to syncd. Syncd is in INIT_VIEW mode and will prorcess events from SWSS in different way by marking ASIC DB entries as TEMP_ (no configuration applied to HW). 

O/A notifies syncd to do APPLY_VIEW. syncd will do comparison logic between current (the view discovered from current ASIC state) view and temporary view and push the delta config to the HW.

From https://github.com/Azure/SONiC/blob/master/doc/warm-reboot/SONiC_Warmboot.md#the-existing-syncd-initapply-view-framework

"Essentially there will be two views created for warm restart. The current view represents the ASIC state before shutdown, temp view represents the new intended ASIC state after restart."

If we can reuse this approach in different way: e.g apply all configuration that is in temp view instead of computing delta ?

It could solve two issues:
  - We have exact moment of time when ISSU end can be called
  - This will insert all routes in ASIC as before shutdown, so we will not wait untill BGP routes are advertised
