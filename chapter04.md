# Chapter 04: SSL Certificates

We have our nodes provisioned, updated, keys synched, everthing. Now the next logical step is to configure Kubernetes software components on the nodes. However, we need SSL certificates first. So in this chapter, we generate SSL certificates.

# Configure / setup TLS certificates for the cluster:

Reference: [https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/02-certificate-authority.md](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/02-certificate-authority.md)


Before we start configuring various services on the nodes, we need to create the SSL/TLS certifcates, which will be used by the kubernetes components . Here I will setup a single certificate, but in production you are advised to create individual certificates for each component/service. We need to secure the following Kubernetes components:

* etcd
* Kubernetes API Server
* Kubernetes Kubelet


We will use CFSSL to create these certificates.

Linux:
```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

## Create a Certificate Authority

### Create CA CSR config file:

```
echo '{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}' > ca-config.json
```

### Generate CA certificate and CA private key:

First, create a CSR  (Certificate Signing Request) for CA:

```
echo '{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "NO",
      "L": "Oslo",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oslo"
    }
  ]
}' > ca-csr.json
```


Now, generate CA certificate and it's private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

```
[kamran@kworkhorse certs-baremetal]$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2016/09/08 11:32:54 [INFO] generating a new CA key and certificate from CSR
2016/09/08 11:32:54 [INFO] generate received request
2016/09/08 11:32:54 [INFO] received CSR
2016/09/08 11:32:54 [INFO] generating key: rsa-2048
2016/09/08 11:32:54 [INFO] encoded CSR
2016/09/08 11:32:54 [INFO] signed certificate with serial number 161389974620705926236327234344288710670396137404
[kamran@kworkhorse certs-baremetal]$ 
```

This should give you the following files:

```
ca.pem
ca-key.pem
ca.csr
```

In the list of generated files above, **ca.pem** is your CA certificate, **ca-key.pem** is the CA-certificate's private key, and **ca.csr** is the certificate signing request for this certificate.


You can verify that you have a certificate, by using the command below:

```
openssl x509 -in ca.pem -text -noout
```

It should give you the output similar to what is shown below:

```
[kamran@kworkhorse certs-baremetal]$ openssl x509 -in ca.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            1c:44:fa:0c:9d:6f:5b:66:03:cc:ac:f7:fe:b0:be:65:ab:73:9f:bc
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=NO, ST=Oslo, L=Oslo, O=Kubernetes, OU=CA, CN=Kubernetes
        Validity
            Not Before: Sep  8 09:28:00 2016 GMT
            Not After : Sep  7 09:28:00 2021 GMT
        Subject: C=NO, ST=Oslo, L=Oslo, O=Kubernetes, OU=CA, CN=Kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c4:60:18:aa:dd:71:98:00:79:63:ee:31:82:11:
                    db:26:fb:f1:74:47:7b:85:f4:b0:cf:b2:d7:ce:59:
                    26:b6:f0:01:ea:4a:b1:a0:53:ae:45:51:1c:2a:98:
                    55:00:a5:1c:07:6b:96:f9:26:84:6e:0e:23:20:07:
                    85:6a:3c:a7:9c:be:f1:b6:95:d9:6a:68:be:70:7d:
                    6b:31:c6:78:80:78:27:ed:77:f2:ef:71:3b:6b:2d:
                    66:5f:ce:71:46:16:0f:b9:e7:55:a6:e3:03:75:c4:
                    17:59:7d:61:b1:84:19:06:8d:90:0d:d9:cb:ee:72:
                    cd:a2:7f:4e:ed:37:53:fc:cc:e4:12:b8:49:ad:bf:
                    f2:0f:79:60:ea:08:9b:ed:9c:65:f8:9b:8a:81:b5:
                    cc:1e:24:bd:9c:a9:fe:68:fa:49:73:cf:b4:aa:69:
                    1c:b1:e3:6b:a5:67:89:15:e8:e1:69:af:f9:b4:4b:
                    c1:b8:33:fe:82:54:a7:fd:24:3b:18:3d:91:98:7a:
                    e5:40:0d:1a:d2:4e:1c:38:12:c4:b9:8a:7e:54:8e:
                    fe:b2:93:01:be:99:aa:18:5c:50:24:68:03:87:ec:
                    58:35:08:94:5b:b4:00:db:58:0d:e9:0f:5e:80:66:
                    c7:8b:24:bd:4b:6d:31:9c:6f:b3:a2:0c:20:bb:3b:
                    da:b1
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:2
            X509v3 Subject Key Identifier: 
                9F:0F:21:A2:F0:F1:FF:C9:19:BE:5F:4C:30:73:FD:9C:A6:C1:A0:3C
            X509v3 Authority Key Identifier: 
                keyid:9F:0F:21:A2:F0:F1:FF:C9:19:BE:5F:4C:30:73:FD:9C:A6:C1:A0:3C

    Signature Algorithm: sha256WithRSAEncryption
         0b:e0:60:9d:5c:3e:95:50:aa:6d:56:2b:83:90:83:fe:81:34:
         f2:64:e1:2d:56:13:9a:ec:13:cb:d0:fc:2f:82:3e:24:86:25:
         73:5a:79:d3:07:76:4e:0b:2e:7c:56:7e:82:e1:6e:8f:89:94:
         61:5d:20:76:31:4c:a6:f0:ad:bc:73:49:d9:81:9c:1f:6f:ad:
         ea:fd:8c:4a:c5:9c:f9:77:0a:76:c3:b7:b4:b7:dc:d4:4d:3c:
         5a:47:d6:d7:fa:07:30:34:3b:f4:4c:59:1f:4e:15:e8:11:b6:
         b6:83:61:28:a9:86:70:f9:72:cd:91:2d:c3:d6:87:37:83:04:
         74:e2:ff:67:3d:ef:bf:3b:67:88:a9:64:2b:41:72:d5:34:e5:
         93:52:2e:4a:d5:6b:8d:8c:b3:66:fa:32:18:e0:5f:9e:f1:68:
         dc:51:81:52:dc:bc:8f:01:b5:22:92:d5:5e:1c:1c:f0:a3:ab:
         a8:c5:9d:84:60:80:e4:82:52:09:1a:1c:8d:1b:af:f9:a5:66:
         06:9a:fe:f4:b1:5f:6e:51:de:49:1f:07:eb:05:3f:f1:39:cc:
         29:aa:67:b0:e6:4a:6a:dd:14:6f:41:8d:67:f7:4b:55:99:49:
         3c:4f:56:5e:a5:dd:6c:7b:2c:23:32:ee:a1:d2:0a:d4:dd:b7:
         28:86:b4:42
[kamran@kworkhorse certs-baremetal]$ 
```

## Generate the single Kubernetes TLS certificate:
**Reminder:** We will generate a TLS certificate that will be valid for all Kubernetes components. This is being done for ease of use. In production you should strongly consider generating individual TLS certificates for each component.



We should also setup an environment variable named `KUBERNETES_PUBLIC_IP_ADDRESS` with the value `10.240.0.20` . This will be handy in the next step.

Above explanation is not entirely true.


We need to setup KUBERNETES_PUBLIC_IP_ADDRESS with the IP through which we are going to access Kubernetes *from the internet*. The csr file below already has the VIP of the controller nodes listed. If you are setting up this bare-metal cluster for your corporate, then there might be a public IP which you use to access the corporate services running in your corporate DMZ. May be you can use one of those public IPs, or may be you can acquire a new public IP and assign it to the public facing interface of your edge router. So this variable here `KUBERNETES_PUBLIC_IP_ADDRESS` holds that public IP. That is what it is meant to be.



```
export KUBERNETES_PUBLIC_IP_ADDRESS='10.240.0.20'
```

### Create Kubernetes certificate CSR config file:

Be careful in creating this file. Make sure you use all the possible hostnames of the nodes you are generating this certificate for. This includes their FQDNs. When you setup node names like "nodename.example.com" then you need to include that in the CSR config file below. Also add a few extra entries for worker nodes, as you might want to increase the number of worker nodes later in this setup. So even though I have only two worker nodes right now, I have added two extra in the certificate below, worker 3 and 4. The hostnames controller.example.com and kubernetes.example.com are supposed to point to the VIP (10.240.0.20) of the controller nodes. All of these has to go into the infrastructure DNS.

**Note:** Kelsey's guide set "CN" to be "kubernetes", whereas I set it to "*.example.com" . See: [https://cabforum.org/information-for-site-owners-and-administrators/](https://cabforum.org/information-for-site-owners-and-administrators/)

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "*.example.com",
  "hosts": [
    "10.32.0.1",
    "etcd1",
    "etcd2",
    "etcd3",
    "etcd1.example.com",
    "etcd2.example.com",
    "etcd3.example.com",
    "10.240.0.11",
    "10.240.0.12",
    "10.240.0.13",
    "controller1",
    "controller2",
    "controller",
    "controller1.example.com",
    "controller2.example.com",
    "controller.example.com",
    "10.240.0.21",
    "10.240.0.21",
    "10.240.0.20",
    "worker1",
    "worker2",
    "worker3",
    "worker4",
    "worker1.example.com",
    "worker2.example.com",
    "worker3.example.com",
    "worker4.example.com",
    "10.240.0.31",
    "10.240.0.32",
    "10.240.0.33",
    "10.240.0.34",
    "lb1",
    "lb2",
    "lb",
    "lb1.example.com",
    "lb2.example.com",
    "lb.example.com",
    "10.240.0.41",
    "10.240.0.42",
    "10.240.0.40",
    "kubernetes.example.com",
    "${KUBERNETES_PUBLIC_IP_ADDRESS}",
    "localhost",
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "NO",
      "L": "Oslo",
      "O": "Kubernetes",
      "OU": "Cluster",
      "ST": "Oslo"
    }
  ]
}
EOF
```

### Generate the Kubernetes certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

```
[kamran@kworkhorse certs-baremetal]$ cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   kubernetes-csr.json | cfssljson -bare kubernetes
2016/09/08 14:04:04 [INFO] generate received request
2016/09/08 14:04:04 [INFO] received CSR
2016/09/08 14:04:04 [INFO] generating key: rsa-2048
2016/09/08 14:04:04 [INFO] encoded CSR
2016/09/08 14:04:04 [INFO] signed certificate with serial number 448428141554905058774798041748928773753703785287
2016/09/08 14:04:04 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[kamran@kworkhorse certs-baremetal]$
```

After you execute the above code, you get the following additional files:

```
kubernetes-csr.json
kubernetes-key.pem
kubernetes.pem
```

Verify the contents of the generated certificate:

```
openssl x509 -in kubernetes.pem -text -noout
```


```
[kamran@kworkhorse certs-baremetal]$ openssl x509 -in kubernetes.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            72:f8:47:b0:9c:ff:4e:f1:4e:3a:0d:5c:e9:f9:77:e9:7d:85:fd:ae
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=NO, ST=Oslo, L=Oslo, O=Kubernetes, OU=CA, CN=Kubernetes
        Validity
            Not Before: Sep  9 08:26:00 2016 GMT
            Not After : Sep  9 08:26:00 2017 GMT
        Subject: C=NO, ST=Oslo, L=Oslo, O=Kubernetes, OU=Cluster, CN=*.example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:e8:c4:01:e6:06:79:6b:b1:00:ec:7a:d4:c9:86:
                    77:f7:b2:e5:c6:e5:c8:6a:65:a1:89:d6:f6:66:09:
                    26:c3:9d:bd:39:2d:ee:eb:a8:88:d7:d9:85:3e:bf:
                    82:e0:34:83:68:70:33:6a:61:ae:c9:93:69:75:06:
                    57:da:a8:47:39:89:e1:a7:e8:72:27:89:46:6d:df:
                    fe:ed:75:99:f5:74:f0:28:22:05:f5:ac:83:af:2e:
                    e9:e0:79:0d:9b:a6:7e:71:78:90:b2:a0:14:54:92:
                    66:c1:16:e9:a2:9a:a8:4d:fb:ba:c3:22:d8:e1:f3:
                    d5:38:97:08:2b:d5:ec:1f:ba:01:9f:02:e5:7e:c9:
                    a2:a8:2d:b3:ba:33:ba:f0:61:da:ff:1a:e8:1f:61:
                    f9:1b:42:eb:f8:be:52:bf:5e:56:7d:7e:85:f7:8b:
                    01:2f:e5:c9:56:53:af:b4:87:e8:44:e2:8f:09:bf:
                    6e:85:42:4d:cb:7a:f9:f4:03:85:3f:af:b7:2e:d5:
                    58:c0:1c:62:2b:fc:b8:b7:b7:b9:d3:d3:6f:82:19:
                    89:dc:df:d9:f3:43:13:e5:e0:04:f4:8d:ce:b0:98:
                    88:81:b5:96:bb:a2:cf:90:86:f4:16:6a:34:3d:c6:
                    f7:a1:e1:2c:d4:3f:c0:b5:32:70:c1:77:2e:17:20:
                    7e:7b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier: 
                A4:9B:A2:1A:F4:AF:71:A6:2F:C7:8B:BE:83:7B:A0:DB:D3:70:91:12
            X509v3 Authority Key Identifier: 
                keyid:9F:0F:21:A2:F0:F1:FF:C9:19:BE:5F:4C:30:73:FD:9C:A6:C1:A0:3C

            X509v3 Subject Alternative Name: 
                DNS:etcd1, DNS:etcd2, DNS:etcd1.example.com, DNS:etcd2.example.com, DNS:controller1, DNS:controller2, DNS:controller1.example.com, DNS:controller2.example.com, DNS:worker1, DNS:worker2, DNS:worker3, DNS:worker4, DNS:worker1.example.com, DNS:worker2.example.com, DNS:worker3.example.com, DNS:worker4.example.com, DNS:controller.example.com, DNS:kubernetes.example.com, DNS:localhost, IP Address:10.32.0.1, IP Address:10.240.0.11, IP Address:10.240.0.12, IP Address:10.240.0.21, IP Address:10.240.0.22, IP Address:10.240.0.31, IP Address:10.240.0.32, IP Address:10.240.0.33, IP Address:10.240.0.34, IP Address:10.240.0.20, IP Address:127.0.0.1
    Signature Algorithm: sha256WithRSAEncryption
         5f:5f:cd:b0:0f:f6:7e:9d:6d:8b:ba:38:09:18:66:24:8b:4b:
         5b:71:0a:a2:b4:36:79:ae:99:5a:9b:38:07:89:05:90:53:ee:
         8c:e5:52:c9:ef:8e:1a:97:62:e7:a7:c5:70:06:6f:39:30:ba:
         32:dd:9f:72:c7:d3:09:82:4a:b6:2c:80:35:ec:e2:8f:97:dd:
         e6:34:e9:27:e6:e0:2a:9d:d9:42:94:a5:45:fe:d0:b2:30:88:
         1f:b1:5e:1c:91:a2:53:f8:6b:ad:2e:ae:b3:8a:4b:fe:aa:97:
         7d:65:2a:39:02:f8:a0:28:e8:d2:d0:bf:fb:1b:4f:57:9c:3f:
         bf:78:07:0b:c9:67:12:48:63:a2:f0:59:ff:8b:a2:10:26:d3:
         3a:0b:c3:73:85:2e:ee:14:ea:2f:1e:30:fb:78:b6:79:c9:6c:
         76:f1:fe:02:26:13:69:7c:27:74:31:21:c6:43:b5:b3:17:94:
         ed:ab:b2:05:fe:07:90:8d:6f:38:67:dc:34:6a:2d:5b:1e:f1:
         2b:b4:17:88:d6:9d:b3:0a:86:d4:0a:ad:c2:a3:bf:19:8c:99:
         74:73:be:b0:65:da:b9:cf:78:e6:14:64:ce:04:0e:48:8d:c9:
         16:c0:c7:8f:9e:9f:66:85:e6:c8:13:2e:73:20:22:35:db:ef:
         0b:cf:b6:03
[kamran@kworkhorse certs-baremetal]$ 
```

## Copy the certificates to the nodes:

```
[kamran@kworkhorse certs-baremetal]$ for node in etcd{1,2,3} controller{1,2} worker{1,2} lb{1,2}; do scp ca.pem kubernetes-key.pem kubernetes.pem  root@${node}:/root/ ; done
ca.pem                                                                                                        100% 1350     1.3KB/s   00:00    
kubernetes-key.pem                                                                                            100% 1679     1.6KB/s   00:00    
kubernetes.pem                                                                                                100% 1927     1.9KB/s   00:00    
ca.pem                                                                                                        100% 1350     1.3KB/s   00:00    
kubernetes-key.pem                                                                                            100% 1679     1.6KB/s   00:00    
kubernetes.pem                                                                                                100% 1927     1.9KB/s   00:00    
ca.pem                                                                                                        100% 1350     1.3KB/s   00:00    
kubernetes-key.pem                                                                                            100% 1679     1.6KB/s   00:00    
kubernetes.pem                                                                                                100% 1927     1.9KB/s   00:00    
ca.pem                                                                                                        100% 1350     1.3KB/s   00:00    
kubernetes-key.pem                                                                                            100% 1679     1.6KB/s   00:00    
kubernetes.pem                                                                                                100% 1927     1.9KB/s   00:00    
ca.pem                                                                                                        100% 1350     1.3KB/s   00:00    
kubernetes-key.pem                                                                                            100% 1679     1.6KB/s   00:00    
kubernetes.pem                                                                                                100% 1927     1.9KB/s   00:00    
ca.pem                                                                                                        100% 1350     1.3KB/s   00:00    
kubernetes-key.pem                                                                                            100% 1679     1.6KB/s   00:00    
kubernetes.pem                                                                                                100% 1927     1.9KB/s   00:00    
[kamran@kworkhorse certs-baremetal]$ 
```

