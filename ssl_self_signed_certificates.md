# Self-Signed Certificates
### (extracted from [How To Secure Consul with TLS Encryption on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-secure-consul-with-tls-encryption-on-ubuntu-14-04))

Hostname            | IP Address | Role
------------------- | ---------- | --------------------
server1.example.com | 192.0.2.1  | bootstrap consul server
server2.example.com | 192.0.2.2  | consul server
server3.example.com | 192.0.2.3  | consul server
agent1.example.com  | 192.0.2.50 | consul client


## Create the SSL Structure

1. On each consul members  
  
        mkdir -p /etc/consul.d/ssl

2. On the server you plan to use a CA authority:

        mkdir -p /etc/consul.d/ssl/CA
        
    * Secure it with `chmod 0700 /etc/consul.d/ssl/CA`
    * Go to it with `cd /etc/consul.d/ssl/CA`
    
3. Create a file that will be incremented with the next available serial number for certificates. We need to pre-seed this with a value.

   Also need to provide a file where our certificate authority (we) can record the certificates that it signs. We will call this file certindex.

        echo "000a" > serial
        touch certindex

   
## Create a Self-Signed Root Certificate

1. To get started with our certificate authority, the first step we need to do is create a self-signed root certificate.

        openssl req -x509 -newkey rsa:2048 -days 3650 -nodes -out ca.cert
        
    Let's go over what this means:
    
        req: This argument tells openssl that you are interested in operating on a PKCS#10 certificate, either by creating or processing requests.
        
        -x509: This argument specifies that you would like a self-signed certificate instead of a certificate request. This is commonly done for root CA certificates.
        
        -newkey rsa:2048: This tells openssl to generate a new certificate request and private key. We pass it an argument specifying that we want an RSA key of 2048 bits.
        
        -days 3650: Here, we specify the number of days that the certificate is considered valid. We are using a value of 3650, which is 10 years.
        
        -nodes: This specifies that the generated private key will not be encrypted with DES, which would require a password. This avoids that requirement.
        
        -out ca.cert: This sets the filename that will be used for the generated certificate file.
    
2.  For the `Common Name`, which will be important in our other certificate, you can put whatever you would like.
 
        Country Name (2 letter code) [AU]:US
        State or Province Name (full name) [Some-State]:New York
        Locality Name (eg, city) []:New York City
        Organization Name (eg, company) [Internet Widgits Pty Ltd]:DigitalOcean
        Organizational Unit Name (eg, section) []:
        Common Name (e.g. server FQDN or YOUR name) []:ConsulCA
        Email Address []:admin@example.com

    When you are finished, you should have a `ca.cert` certificate file, as well as an associated key called `privkey.pem`.

## Create a Wildcard Certificate Signing Request

Now that we have the root CA certificate, we can generate a certificate signing request for our client machines.

1. In this case, all of our consul members are clients, including the server we're operating on now. Instead of generating a unique certificate for each server and signing it with our CA, we are going to create a wildcard certificate that will be valid for any of the hosts in our domain.

        openssl req -newkey rsa:1024 -nodes -out consul.csr -keyout consul.key
        
    The difference between the self-signed root CA certificate request that we created and the new certificate signing request we're generating now are here:

		no -x509 flag: We have removed the -x509 flag in order to generate a certificate signing request instead of a self-signed certificate.
		
		-out consul.csr: The outputted file is not a certificate itself, but a certificate signing request.
		
		-keyout consul.key: We have specified the name of the key that is associated with the certificate signing request.

1. Again, we will be prompted for our responses for the certificate signing request (CSR). This is more important than the answers we provided for the self-signed root CA cert.

        Common Name (e.g. server FQDN or YOUR name) []:*.example.com

	You can safely skip the challenge password and optional company name prompts that are added onto the end.
	
## Create a Certificate Authority Configuration File

			$ cat /etc/consul.d/ssl/CA/myca.conf
			[ ca ]
			default_ca = myca
			
			[ myca ]
			unique_subject = no
			new_certs_dir = .
			certificate = ca.cert
			database = certindex
			private_key = privkey.pem
			serial = serial
			default_days = 3650
			default_md = sha1
			policy = myca_policy
			x509_extensions = myca_extensions
			
			[ myca_policy ]
			commonName = supplied
			stateOrProvinceName = supplied
			countryName = supplied
			emailAddress = optional
			organizationName = supplied
			organizationalUnitName = optional
			
			[ myca_extensions ]
			basicConstraints = CA:false
			subjectKeyIdentifier = hash
			authorityKeyIdentifier = keyid:always
			keyUsage = digitalSignature,keyEncipherment
			extendedKeyUsage = serverAuth,clientAuth


## Move the Files to the Correct Location

	cp ca.cert consul.key consul.cert ../
	
	
## Done!

    "ca_file": "/etc/consul.d/ssl/ca.cert",
    "cert_file": "/etc/consul.d/ssl/consul.cert",
    "key_file": "/etc/consul.d/ssl/consul.key",
