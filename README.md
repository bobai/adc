# Tracking HW problem on Dell AMD servers using Ubuntu
This page has been made to track an apparent hardware/driver incompatibility between Ubuntu 18.04 LTS and recent Dell AMD servers.

I am posting our findings and test tools here in the hope that they may help others encountering the same problems.

# Environment
* Dell EMC PowerEdge R7425 and R7415
  - PERC H330 controler with a
  - Dell THNSF8200CCSE 200Gb SATA 6Gbps MLC SFF SSD
* OS: Ubuntu 18.04 LTS server
* Kernel: 4.15.0-43-generic

# Symptoms
A server may run stably for day or weeks. Then random programs consistently fail with "Segmentation fault". A closer inspection, which may be difficult to conduct, finds massive filesystem corruption, with changes to files that should not have been changed.

# It this the problem you are experiencing?
This is only likely if you have the hardware combination described in [Environment](#Environment) section. 

If you have the hardware. You can do a clean install of Ubuntu 18.04 LTS server and use the method described in the document below to quickly provoke the error to make sure. 

[Provoking PERC H330 block device errors on a Dell AMD EPIC host](AMD-PERC-ERROR.md)