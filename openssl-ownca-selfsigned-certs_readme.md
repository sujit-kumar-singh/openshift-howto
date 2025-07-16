# Example of Own CA creation, then use the CA to sign the own generate CSRs

## Create CA (This is just once in a long time)
```bash
export CANAME=MyOwnRootCA

# Generate the CA key

openssl genrsa -des -out $CANAME.key 4096

# if "-des" is not allowed then use "-des3

# The above asks for a passphrase, keep the passphrase safe.
# openssl genrsa -des3 -out $CANAME.key 4096

openssl req -x509 -new -nodes -key $CANAME.key -sha256 -days 1826 -out $CANAME.crt -subj '/C=AE/ST=Dubai/L=Dubai/O=Mycompany Corp/CN=$CANAME'
```

## Generate Key and CSR

```bash
export cn='apps.myocp.ucmcswg.com'

openssl genrsa -out $cn.key 4096

openssl req -new -subj "/C=AE/ST=Dubai/L=Dubai/O=Mycompany Corp/CN=*.$cn" -key $cn.key -out $cn.csr -addext "subjectAltName = DNS:*.$cn"

```

## Sign the CSR with the CA

## Note, passphrase stored in a file, keep the file safe.
```bash
cat MyOwnRootCA.key > passphrase.txt
```

## Sign the CSR

```bash
openssl x509 -req -passin file:passphrase.txt -in $cn.csr -out $cn.pem -CA $CANAME.crt -CAkey $CANAME.key -CAcreateserial -days 1825 -sha256
```
