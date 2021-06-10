# HSM PKCS#11 Key and certificate generation
___

This repo provides a PKCS#11 key generation bash script `p11-keygen.sh`. This script can be used to generate keys inside an HSM as well as issuing a self-signed certificate for that key and loading it to the HSM device.

The typical use case for this script is to handle the following use case:

- A HSM is available to applications on a host.
- One or more HSM Slots are available protected by a password/pin
- No keys or self-signed certificates are present in these slots
- A HSM client is available on the host

This script may then be used to execute the following key generation process:

- Generate key pair of suitable type and key size inside the HSM
- Generate a self-signed certificate for the public key
- Install the self-signed certificate in the HSM

This script is intended to be used in a Linux host such as a docker container, but it can be installed individually and used independently.

## Requirements
This script requires that the following components are installed:
- OpenSC
- libp11
- libengine-pkcs11-openssl
- OpenSSL

This script and the tools above must be installed on a host that is connected to the HSM device and it must have a HSM client installed available through a PKCS#11 library file (such as `/usr/lib/softhsm/libsofthsm2.so` for SoftHSM 2).


## Installing supporting libraries

The following script can be used to install all necessary libraries except for openssl which is expected to already exist.

```
PKCS11TOOL_VERSION=0.21.0
apt-get update && apt-get install -y pcscd libccid libpcsclite-dev libssl-dev \
    libreadline-dev autoconf automake build-essential docbook-xsl xsltproc libtool pkg-config && \
    wget https://github.com/OpenSC/OpenSC/releases/download/${PKCS11TOOL_VERSION}/opensc-${PKCS11TOOL_VERSION}.tar.gz && \
    tar xfvz opensc-*.tar.gz && \
    cd opensc-${PKCS11TOOL_VERSION} && \
    ./bootstrap && ./configure --prefix=/usr --sysconfdir=/etc/opensc && \
    make && make install && \
    cd .. && rm -rf opensc*

LIBP11_VERSION=0.4.11
apt-get install -y libengine-pkcs11-openssl
curl -fsL https://github.com/OpenSC/libp11/releases/download/libp11-${LIBP11_VERSION}/libp11-${LIBP11_VERSION}.tar.gz \
     -o libp11-${LIBP11_VERSION}.tar.gz \
        && tar -zxf libp11-${LIBP11_VERSION}.tar.gz \
        && rm libp11-${LIBP11_VERSION}.tar.gz \
        && cd libp11-${LIBP11_VERSION} \
        && ./configure \
        && make \
        && make install \
        && rm -r /usr/src/build \
```

## Making HSM Slots

In the typical case, the HSM slots should already be pre-installed in the HSM and a slot PIN should be available that allows key generation and uploading of a self-signed certificate.

For environments, such as a test environment using SoftHSM 2, necessary slots can be created using the installed pkcs11-tool command (Available from OpenSC).

The following example creates a new HSM slot named "csca" as the first slot of the HSM :

```
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
    --init-token \
    --slot 0 \
    --so-pin SoSecrret \
    --init-pin --pin Secret \
    --label csca

```

Note that the slot number '0' must match the next available uninitialized slot index. Creating a new slot after the first slot consequently must specify slot '1'.

## Using the p11-keygen script

The script is a bash script and is executed from the command line by:

> `bash ./p11-keygen.sh [options...]`

A help menu is available by the -h or --help option:

```
> bash ./p11-keygen.sh --help
Usage: ./p11-keygen.sh [options...]

   -p, --passwd           Password for HSM slot (will be prompted for if not given)
   -s, --slot             Slot ID (Not slot index) as decimal or hex integer, for the HSM slot. Hex identifiers starts with '0x'.
   -a, --alias            The alias of the generated key
   -m, --module           PKCS11 .so library file path (default can be defined by environment variable PKCS11_MODULE)
   -i, --kid              Integer or hex key identifier (default is random generated)
   -d, --dn               Certificate subject dn (default is CN=(--alias)
       --key-type         Key type (default is EC:secp256r1)
       --hash             Must be 'sha256', 'sha384' or 'sha512' (default is sha256)
   -v  --valid-days       Certificate validity time (default is 365)
       --provider-config  Takes name of the provider as input to create a SUNPKCS11 provider configuration file. Provider configuration
                          is done per slot (not per key). No key generation if this option is selected
       --list             Show a list of available slots - No key generation
   -h, --help             Prints this help

Environment variables
   PKCS11_MODULE         Defines a default PKCS11 HSM library file location if not set by the -m or --module parameter
   LIBPKCS11             Modifies the location of the OpenSSL PKCS11 library file used for OpenSSL integration
                         If not set, this location defaults to /usr/lib/x86_64-linux-gnu/engines-1.1/libpkcs11.so


```

### List available Slots

First step to generate keys is to list available slots to determine their slot ID. this is done by the --list option.

Example showing 2 available slots:

```
bash ./p11-keygen.sh --list
Module not given, defaulting to /usr/lib/softhsm/libsofthsm2.so
Available slots:
Slot 0 (0xc3801ea): SoftHSM slot ID 0xc3801ea
  token label        : dgc_dc
  token manufacturer : SoftHSM project
  token model        : SoftHSM v2
  token flags        : login required, rng, SO PIN count low, token initialized, PIN initialized, other flags=0x20
  hardware version   : 2.4
  firmware version   : 2.4
  serial num         : 2a94286c8c3801ea
  pin min/max        : 4/255
Slot 1 (0x1d15753d): SoftHSM slot ID 0x1d15753d
  token label        : csca
  token manufacturer : SoftHSM project
  token model        : SoftHSM v2
  token flags        : login required, rng, token initialized, PIN initialized, other flags=0x20
  hardware version   : 2.4
  firmware version   : 2.4
  serial num         : 525dd6df9d15753d
  pin min/max        : 4/255
Slot 2 (0x2): SoftHSM slot ID 0x2
  token state:   uninitialized
```

The slot ID:s that must be used in the key generation process is `0xc3801ea` for slot with label `dgc_dc` and `0x1d15753d` for slot with label `csca`.

### Generating keys

Parameters for key generation is set as illustrated by the --help option.

The following example illustrates a key generation using the default settings (P-256 EC key with a self signed certificate signed using SHA-256, valid for 365 days and subject name set to CN=${alias})

>`bash ./p11-keygen.sh -s 0x1d15753d -a csca-key01`<br>

The following example illustrates an extended case where:

 - The key is an RSA 3072 bit key
 - The self signed certificate is signed using SHA-512
 - The self signed certificate is valid for 10 years
 - The DN of the certificate is set to C=SE,O=Example AB,OU=Digital Green Certificates,CN=Document signer 01


```
 bash ./p11-keygen.sh \
     -s 0x1d15753d \
     -a csca-key03 \
     --key-type RSA:3072 \
     --hash sha512 \
     -v 3652 \
     --dn "/C=SE/O=Example AB/OU=Digital Green Certificates/CN=Document signer 01"

```

Executing the examples above generates the following result message:

```
Public Key Object; EC  EC_POINT 256 bits
  EC_POINT:   044104..764cc270c193a50036bef8aa
  EC_PARAMS:  06082a8648ce3d030107
  label:      csca-key01
  ID:         121376
  Usage:      encrypt, verify, wrap, derive
  Access:     local
Certificate Object; type = X.509 cert
  label:      csca-key01
  subject:    DN: CN=csca-key01
  ID:         121376
Public Key Object; RSA 3072 bits
  label:      csca-key03
  ID:         8799
  Usage:      encrypt, verify, wrap
  Access:     local
Certificate Object; type = X.509 cert
  label:      csca-key03
  subject:    DN: C=SE, O=Example AB, OU=Digital Green Certificates, CN=Document signer 01
  ID:         8799

```

## Creating a SUN PKCS#11 provider configuration file

The script can provide a SUN PKCS#11 provider configuration file accrording to [Java 11 SUN PKCS#11 Reference Guide](https://docs.oracle.com/en/java/javase/11/security/pkcs11-reference-guide1.html#GUID-C4ABFACB-B2C9-4E71-A313-79F881488BB9).

Note that this configuratioin file is genereated per slot and not per key. Each key in a slot is accessed by its alias and the slot pin, those parameters are not included in the provider configuraiton file.

A provider configuration file for the slot in the examples above is generated by:

`bash ./p11-keygen.sh --provider-config HSM-Slot -s 0x1d15753d`

This creates a file named HSM-Slot-p11 with the following content (but with the current default PKCS#11 module lib):

```
name = HSM-Slot
library = /usr/lib/softhsm/libsofthsm2.so
slot = 487945533

```
