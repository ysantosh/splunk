### SSL related troubleshooting

#### 1: Error like 
```
12-27-2022 10:40:09.842 +0000 ERROR SSLCommon [0 MainThread] - Can't read key file /splunk/etc/auth/self_signed/myMainServerCertificate.pem errno=101077092 error:06065064:digital envelope routines:EVP_DecryptFinal_ex:bad decrypt.
12-27-2022 10:40:09.876 +0000 ERROR SSLCommon [19012 MainThread] - Can't read key file /splunk/etc/auth/self_signed/myMainServerCertificate.pem errno=101077092 error:06065064:digital envelope routines:EVP_DecryptFinal_ex:bad decrypt.
12-27-2022 10:40:17.236 +0000 ERROR SSLCommon [19070 HTTPDispatch] - Can't read key file /splunk/etc/auth/self_signed/myMainServerCertificate.pem errno=101077092 error:06065064:digital envelope routines:EVP_DecryptFinal_ex:bad decrypt
```

#### Issue - In server.conf sslPassword is wrong
Reset by updating it in server.conf and restarting splunk 
```
sslPassword=<password>
```  
Note that there is no space next to '=' 
  
