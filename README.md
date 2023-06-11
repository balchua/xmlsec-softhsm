# Xml Digital Signature with SoftHSM and OpenSSL

This repo provides all the instructions on how to use OpenSSL + SoftHSM (PKCS11) with xmlsec


## Requirements

* Install `openssl`
 
    ``` shell
    sudo apt-get install openssl -y
    ```

* Install `opensc` and `gnutls-bin` and `xmlsec1`

    ``` shell
    sudo apt-get install opensc gnutls-bin xmlsec1 -y
    ```
* Install `libpkcs11`

    ``` shell
    curl -LOsSf  https://github.com/OpenSC/libp11/releases/download/libp11-0.4.12/libp11-0.4.12.tar.gz
    tar -zxvf libp11-0.4.12.tar.gz
    cd libp11-0.4.12
    ./configure --with-pkcs11-module && make
    # install to /usr/local/lib
    sudo make install
    ```
* Install SoftHSM via source.

    ``` shell
    curl -LOsSf https://dist.opendnssec.org/source/softhsm-2.6.1.tar.gz
    tar -zxvf softhsm-2.6.1.tar.gz
    # compile the source
    cd softhsm-2.6.1
    ./configure --disable-gost
    make
    # this installs the binaries to /usr/local/bin
    sudo make install
    ```

* Check if it is installed correctly
    * Shared library
        ``` shell
        $ find /usr/local/lib -type f -iname "libsofthsm2.so" -exec file {} \; 2> /dev/null

        /usr/local/lib/softhsm/libsofthsm2.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=02a8427be4eaf80692e7a125f67aed0e197bc0d6, with debug_info, not stripped

        ```
    * The binaries

        ``` shell
        $ find /usr/local/bin/ -type f -iname "softhsm2-*" 2> /dev/null
        /usr/local/bin/softhsm2-util
        /usr/local/bin/softhsm2-dump-file
        /usr/local/bin/softhsm2-keyconv

        ```
### Initialize SoftHSM

#### Check if SoftHSM is initialized

``` shell
$ sudo softhsm2-util --show-slots
Available slots:
Slot 0
    Slot info:
        Description:      SoftHSM slot ID 0x0                                             
        Manufacturer ID:  SoftHSM project                 
        Hardware version: 2.6
        Firmware version: 2.6
        Token present:    yes
    Token info:
        Manufacturer ID:  SoftHSM project                 
        Model:            SoftHSM v2      
        Hardware version: 2.6
        Firmware version: 2.6
        Serial number:                    
        Initialized:      no
        User PIN init.:   no
        Label:                   
```        

You can see that the `Initialized:` is set to `no`.

#### Initialize SoftHSM token

For simplicity:

SO PIN to be `492940`

User PIN: `311020`

``` shell
$ softhsm2-util --init-token --free --label "my-token"
Slot 0 has a free/uninitialized token.
=== SO PIN (4-255 characters) ===
Please enter SO PIN: ******
Please reenter SO PIN: ******
=== User PIN (4-255 characters) ===
Please enter user PIN: ******
Please reenter user PIN: ******
The token has been initialized and is reassigned to slot 1580511925
```

#### Check

``` shell
softhsm2-util --show-slots
Available slots:
Slot 1580511925
    Slot info:
        Description:      SoftHSM slot ID 0x5e34b2b5                                      
        Manufacturer ID:  SoftHSM project                 
        Hardware version: 2.6
        Firmware version: 2.6
        Token present:    yes
    Token info:
        Manufacturer ID:  SoftHSM project                 
        Model:            SoftHSM v2      
        Hardware version: 2.6
        Firmware version: 2.6
        Serial number:    be1abca75e34b2b5
        Initialized:      yes
        User PIN init.:   yes
        Label:            my-token                        
Slot 1
    Slot info:
        Description:      SoftHSM slot ID 0x1                                             
        Manufacturer ID:  SoftHSM project                 
        Hardware version: 2.6
        Firmware version: 2.6
        Token present:    yes
    Token info:
        Manufacturer ID:  SoftHSM project                 
        Model:            SoftHSM v2      
        Hardware version: 2.6
        Firmware version: 2.6
        Serial number:                    
        Initialized:      no
        User PIN init.:   no
        Label:              
```

### Create public key pair

Use `pkcs11-tool` to generate key pairs.

Use the USER PIN defined above `311020`

``` shell
$ pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so -l --token-label "my-token" -k --key-type rsa:2048 --id 1001 --label "my-key"

Logging in to "my-token".
Please enter User PIN: 
Key pair generated:
Private Key Object; RSA 
  label:      my-key
  ID:         1001
  Usage:      decrypt, sign, unwrap
  Access:     sensitive, always sensitive, never extractable, local
Public Key Object; RSA 2048 bits
  label:      my-key
  ID:         1001
  Usage:      encrypt, verify, wrap
  Access:     local

```

**Take note of the key id `1001`**  You will use it to generate the 


#### Check if key is generated

```shell
$ pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so -l -t -O
Using slot 0 with a present token (0x5e34b2b5)
Logging in to "my-token".
Please enter User PIN: 
Private Key Object; RSA 
  label:      my-key
  ID:         1001
  Usage:      decrypt, sign, unwrap
  Access:     sensitive, always sensitive, never extractable, local
Public Key Object; RSA 2048 bits
  label:      my-key
  ID:         1001
  Usage:      encrypt, verify, wrap
  Access:     local
C_SeedRandom() and C_GenerateRandom():
  seems to be OK
Digests:
  all 4 digest functions seem to work
  MD5: OK
  SHA-1: OK
Signatures: not implemented
Verify (currently only for RSA)
  testing key 0 (my-key)
    RSA-X-509: OK
    RSA-PKCS: OK
    SHA1-RSA-PKCS: OK
    MD5-RSA-PKCS: OK
Decryption (currently only for RSA)
  testing key 0 (my-key)
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
 -- mechanism can't be used to decrypt, skipping
    RSA-PKCS: OK
    RSA-PKCS-OAEP: mgf not set, defaulting to MGF1-SHA256
This version of OpenSSL only supports SHA1 for OAEP, returning
    RSA-X-509: OK
No errors


```
#### Extract public key

``` shell
$ pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so -l --read-object --type pubkey --label my-key --output-file pubkey.der
Using slot 0 with a present token (0x5e34b2b5)
Logging in to "my-token".
Please enter User PIN:

# Convert the DER format to pem

$ openssl rsa -pubin -inform DER -in pubkey.der -outform PEM -out rsa01pub.pem
```

#### Check the certificate using OpenSSL

``` shell
$ openssl asn1parse -in ~/pubkey.der -inform DER
```

``` shell


# Create certificate signed by pkcs11
sudo OPENSSL_CONF=/home/thor/workspace/xmldsig-openssl/openssl.cnf openssl req -new -x509 -days 7300 -sha512 -subj "/CN=Xmlsec" -engine pkcs11 -keyform engine -key "pkcs11:token=my-token;object=my-key;type=private;pin-value=311020" -out xmlsec.pem

# Sign the doc template and embed the certificate
sudo OPENSSL_CONF=/home/thor/workspace/xmldsig-openssl/openssl.cnf xmlsec1 --sign "--privkey-openssl-engine:my-key" "pkcs11;pkcs11:token=my-token;object=my-key;pin-value=311020,xmlsec.pem" --output output-signed.xml doc.xml

# verify the signed doc and since this is a self signed cert, we use untrusted-pem
xmlsec1 --verify --untrusted-pem xmlsec.pem  output-signed.xml
```

#### Deleting privkey, pubkey

``` shell
$ pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so -l --delete-object --type pubkey --label my-key
```

#### Show the list objects

``` shell
$ pkcs11-tool --modul /usr/local/lib/softhsm/libsofthsm2.so -l -t -O
```

# References

https://illuad.fr/2022/01/30/install-softhsmv2-and-use-it-via-openssl-and-pkcs11-11.html
https://dearzubi.medium.com/integrating-softhsm-with-openssl-using-opensc-pkcs11-184c3f92e397

https://github.com/mylamour/blog/issues/80