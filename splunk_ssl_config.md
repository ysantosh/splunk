# SSL config in splunk
## Disable ssl2/ssl3 on splunk server for poodle vulnerabilities 
Use option sslVersions to perform this. This feature is available in 6.2.0 release of the splunk. 
You can simply set the following property in the server.conf  
```
[sslConfig]
sslVersions=*,-ssl2,-ssl3
...
```
this will disable ssl2 and sslv3 protocols.

server.conf also has option 
sslVersionsForClient = <versions_list>
* This is usually less critical, since SSL/TLS will always pick the highest
  version both sides support.  However, this can be used to prohibit making
  connections to remote servers that only support older protocols.

* For forwarder connections, there is a separate "sslVersions" setting in outputs.conf and inputs.conf
* For connections to SAML servers, there is a separate "sslVersions" setting in authentication.conf
* When configured in Federal Information Processing Standard (FIPS) mode, the
  "ssl3" version is always disabled, regardless of this configuration.

## Test that ssl3 is disabled or not
```
 echo "" | openssl s_client -connect hostname:8089 -ssl3
```
Output below shows ssl3 is disabled.
```
140641529673632:error:14094410:SSL routines:SSL3_READ_BYTES:sslv3 alert handshake failure:s3_pkt.c:1275:SSL alert number 40
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
SSL-Session:
    Protocol  : SSLv3
    Cipher    : 0000
```


## Create self certificate and deploy in splunk
### Generate a private key for your root certificate and CSR
1. Create a new directory and cd to it
```mkdir $SPLUNK_HOME/etc/auth/self_signed ; cd $SPLUNK_HOME/etc/auth/self_signed```
2. Create a key to sign your certificates.
```$SPLUNK_HOME/bin/splunk cmd openssl genrsa -aes256 -out myCAPrivateKey.key 2048```
3. Generate a new Certificate Signing Request (CSR) and sign it with your key myCAPrivateKey.key
```$SPLUNK_HOME/bin/splunk cmd openssl req -new -key myCAPrivateKey.key -out myCACertificate.csr```
Provide the requested certificate information-
C=US, ST=California, L= sunnyvale, O=Jet, OU=Jet, CN=test.com
4. Use the CSR myCACertificate.csr to generate the public certificate:
```$SPLUNK_HOME/bin/splunk cmd openssl x509 -req -in myCACertificate.csr -sha512 -signkey myCAPrivateKey.key -CAcreateserial -out myCACertificate.pem -days 1095
```
A new file myCACertificate.pem appears in your directory. This is the public CA certificate that you will distribute to your Splunk instances.

### Create the server certificate
Now that you have created a root certificate to serve as your CA, you must create and sign your server certificate.
1. Generate a new RSA private key for your server certificate. 
```
$SPLUNK_HOME/bin/splunk cmd openssl genrsa -aes256 -out myServerPrivateKey.key 2048
```
2: Generate a new Certificate Signing Request (CSR) and sign it with your key myServerPrivateKey.key
```$SPLUNK_HOME/bin/splunk cmd openssl req -new -key myServerPrivateKey.key -out myServerCertificate.csr```
Provide the requested certificate information-
C=US, ST=California, L= sunnyvale, O=Jet, OU=Jet, CN=test.com
3. Use the CSR myServerCertificate.csr and your CA certificate and private key to generate a server certificate.
```$SPLUNK_HOME/bin/splunk cmd openssl x509 -req -in  myServerCertificate.csr -SHA256 -CA myCACertificate.pem -CAkey myCAPrivateKey.key -CAcreateserial -out myServerCertificate.pem -days 1095
```
A new public server certificate myServerCertificate.pem appears in your directory.

### Create a single PEM file
Combine your server certificate and public certificate, in that order, into a single PEM file.
The following is an example for:
cat myServerCertificate.pem myServerPrivateKey.key myCACertificate.pem > myMainServerCertificate.pem

### Push PEM files to Splunk instances - Copy below PEM files to directory and pushed to all splunk instance
```
ls  $SPLUNK_HOME/etc/auth/self_signed 
myMainServerCertificate.pem 
myCACertificate.pem
```

### Configure Splunk to use your own certificates

Configure your indexer to use your certificates
1. Copy your server certificate and CA public certificate into an accessible folder on the indexer(s) you intend to configure. For example: $SPLUNK_HOME/etc/auth/self_signed/
Warning: If you configure inputs.conf or outputs.conf in an app directory, the password is NOT encrypted and the clear-text value remains in the file. For this reason, you may prefer to create different certificates (signed by the same root CA) to use when configuring SSL in app directories.
2. Create and Push configs from cluster-master to cluster-members (indexers) to use your certificates
$SPLUNK_HOME/etc/master_apps/_cluster/local/

3. Configure inputs.conf on the indexer(s) to use the new server certificate. In $SPLUNK_HOME/etc/master_apps/_cluster/local/inputs.conf (or in the appropriate directory of any app you are using to distribute your forwarding configuration), stanzas:
[splunktcp-ssl://9997]
disabled = 0

[SSL]
serverCert = $SPLUNK_HOME/etc/auth/self_signed/myMainServerCertificate.pem
sslPassword = ******* 		//Server cert password
requireClientCert = true

4. Your $SPLUNK_HOME/etc/master_apps/_cluster/local/server.conf should also have the following
[sslConfig]
sslRootCAPath = $SPLUNK_HOME/etc/auth/self_signed/myCACertificate.pem
serverCert = $SPLUNK_HOME/etc/auth/self_signed/myMainServerCertificate.pem
sslPassword = ******* 		//Server cert password

5. Push configs from cluster-master to cluster-members
$SPLUNK_HOME/bin/splunk apply cluster-bundle 

### Configure your Splunk deployer, cluster master (act as forwarder) to use your certificates
1. Copy your server certificate and CA public certificate into an accessible folder on the splunk instance(s) you intend to configure $SPLUNK_HOME/etc/auth/self_signed/
2. Create configuration on splunk deployer and cluster master (act as forwarders - sending internal data to indexer tier) to use your certificates under $SPLUNK_HOME/etc/system/local/

2. Define the configuration in $SPLUNK_HOME/etc/system/local/outputs.conf (or in the appropriate directory of any app you are using to distribute your forwarding configuration):
[tcpout:group1]
server=idx1.com:9997,idx2.com:9997  //Indexers
clientCert = $SPLUNK_HOME/etc/auth/self_signed/myMainServerCertificate.pem
useClientSSLCompression = true
sslPassword = *******  // CA Cert password
sslVerifyServerCert = true

3. Your $SPLUNK_HOME/etc/system/local/server.conf should also have the following 
[sslConfig]
sslRootCAPath = $SPLUNK_HOME/etc/auth/self_signed/myCACertificate.pem
serverCert = $SPLUNK_HOME/etc/auth/self_signed/myMainServerCertificate.pem
sslPassword = ******* //Server cert password

4. Restart splunkd.
```# $SPLUNK_HOME/bin/splunk restart splunkd```

### Configure your Splunk search head (act as forwarder) to use your certificates
1. Copy your server certificate and CA public certificate into an accessible folder on the search head(s) you intend to configure. For example: $SPLUNK_HOME/etc/auth/self_signed/
Warning: If you configure inputs.conf or outputs.conf in an app directory, the password is NOT encrypted and the clear-text value remains in the file. For this reason, you may prefer to create different certificates (signed by the same root CA) to use when configuring SSL in app directories.
2. Create and Push configuration from deployer to shcluster-members (search heads) to use your certificates
$SPLUNK_HOME/etc/shcluster/apps/certs_config/local

2. Define the configuration in
``` 
$SPLUNK_HOME/etc/shcluster/apps/certs_config/local/outputs.conf (distribute your forwarding configuration):
[tcpout:group1]
server=idx1.com:9997,idx2.com:9997  	//Indexers lists
clientCert = $SPLUNK_HOME/etc/auth/self_signed/myMainServerCertificate.pem
useClientSSLCompression = true
sslPassword = *******  		// CA Cert password
sslVerifyServerCert = true
```

3. Your $SPLUNK_HOME/etc/shcluster/apps/certs_config/local/server.conf should also have the following 
```[sslConfig]
sslRootCAPath = $SPLUNK_HOME/etc/auth/self_signed/myCACertificate.pem
serverCert = $SPLUNK_HOME/etc/auth/self_signed/myMainServerCertificate.pem
sslPassword = ******* 		//Server cert password
```

4. Push configuration from deployer to shcluster-members
$SPLUNK_HOME/bin/splunk apply shcluster-bundle -target <URI>:<management-port> -auth <admin username>:<password> 

Note : Whenever you add new search head to search head cluster kindly comment sslPassword line under [sslConfig] stanza from $SPLUNK_HOME/etc/system/local/server.conf file -
```[sslConfig]           
#sslPassword = ******* 	 //splunk default cert encrypted password 
```
Need to perform this because system/local has high precedence than app/local configuration which may override your own certificate password.
Save server.conf and restart splunk # $SPLUNK_HOME/bin/splunk restart splunkd
Configure your universal forwarders to use your certificates
1. Copy your client certificate and CA public certificate into an accessible folder on the splunk forwarder(s) you intend to configure $SPLUNK_HOME/etc/auth/self_signed/
2. Create configuration on splunk universal forwarders to use your certificates
$SPLUNK_HOME/etc/system/local/

2. Define the configuration in $SPLUNK_HOME/etc/system/local/outputs.conf (or in the appropriate directory of any app you are using to distribute your forwarding configuration):
```[tcpout:group1]
Server = idx1.com:9997,idx2.com:9997  	//Indexers
clientCert = $SPLUNK_HOME/etc/auth/self_signed/myMainServerCertificate.pem
useClientSSLCompression = true
sslPassword = *******  		// CA Cert password
sslVerifyServerCert = true
```
3. Your $SPLUNK_HOME/etc/system/local/server.conf should also have the following 
```[sslConfig]
sslRootCAPath = $SPLUNK_HOME/etc/auth/self_signed/myCACertificate.pem
sslPassword = ******* 		//Server cert password
```
4. Restart splunkd.
```# $SPLUNK_HOME/bin/splunk restart splunkd```

### Verify the SSL connection
Splunk query to verify SSL connection :
index=_internal source=*metrics.log* group=tcpin_connections | dedup hostname | table _time hostname version sourceIp destPort ssl





[Ref]
http://docs.splunk.com/Documentation/Splunk/7.0.2/Security/AboutsecuringyourSplunkconfigurationwithSSL
https://serverfault.com/questions/339365/how-do-i-find-ssl-enabled-ports-or-ssl-instances-on-linux-rhel-5-3
https://conf.splunk.com/session/2015/conf2015_DWaddle_DefensePointSecurity_deploying_SplunkSSLBestPractices.pdf
http://docs.splunk.com/Documentation/Splunk/7.0.2/Security/SecureSplunkWebusingasignedcertificate


