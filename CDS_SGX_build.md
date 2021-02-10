# Build Signal Contact Discovery Service with SGX enclave

This document describes how to build Contact Discovery Service with SGX support. 
We will use sources from this: 
https://github.com/clostra/ContactDiscoveryService/tree/deploy-fixes branch. 

## Prerequisites

### Hardware
SGX requires support from hardware. 

These instructions were tested on system with:
 - Motherboard model: MSI Z390 A Pro
 - Motherboard firmware: 7B98v1B, 2020-08-11
 - CPU: Intel Core i9 9900K
 - OS: Ubintu 18.04 desktop, 64 bit

The same instructions should work for Ubuntu 18.04 server, and with appropriate changes (hopefully) for Ubuntu 20.04.

### Before you start
Make sure that your hardware (motherboard, firmware, CPU) supports SGX. SGX support for CPU can be checked here: https://ark.intel.com/content/www/us/en/ark.html. Motherboard SGX support can be checked on a vendor's site. 

Check the following motherboard settings: 

 - Legacy BIOS support must be switched OFF

 - Secure Boot must be switched OFF

 - CPU SGX extension support must be turned ON

 _Note:_ Not all of above settings (except CPU SGX extension support) may be required on other systems.

### Software requirements

--- To check - may be I missed something 

```
sudo apt install build-essential ocaml  ocamlbuild automake autoconf libtool wget python libssl-dev libcurl4-openssl-dev protobuf-compiler libprotobuf-dev  git cmake perl

sudo apt install dkms 

sudo apt install linux-headers-$(uname -r)

sudo apt install openjdk-11-jdk-headless

sudo apt install maven

To run we will need: 
sudo apt install redis-server

sudo apt install redis-sentinel
```

## SGX installation

Intel SGX includes 3 parts:

 - a driver 
 - platform software (PSW)
 - SGX SDK

 All SDK-related stuff is installed in /opt/intel directory (dthe river uninstall script, aesm service, sgxsdk). 

 We use Intel SGX DCAP 1.9, which can be downloaded here: https://01.org/intel-softwareguard-extensions/downloads/intel-sgx-dcap-1.9-release

 We will use the following files: driver (sgx_linux_x64_driver_1.36.2.bin) and 
 sdk (sgx_linux_x64_sdk_2.12.100.3.bin). The Intel_SGX_DCAP_Linux_SW_Installation_Guide.pdf from the doc directory contains a lot of useful (but not always correct) information.


## Install SGX Driver

```bash
chmod 777 sgx_linux_x64_driver_1.36.2.bin
sudo ./sgx_linux_x64_driver_1.36.2.bin
```

If the installation is successfull, the driver is loaded.
Check it with the command:
```bash
lsmod | grep sgx
```

Output should look like: 
```bash
intel_sgx              53248  1
```

To check sgx devices in /dev directory, run:
```bash
ls /dev/sgx/ 
enclave  provision
```
SGX driver keep the uninstall script in /opt/intel/sgxdriver directory. 

Add the current user to sgx_prv group:
```bash
sudo usermod -aG sgx_prv `whoami`
```

_NOTE:_
* If installation fails or the driver is not loaded or SGX devices are not present, stop here and try to resolve these issues. Without a working driver all the next steps do not have any sense. 

## Install Platform Software
Platform software now split on several packages, and the easiest way to install it is to use the Intel repository.

Add the repository:

```
echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu bionic main' | sudo tee /etc/apt/sources.list.d/intel-sgx.list

wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | sudo apt-key add –
sudo apt-get update
```

Install the necessary packages:

```
sudo apt install libsgx-launch 
sudo apt install libsgx-urts
sudo apt install libsgx-epid 
sudo apt install libsgx-quote-ex 
sudo apt install libsgx-dcap-ql
sudo apt install libsgx-ae-epid
sudo apt install sgx-aesm-service
```

aesmd service files installed to /opt/intel/sgx-aesm-service

Check that Architectural Enclave Service Manager (AESM) is running:

```
systemctl status aesmd
```
Service should be active and running:

```
aesmd.service - Intel(R) Architectural Enclave Service Manager
   Loaded: loaded (/lib/systemd/system/aesmd.service; enabled; vendor preset: en
   Active: active (running) since Wed 2021-01-20 09:29:13 MSK; 7h ago
  Process: 1451 ExecStart=/opt/intel/sgx-aesm-service/aesm/aesm_service (code=ex
  Process: 1449 ExecStartPre=/bin/chmod 0750 /var/opt/aesmd/ (code=exited, statu
  Process: 1437 ExecStartPre=/bin/chown -R aesmd:aesmd /var/opt/aesmd/ (code=exi
  Process: 1436 ExecStartPre=/bin/chmod 0755 /var/run/aesmd/ (code=exited, statu
  Process: 1433 ExecStartPre=/bin/chown -R aesmd:aesmd /var/run/aesmd/ (code=exi
  Process: 1426 ExecStartPre=/bin/mkdir -p /var/run/aesmd/ (code=exited, status=
  Process: 1381 ExecStartPre=/opt/intel/sgx-aesm-service/aesm/linksgx.sh (code=e
 Main PID: 1487 (aesm_service)
    Tasks: 4 (limit: 4915)
   CGroup: /system.slice/aesmd.service
           └─1487 /opt/intel/sgx-aesm-service/aesm/aesm_service
```

_NOTE:_
 * If aesmd service is not running - stop here and try to resolve this issue.


## Install SGX SDK
Make SGX SDK installation script executable and run it:
```
chmod +x sgx_linux_x64_sdk_2.12.100.3.bin
sudo ./sgx_linux_x64_sdk_2.12.100.3.bin
```

Script will ask you if you wish to install SDK in the current directory. Answer, 'n' and enter the following desired path: /opt/intel

SDK will be placed in /opt/intel/sgxsdk directory.

## Check SGX 
SGX SDK contains directory SampleCode. Copy it to a directory where you have write access. 
Enter to the RemoteAttestation subdirectory and run:

```
cd SampleCode/RemoteAttestation
source /opt/intel/sgxsdk/environment
make 
LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:sample_libcrypto;./app 
```

If program starts and prints some output with the following last lines:

```
Secret successfully received from server.
Remote attestation success!
Call enclave_ra_close success.
Enter a character before exit ...
```
then your SGX setup is working.


## Build CDS

Get the sources and checkout branch use-latest-intel-ias-auth-method:
```
git clone https://github.com/on-premise-signal/ContactDiscoveryService
cd ContactDiscoveryService/
git checkout use-latest-intel-ias-auth-method
```

There are two modifications required:

1. By default CDS builds with SGX SDK ver. 2.1.3. Let us change it to ver 2.12.
To do this use the following patch:

```patch
Index: enclave/sgx_enclave.mk
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/enclave/sgx_enclave.mk b/enclave/sgx_enclave.mk
--- a/enclave/sgx_enclave.mk	(revision ebabafb2b6befa2ec19b8340fab77c570a2d0ca7)
+++ b/enclave/sgx_enclave.mk	(date 1611152176002)
@@ -7,8 +7,8 @@
 ## linux sdk
 ##
 
-SGX_SDK_SOURCE_GIT_TAG ?= sgx_2.1.3
-SGX_SDK_SOURCE_GIT_REV ?= sgx_2.1.3-g75dd558bdaff
+SGX_SDK_SOURCE_GIT_TAG ?=sgx_2.12
+SGX_SDK_SOURCE_GIT_REV ?=sgx_2.12-gd3bd1571240b
 export SGX_SDK_SOURCE_DIR := linux-sgx-$(SGX_SDK_SOURCE_GIT_REV)
 export SGX_SDK_SOURCE_INCLUDEDIR := $(SGX_SDK_SOURCE_DIR)/common/inc
 export SGX_SDK_SOURCE_LIBDIR := $(SGX_SDK_SOURCE_DIR)/build/linux

```

2. To address undefined symbol: sgx_get_extended_epid_group_id in enclave-jni.so issue,
we need to add the explicit dependency on libsgx_epid.so library. Use the following patch: 

```patch
Index: enclave/jni/Makefile
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/enclave/jni/Makefile b/enclave/jni/Makefile
--- a/enclave/jni/Makefile	(revision ebabafb2b6befa2ec19b8340fab77c570a2d0ca7)
+++ b/enclave/jni/Makefile	(date 1610628256493)
@@ -16,9 +16,12 @@
 STATIC_SOURCES := $(srcdir)/sgxsd.c $(srcdir)/sabd_enclave_u.c
 STATIC_OBJECTS := $(STATIC_SOURCES:.c=.o)
 
+
+SGX_LIBRARY_PATH := $(SGX_SDK)/lib64
+
 SOURCES := $(srcdir)/sgxsd-jni.c
 OBJECTS := $(SOURCES:.c=.o) $(STATIC_TARGET)
-LDFLAGS := $(CFLAGS) -L../$(SGX_LIBDIR) -l$(SGX_URTS_LIB)
+LDFLAGS := $(CFLAGS) -L../$(SGX_LIBDIR) -l$(SGX_URTS_LIB) -L$(SGX_LIBRARY_PATH) -lsgx_epid
 
 .PHONY: all static
 all: $(TARGET)
```

Then build enclave:

```
make -C enclave all install
```

and then build CDS:

```
mvn package
```

## Run CDS

To run CDS you have to adjust config file, and run redis-sentinel

### Run redis sentinel

The fastest way to run Redis Sentinel for testing is to install it from the system repository:

```
sudo apt update
sudo apt install -y redis-server redis-sentinel
```

Edit Redis Server and Sentinel config files in the `/etc/redis` directory.
Or you can leave them in default state

Restart services (if you changed config files):

```
sudo systemctl restart redis-sentinel
sudo systemctl restart redis-server
```

### Adjust config file

Example of config file. Place it into `service/config/directory`. 

```
enclave:
  spid:         A6FE97EJA8FAU19JH8C83B3JAVA102CE # SPID from Intel Software Guard Extensions Attestation Service account
  iasSecretKey: f7d6fn13svsd23mqwfq9wfq721ve32fd # Secret Key from Intel Software Guard Extensions Attestation Service account
  iasHost:      https://api.trustedservices.intel.com/sgx/dev
  acceptGroupOutOfDate: true # If `True` 'Group Out of Date' error is ignored.
  instances:
    - mrenclave: 92c5382f1b904ebfa61460af52ba88ccc84ca384d9c66895a5025f9e20c334f9 # Content of the enclave/*.mrenclave file.
      debug:     false
      
signal:
  userToken:    54e12969e0d12e737078c453f5abe3f0 # Generated with command `head -c 16 /dev/urandom | hexdump -ve '1/1 "%.2x"'`
  serverToken:  a140221e0da70023a7261ead94964fc8 # backupService/userAuthenticationTokenSharedSecret from the Signal Server config
  
redis:
  masterName: mymaster # Name of the Redis Sentinel master node. Look at the /etc/redis/sentinel.conf
  sentinelUrls:
    - 127.0.0.1:6379 # Address where Redis Sentinel is listening
    - 
directory:
  initialSize: 10000000
  minLoadFactor: 0.75
  maxLoadFactor: 0.85
  sqs:
    accessKey: AKIA4LSDV0870AVADVSWIM
    accessSecret: rTHNM39lhwvwelhvw982vdslvjhsdvT0RAL
    queueUrl: https://sqs.us-east-1.amazonaws.com/92837109842/test-signal.fifo
    queueRegion: us-east-1
    
limits:
  contactQueries:
    bucketSize:        50000 # Leaky bucket size
    leakRatePerMinute: 50000 # Leaky bucket rate per minute
  remoteAttestations:
    bucketSize:        50000 # Leaky bucket size
    leakRatePerMinute: 50000 # Leaky bucket rate per minute
    
server: # Server configuration
  applicationConnectors:
    - type: http
      port: 8080 # Server will listen this port
  adminConnectors:
    - type: http
      port: 8081 # Admin interface port
```

### Run service
```
java -jar service/target/contactdiscovery-<version>.jar server service/config/yourconfig.yml
```
