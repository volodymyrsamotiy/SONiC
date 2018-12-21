# VNET Unit Test Plan

* [Overview](#Overview)
    * [Scope](#Scope)
    * [Testbed](#Testbed)
    * [Setup configuration](#Setup%20configuration)
* [Test](#Test)
    * [PyTest scripts to setup and run test](#PyTest%20scripts%20to%20setup%20and%20run%20test)
        * [test_vnet.py](#test_vnet.py)
    * [Test cases](#Test%20cases)
    * [Mellanox SAI Attributes](Mellanox%20SAI%20Attributes)
* [Open Questions](#Open%20Questions)

## Overview
The purpose is to test VNET feature on the SONiC virtual switch environment.

### Scope
The test is targeting a running SONIC virtual switch environment. The purpose is not full fuctional testing on real HW, but unit and integration testing of VNET feature on SONIC virtual switch. The main goal is to test SwSS implementation for VNET implementation by applying config to AppDB/ConfigDB and checking corresponding AsicDB entries.

### Testbed
The test will run on the SONiC virtual switch environment.

## Setup configuration
No setup pre-configuration is required, test will configure and clean-up all the configuration. The test can be ran on any Linux machine without real HW.

## Test

### PyTest scripts to setup and run the test
There is already ```test_vnet.py``` test suite implemented for the legacy implementation but not for Mellanox implementation.
SONiC configuration in ConfigDB and AppDB is the same for both implementations, so no changes required from SONiC UI perspective.
The only difference is in SAI configuration, so in order to test Mellanox implemention need to modify current test:
- Modify test in order to be able to run it in two modes: first for legacy implementation and second one for Mellanox.
- Change test "check AsicDB config" logic in order to support both legacy and Mellanox implementations.
- Test should verify different SAI attributes in AsicDB for each implementation.

#### test_vnet.py
Test suite script ```test_vnet.py``` should test the following:
- Create VxLAN tunnel
- Create VNET entry
- Create RIF for VNET
- Create VNET route

### Test cases
All listed below test cases are already implemented for legacy implementation in ```test_vnet.py```. This test suite shoud be modified in order to run it for Mellanox implementation: need to add support of checking different SAI attributes (for Mellanox it is ```sai_bmtor_api```).

Link to ```test_vnet.py``` PyTest test suite: 
[https://github.com/Azure/sonic-swss/blob/master/tests/test_vnet.py](https://github.com/Azure/sonic-swss/blob/master/tests/test_vnet.py)

Link to ```sai_bmtor_api``` SAI header: 
[https://github.com/opencomputeproject/SAI/blob/master/experimental/saiexperimentalbmtor.h](https://github.com/opencomputeproject/SAI/blob/master/experimental/saiexperimentalbmtor.h)

#### Mellanox SAI Attributes
For Mellanox and legacy implementation different SAI attributes are used so need to check different entries in AsicDB.
##### VxLAN Tunnel
SAI attributes are common for Mellanox and legacy implementation, no changes required.
##### VNET Entry
Mellanox ```sai_bmtor_api``` SAI attributes to be checked:
* TODO: Clarify and add list of specific SAI attributes
##### RIF for VNET
Mellanox ```sai_bmtor_api``` SAI attributes to be checked:
* TODO: Clarify and add list of specific SAI attributes
##### VNET Route
Mellanox ```sai_bmtor_api``` SAI attributes to be checked:
* TODO: Clarify and add list of specific SAI attributes

#### Test case # 1 – "Create Interface, Tunnel and VNET"
##### Test objective
Verify that VxLAN tunnel, VNET and interface can be created successfully.
##### Test steps
* Create VxLAN tunnel in ConfigDB.
* Verify that VxLAN tunnel is created in AsicDB by checking SAI attributes.
* Create VNET entry in ConfigDB.
* Verify that VNET entry is created in AsicDB by checking SAI attributes.
* Create RIF for the VNET in ConfigDB.
* Verify that RIF is created in AsicDB by checking SAI attributes.
* Create VNET route in ConfigDB.
* Verify that VNET route is created in AsicDB by checking SAI attributes.
### Test case # 2 – "Create two VNETs"
#### Test objective
Verify that  two VNETs can be created successfully.
#### Test steps
* Create VxLAN tunnel in ConfigDB.
* Verify that VxLAN tunnel is created in AsicDB by checking SAI attributes.
* Create VNET_1 entry in ConfigDB.
* Verify that VNET_1 entry is created in AsicDB by checking SAI attributes.
* Create RIF for the VNET_1 in ConfigDB.
* Verify that RIF for VNET_1 is created in AsicDB by checking SAI attributes.
* Create VNET_1 routes in ConfigDB.
* Verify that VNET_1 routes are created in AsicDB by checking SAI attributes.
* Create VNET_2 entry in ConfigDB.
* Verify that VNET_2 entry is created in AsicDB by checking SAI attributes.
* Create RIF for the VNET_2 in ConfigDB.
* Verify that RIF for VNET_2 is created in AsicDB by checking SAI attributes.
* Create VNET_2 routes in ConfigDB.
* Verify that VNET_2 routes are created in AsicDB by checking SAI attributes.
### Test case # 3 – "Create two VNETs and VNET peering"
#### Test objective
Verify that two VNETs and VNET peering can be created successfully.
#### Test steps
* Create VxLAN tunnel in ConfigDB.
* Verify that VxLAN tunnel is created in AsicDB by checking SAI attributes.
* Create VNET_1 entry in ConfigDB.
* Verify that VNET_1 entry is created in AsicDB by checking SAI attributes.
* Create VNET_2 entry in ConfigDB.
* Verify that VNET_2 entry is created in AsicDB by checking SAI attributes.
* Create RIF for the VNET_1 in ConfigDB.
* Verify that RIF for VNET_1 is created in AsicDB by checking SAI attributes.
* Create RIF for the VNET_2 in ConfigDB.
* Verify that RIF for VNET_2 is created in AsicDB by checking SAI attributes.
* Create VNET_1 routes in ConfigDB.
* Verify that VNET_1 routes are created in AsicDB by checking SAI attributes.
* Create VNET_2 routes in ConfigDB.
* Verify that VNET_2 routes are created in AsicDB by checking SAI attributes.

## Open Questions
1. Do we need any virtual switch modifications in order to add support of new SAI APIs (```sai_bmtor_api```)?
2. Orchagent implementation is different for Mellanox and other platforms, so virtual switch should be able to run in different platform modes.
    It is needed because there is platform check in orchagent in order to use different logic for Mellanox and other platforms:
    ```cpp
    if (platform == MLNX_PLATFORM_SUBSTRING)
    {
        vnet_orch = new VNetBitmapOrch(m_applDb, APP_VNET_TABLE_NAME);
    }
    else
    {
        vnet_orch = new VNetVrfOrch(m_applDb, APP_VNET_TABLE_NAME);
    }
    ```
    It requires some investigation and for now the question is: "What is needed in order to run virtual switch as different platforms?".
3. Do we need to add more test cases?
