Instructions for how to bypass the majority of the self-signed-certificate errors that occur when developing using WSL on a MITM network.

## RHEL 
1. Download certificate on Windows as a `.cer` file
2. Copy certificate onto local WSL drive- I'll usually just put it `/~`
3. Convert `.cer` to `.crt` using `openssl`
   4. `$: openssl x509 -in your-certificate.cer -out new-certificate-name.crt`
5. Copy the new `.crt` file to the trusted certificates directory 
   6. `$: cp new-certificate-name.crt /etc/pki/ca-trust/source/anchors`
7. Update the trust store
   8. `$: sudo update-ca-trust`

(Optional)

- You can verify the cert is trusted using `openssl`: `$: sudo openssl verify -CAfile /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem /path/to/your/certificate.crt`
- Restart services like `nginx`,`httpd`, etc.