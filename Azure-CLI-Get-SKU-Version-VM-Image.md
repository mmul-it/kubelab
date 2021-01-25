# Using az cli to get sku and version of a specific VM image

This command sequence explains how to get the available sku/version starting
from the location.

* First, you want to know which publishers are available in the zone (i.e.
  ukwest):
  ```
  az vm image list-publishers --location ukwest --output table
  Location    Name
  ----------  ----------------------------------------------------------------------------------
  ukwest      128technology
  ukwest      1580863854728
  ukwest      1583465680865
  ukwest      1585118004523
  ukwest      1597644262255
  ukwest      1598955805825
  ukwest      1e
  ukwest      2021ai
  ukwest      3cx-pbx
  ukwest      42crunch1580391915541
  ukwest      4psa
  ukwest      5nine-software-inc
  ukwest      7isolutions
  ukwest      a10networks
  ...
  ```
* Then you'll need to get all the offered images from the specific publisher
  (i.e. OpenLogic):
  ```
  az vm image list-offers --location ukwest --publisher OpenLogic --output table
  Location    Name
  ----------  -------------------------
  ukwest      CentOS
  ukwest      CentOS-CI
  ukwest      CentOS-HPC
  ukwest      CentOS-LVM
  ukwest      CentOS-SRIOV
  ukwest      centos_corevm_test-200331
  ukwest      centos_corevm_test-200408
  ```
* At this point you'll want to know the available skus for the offered image
  (i.e. CentOS):
  ```
  az vm image list-skus --location ukwest --publisher OpenLogic --offer CentOS --output table
  Location    Name
  ----------  --------
  ukwest      6.10
  ukwest      6.5
  ...
  ukwest      7_9
  ukwest      7_9-gen2
  ...
  ukwest      8_3
  ukwest      8_3-gen2
  ```
* Finally, you'll get the specific version depending on the sku you selected
  (i.e. 7_9-gen2):
  ```
  az vm image list --location ukwest --publisher OpenLogic --offer CentOS --sku 7_9-gen2 --all  --output table
  Offer    Publisher    Sku       Urn                                       Version
  -------  -----------  --------  ----------------------------------------  --------------
  CentOS   OpenLogic    7_9-gen2  OpenLogic:CentOS:7_9-gen2:7.9.2020111901  7.9.2020111901
  ```
