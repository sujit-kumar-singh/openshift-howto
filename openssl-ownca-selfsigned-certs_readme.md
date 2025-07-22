# Example of Own CA creation, then use the CA to sign the own generate CSRs

## Create CA (This is just once in a long time)
```bash

cat > passphrase.txt << EOF
3a53XXXXXXXXXXXXXXXed186
EOF

export CANAME=MyOwnRootCA

# Create CA key and Cert

openssl genrsa -des3 -out $CANAME.key -passout file:passphrase.txt 4096

openssl req -x509 -new -nodes -passin file:passphrase.txt -key $CANAME.key -sha256 -days 1826 -out $CANAME.crt -subj '/C=AE/ST=Dubai/L=Dubai/O=Mycompany Corp/CN=$CANAME'

```

## CSR Creation and Sign

Create the folder to hold the openssl config file, CSR, key and Signed certificate

```bash
export cn='apps.myocp.ucmcswg.com'
mkdir -p new_certs_dir/$cn
cd new_certs_dir/$cn

# backup files and keys if any
cd new_certs_dir/$cn/

# backup previous CSR, keys and config files

time_now=$(date +%d-%M-%Y-%H-%M-%S)

if [[ $(pwd | awk -F '/' '{print $(NF-1)"/"$(NF)}') == "new_certs_dir/$cn" ]]; then

for file in $(ls -1 | grep -v /)
  do
    echo $file
    echo y | cp -pr ${file} ${file}_${time_now}
  done
fi

# Create New openssl-ca.cnf file


export CANAME=MyOwnRootCA

dir="."

cat > openssl-ca.cnf << EOF
[ ca ]
default_ca = $CANAME

[ $CANAME ]
# Directory and file locations.
dir               = ./CA # Directory where your CA files (index.txt, serial) are
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
crlnumber         = $dir/crlnumber
crl               = $dir/crl.pem
private_key       = $dir/private/ca.key.pem # Path to your CA private key
certificate       = $dir/certs/ca.cert.pem   # Path to your CA certificate

# Hashing and encryption.
default_md        = sha256
preserve          = no
policy            = policy_match

[ policy_match ]
country           = optional
state             = optional
organization      = optional
organizationalUnit= optional
commonName        = supplied
emailAddress      = optional

[ req ]
# Parameters for req utility
default_bits        = 4096
default_md          = sha256
default_keyfile     = privkey.pem
prompt              = no
encrypt_key         = no # Do not encrypt generated keys by default
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca # for root CA signing
req_extensions      = v3_req 


[ v3_req ]
subjectAltName = @alt_names


[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = AE
stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = Dubai
localityName                    = Locality Name (eg, city)
localityName_default            = Dubai
organizationName                = Organization Name (eg, company)
organizationName_default        = Mycompany Corp
commonName                      = Common Name (e.g. server FQDN or YOUR name)
commonName_default              = Mycompany CA

[ v3_ca ]
# Extensions for a CA certificate.
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, cRLSign, digitalSignature, keyCertSign

[ usr_cert ]
# Extensions for server certificates that use the CA.
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature, nonRepudiation, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth # Add clientAuth if needed
subjectAltName = @alt_names # This is the crucial part!

[ alt_names ]
DNS.1 = *.$cn
EOF


# Generate key and CSR

openssl genrsa -out $cn.key 4096

openssl req -config openssl-ca.cnf -extensions usr_cert -new -subj "/C=AE/ST=Dubai/L=Dubai/O=Mycompany Corp/CN=*.$cn" -key $cn.key -out $cn.csr

# openssl req -new -subj "/C=AE/ST=Dubai/L=Dubai/O=Mycompany Corp/CN=*.$cn" -key $cn.key -out $cn.csr -addext "subjectAltName = DNS:*.$cn"

# Sign using CA

openssl x509 -req -passin file:../../passphrase.txt -in $cn.csr -out $cn.pem -CA ../../$CANAME.crt -CAkey ../../$CANAME.key -CAcreateserial -days 1825 -sha256 -extfile openssl-ca.cnf -extensions usr_cert

```

### Another method using openssl.cnf configuration file to create key and csr and sign using CA

If you just need to create a CSR with CN and SAN that will be later on signed by internal CA

Create openssl.cnf file like below

```bash

export san1=example.com
export san2=www.example.com
export san3=another.example.org
export san4=10.10.11.1
export cn=www.example.com

cat > openssl.cnf << EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = US
ST = State
L = City
O = Organization
OU = Department
CN = ${cn}

[v3_req]
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = $san1
DNS.2 = $san2
DNS.3 = $san3
IP.1 = $san4
EOF

```

Create the KEY and CSR to be signed together

```bash
openssl req -new -key ${san1}.key -out ${san2}.csr -config openssl.conf
```

Sign the CSR

```bash
openssl x509 -req -passin file:../../passphrase.txt -in ${san}.csr -out ${san}.pem -CA ../../$CANAME.crt -CAkey ../../$CANAME.key -CAcreateserial -days 1825 -sha256 -extfile openssl-ca.cnf -extensions usr_cert

```
