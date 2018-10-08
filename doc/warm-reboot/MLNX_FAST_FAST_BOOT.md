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
SAI XML profile exposes a new node that indicates wheter ISSU mode is enabled:
SAI will start SDK in ```boot_mode=ISSU_ENABLE``` if ISSU enabled or SAI will start SDK in ```boot_mode=NORMAL``` if ISSU disabled.

SAI XML profile also contains initial port mapping/split, speed, etc. configuration that SAI applies to SDK.
In ISSU enabled mode port mapping/split configuration defined in SAI XML should match the port configuration in port_config.ini

### SYNCD
Same as for regular FB

### SWSS
Same as for regular FB

### BGP
Should invoke warm reboot flow

### TEAMD
Should invoke warm reboot flow

## Requirements
* The FFB flow should be issued by Warm reboot CLI command only on Mellanox platforms only on ISSU enabled HWSKUs
* Warm reboot via MLNX FFB should be available only in system level warm reboot
* ISSU disabled Mellanox HWSKUs will not support Warm reboot
* DP downtime should be less than 1 sec
* CP downtime should be the same as for FB flow - less than 90 sec

## Limitations
* ISSU enabled Mellanox HWSKUs cannot change port split configuration
* In ISSU enabled mode only half of resources (RIFs, Routes, ACLs, etc.) are available


# 2 Components changes

## 2.1 ISSU status CLI command
In order to get whether SDK was started in ISSU_ENABLED or NORMAL mode we can add new platform show command in ```show``` commands group.

This command will be used during FFB to check whether we can FFB.

E.g:

```
admin@sonic:~ show platform mellanox issu status
ISSU ENABLED
```

The script will parse SAI XML profile to get the ISSU status node.

The same approach uses MLNX SDK Sniffer configuration.

<b>Q:</b> Should we also provide ```config platform mellanox issu [enable|disable]``` ?

If we will provide ```config``` we will also do a check for SDK version if ISSU supported. If not return error.

The command will then change SAI XML profile and ask user to restart ```swss``` service.

## 2.2 syncd changes

The syncd daemon should handle ```-t warm``` option on start differently for Mellanox platform.
Syncd should NOT perform WARM start logic related to 'init/temp' view.
Syncd should pass to SAI ```SAI_KEY_BOOT_TYPE = 1```, so SAI will initialize SDK in FFB way.

## 2.3 syncd_init_common.sh changes

The syncd_init_common.sh should get that reboot cause was 'fast-fast-boot' and pass option ```-t warm``` to syncd start options.

## 2.2 Shutdown flow
- User issues Warm reboot CLI:
  - If Mellanox platform then invoke ```mlnx-ffb.sh```
  - Else - do regular SONiC warm reboot flow for the rest of platforms
  
### 2.2.1 Mellanox fast fast reboot flow 
#### ```mlnx-ffb.sh```
  - MLNX specific flow:
    - Check if upgrade to new SDK is supported with FFB (```sx_api_issu_start_set()``` to check), if not - return error
    - Burn new FW if new FW is available in next boot SONiC image
    - Execute ISSU start script inside ```syncd``` container via ```sx_api_issu.py``` from sx_examples or some custom script (```sx_api_issu_start_set()```)
  - Dump ARP/FDB entries from APP DB - existing step in FB
  - Mark reboot cause file as MLNX FFB - existing step in FB (<b>Q:</b> I don't think this file was ever used)
  - bgp, teamd dockers config restart in regular WARM SONiC way via CONFIG DB key
    (```WARM_RESTART_TABLE|bgp``` and ```WARM_RESTART_TABLE|teamd```).
  - stop bgp and teamd services via systemctl
    - According to system level WB design doc bgp will enable GR, teamd - just kill
  - execute ```docker kill``` on every other container
  - BOOT_OPTIONS += 'fast-fast-reboot' (instead of 'fast-reboot' in FB case)
  - kexec $BOOT_OPTIONS

## 2.3 Startup flow
Similar to fast-reboot:
  - Start services
    - swss started normaly in non warm way
    - syncd:
      - ```if 'fast-fast-reboot' in $(cat /proc/cmdline); then```
        - started with with ```-t warm``` ( But syncd will not do any WB logic, only passes SAI_WARM_BOOT key to SAI )
    - bgp, teamd start in warm way
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
