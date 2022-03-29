# What?

- occlum-maa demo to review https://github.com/occlum/occlum/issues/797
- PR https://github.com/occlum/occlum/pull/814
- README https://github.com/occlum/occlum/blob/734828491c4cdeb47aeda7c9ebc3b7e80c88abb4/demos/remote_attestation/maa/README.md

# How?

This section lists all steps that were taken to run the given demo.

## Prepare

- Create Azure Confidential VM with Ubuntu 18.04 OS
- `ssh` to the machine



## Install Dependencies 

### Setup Intel SGX enabled platform with DCAP installed. 

Follow [DCAP Quick Install Guide](https://software.intel.com/content/www/us/en/develop/articles/intel-software-guard-extensions-data-center-attestation-primitives-quick-install-guide.html) for the detailed installation procedure.

- Subscribe to the Intel PCS for ECDSA Attestation and obtain the required API keys. (https://api.portal.trustedservices.intel.com/developer)
- Set up Intel's reference caching service, the Provisioning Certification Caching Service (PCCS).
```
sudo su
curl -o setup.sh -sL https://deb.nodesource.com/setup_14.x
chmod a+x setup.sh && ./setup.sh
```

```
apt install -y nodejs
```

```
echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' > /etc/apt/sources.list.d/intel-sgx.list
wget -O - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add -
apt update

```

```
apt install -y sqlite3 python build-essential

```

```
apt install -y cracklib-runtime

```

- Remove `RANDFILE              = $ENV::HOME/.rnd` to avoid future **ERROR:** `Can't load /opt/intel/sgx-dcap-pccs/.rnd into RNG`

```
vim /etc/ssl/openssl.cnf

```

> Remove `RANDFILE              = $ENV::HOME/.rnd` line

```
apt install -y sgx-dcap-pccs

```

> `Do you want to install PCCS now? (Y/N) :` <- Y

> `Enter your http proxy server address, e.g. http://proxy-server:port (Press ENTER if there is no proxy server) :` <- press enter

> `Enter your https proxy server address, e.g. http://proxy-server:port (Press ENTER if there is no proxy server) :` <- press enter

> `Do you want to configure PCCS now? (Y/N) :` <- Y

> `Set HTTPS listening port [8081] (1024-65535) :` <- press enter

> `Set the PCCS service to accept local connections only? [Y] (Y/N) :` <- N

- [x] Get either primary or secondary key from https://api.portal.trustedservices.intel.com/developer 

> `Set your Intel PCS API key (Press ENTER to skip) :` <- YOUR-KEY

> `Choose caching fill method : [LAZY] (LAZY/OFFLINE/REQ) :` <- REQ

> `Set PCCS server administrator password:` <- enter password

> `Re-enter administrator password:` <- repeat password

> `Set PCCS server user password:` <- enter password

> `Re-enter user password:` <- repeat password

> `Do you want to generate insecure HTTPS key and cert for PCCS service? [Y] (Y/N) :` <- Y

### Verify the empty cache

```
sqlite3 /opt/intel/sgx-dcap-pccs/pckcache.db

```

```
sqlite> .tables
crl_cache             pck_certchain         platform_tcbs
enclave_identities    pck_crl               platforms
fmspc_tcbs            pcs_certificates      platforms_registered
pck_cert              pcs_version           umzug
sqlite> select * from fmspc_tcbs;
sqlite>
sqlite> .quit
```

### Provisioning a system
```
apt install -y dkms

```

```
wget https://download.01.org/intel-sgx/sgx-dcap/1.9/linux/distro/ubuntu18.04-server/sgx_linux_x64_driver_1.36.2.bin

```

```
chmod 755 sgx_linux_x64_driver_1.36.2.bin

```

```
./sgx_linux_x64_driver_1.36.2.bin

```

```
dmesg | grep sgx

```
Output:
```
[    7.438194] sgx: EPC section 0x1c0200000-0x1c1bfffff
[    7.438410] sgx: EPC section 0x1c1e00000-0x1c1ffffff
[    7.444736] sgx: intel_sgx: Intel SGX DCAP Driver v1.33.2
[ 2609.846438] intel_sgx: loading out-of-tree module taints kernel.
[ 2609.846462] intel_sgx: module verification failed: signature and/or required key missing - tainting kernel
[ 2609.846786] intel_sgx: EPC section 0x1c0200000-0x1c1bfffff
[ 2609.847043] intel_sgx: EPC section 0x1c1e00000-0x1c1ffffff
[ 2609.851313] intel_sgx: Intel SGX DCAP Driver v1.36.2
```

```
echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' | sudo tee /etc/apt/sources.list.d/intel-sgx.list > /dev/null

```

```
wget -O - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | sudo apt-key add -

```

```
add-apt-repository ppa:ubuntu-toolchain-r/test

```

```
apt update

```

```
apt --fix-broken install
 
```

```
apt install -y sgx-pck-id-retrieval-tool

```

- [x] Make changes as specified in https://www.intel.com/content/www/us/en/developer/articles/guide/intel-software-guard-extensions-data-center-attestation-primitives-quick-install-guide.html 
```
vim /opt/intel/sgx-pck-id-retrieval-tool/network_setting.conf
```

Content of `network_setting.conf`:
```
PCCS_URL=https://localhost:8081/sgx/certification/v3/platforms
USE_SECURE_CERT=FALSE
user_token = PASSWORD-4-USER
proxy_type    = direct
```

- [x] Save the changes and run the provisioning tool
```
PCKIDRetrievalTool
```

Output:
```
Intel(R) Software Guard Extensions PCK Cert ID Retrieval Tool Version 1.12.101.1

Warning: platform manifest is not available or current platform is not multi-package platform.
the data has been sent to cache server successfully and pckid_retrieval.csv has been generated successfully!
```

### Verify the provisioned system
```
sqlite3 /opt/intel/sgx-dcap-pccs/pckcache.db

```

```
sqlite> .tables
crl_cache             pck_certchain         platform_tcbs
enclave_identities    pck_crl               platforms
fmspc_tcbs            pcs_certificates      platforms_registered
pck_cert              pcs_version           umzug
sqlite> select * from fmspc_tcbs;
00906ED50000|0|{"tcbInfo":{"version":2,"issueDate":"2022-03-25T23:00:11Z",
sqlite> .quit
```

## Run occulum demo

```
apt install -y docker.io

```

```
docker pull occlum/occlum:0.26.4-ubuntu18.04

```

### Setup Runtime Configuration

```
wget https://download.01.org/intel-sgx/sgx-dcap/1.9/linux/distro/ubuntu20.04-server/sgx_linux_x64_driver_1.36.2.bin
chmod 755 ./sgx_linux_x64_driver_1.36.2.bin
./sgx_linux_x64_driver_1.36.2.bin

```

```
echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' | sudo tee /etc/apt/sources.list.d/intel-sgx.list > /dev/null
wget -O - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | sudo apt-key add -
apt update

```

```
apt install -y libsgx-urts libsgx-dcap-ql libsgx-dcap-default-qpl

```

```
echo "default quoting type = ecdsa_256" >> /etc/aesmd.conf
```

```
systemctl restart aesmd

```

**ERROR:** `Failed to restart aesmd.service: Unit aesmd.service not found.`

```
vim /etc/sgx_default_qcnl.conf

```
 
### Get and Run Occlum

See https://github.com/occlum/occlum#how-to-use

- https://github.com/occlum/enable_rdfsbase

```
git clone https://github.com/occlum/enable_rdfsbase.git
cd enable_rdfsbase/
make
make install
./install.sh
```

For DCAP driver before v1.41:
```
docker run -it --device /dev/sgx/enclave --device /dev/sgx/provision occlum/occlum:0.26.4-ubuntu18.04
```    


```
cd /opt/intel/sgxsdk/SampleCode/SampleEnclave && make && ./app
```

```
cd ~
git clone https://github.com/qzheng527/occlum.git
cd occlum
git checkout maa
cd demos/remote_attestation/maa
./run.sh
```

### ERROR libocclum_dcap.so.0.1.0 - Fixed
```
*** Build glibc maa demo ***
make: Entering directory '/root/occlum/demos/remote_attestation/maa/gen_quote'
rm -rf gen_maa_json
make: Leaving directory '/root/occlum/demos/remote_attestation/maa/gen_quote'
make: Entering directory '/root/occlum/demos/remote_attestation/maa/gen_quote'
cc gen_maa_json.c -fPIE -pie -I /opt/intel/sgxsdk/include -I /opt/occlum/toolchains/dcap_lib/inc -L/opt/occlum/toolchains/dcap_lib/glibc -locclum_dcap -lssl -lcrypto -o gen_maa_json
make: Leaving directory '/root/occlum/demos/remote_attestation/maa/gen_quote'
/root/occlum/demos/remote_attestation/maa/occlum_instance initialized as an Occlum instance
~/occlum/demos/remote_attestation/maa/occlum_instance ~/occlum/demos/remote_attestation/maa
Enclave sign-tool: /opt/occlum/sgxsdk-tools/bin/x64/sgx_sign
Enclave sign-key: /opt/occlum/etc/template/Enclave.pem
SGX mode: HW
rm -rf /root/occlum/demos/remote_attestation/maa/occlum_instance/build
Building new image...
[+] Home dir is /root
[-] Open token file /root/enclave.token error! Will create one.
[+] Saved updated launch token!
[+] Init Enclave Successful 1726576852994!
Generate the SEFS image successfully
Building the initfs...
[+] Home dir is /root
[+] Open token file success!
[+] Token file valid!
[+] Init Enclave Successful 1803886264322!
Generate the SEFS image successfully
Building libOS...
Signing the enclave...
<EnclaveConfiguration>
    <ProdID>0</ProdID>
    <ISVSVN>0</ISVSVN>
    <StackMaxSize>1048576</StackMaxSize>
    <StackMinSize>1048576</StackMinSize>
    <HeapMaxSize>33554432</HeapMaxSize>
    <HeapMinSize>33554432</HeapMinSize>
    <TCSNum>32</TCSNum>
    <TCSPolicy>1</TCSPolicy>
    <DisableDebug>0</DisableDebug>
    <MiscSelect>0</MiscSelect>
    <MiscMask>0xFFFFFFFF</MiscMask>
    <ReservedMemMaxSize>314572800</ReservedMemMaxSize>
    <ReservedMemMinSize>314572800</ReservedMemMinSize>
    <ReservedMemInitSize>314572800</ReservedMemInitSize>
    <ReservedMemExecutable>1</ReservedMemExecutable>
    <EnableKSS>0</EnableKSS>
    <ISVEXTPRODID_H>0</ISVEXTPRODID_H>
    <ISVEXTPRODID_L>0</ISVEXTPRODID_L>
    <ISVFAMILYID_H>0</ISVFAMILYID_H>
    <ISVFAMILYID_L>0</ISVFAMILYID_L>
</EnclaveConfiguration>
tcs_num 32, tcs_max_num 32, tcs_min_pool 1
The required memory is 385511424B.
The required memory is 0x16fa7000, 376476 KB.
Succeed.
Built the Occlum image and enclave successfully
occlum run to generate quote in maa json format
gen_maa_json: error while loading shared libraries: libocclum_dcap.so.0.1.0: cannot open shared object file: No such file or directory

```
