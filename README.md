# pki.bash

<img align="right" width="220" src="./logo.svg" alt="logo" />

A simple yet powerful public key infrastructure built around OpenSSL 3 with no additional dependencies,
providing TLS server authentication certificates and mTLS client authentication certificates.

## Installation

Clone this repository from GitHub:

```bash
git clone https://github.com/florian-h05/pki.bash.git
```

Make `pki.bash` executable:

```bash
chmod +x pki.bash
```

## Usage

### Setup

Before first use of the PKI, you have to set it up:

```bash
./pki.bash setup "Organization Name"
```

During the creation of the root CA and the intermediate CAs, you will be asked for several PEM passphrases.
Store them safely!

### Deployment of Root CA Certificate

The root CA's certificate has to be installed on all clients that need to trust the server certificates.
It is available in DER format as `ca/root-ca.cer` and in PEM format as `ca/root-ca.crt`.

To install a CA certificate on an operating systems, see:

- Fedora (DER, PEM): [Using Shared System Certificates :: Fedora Docs](https://docs.fedoraproject.org/en-US/quick-docs/using-shared-system-certificates/#proc_adding-new-certificates)
- Debian (PEM): [Baeldung: How to Add, Remove, and Update CA Certificates in Linux](https://www.baeldung.com/linux/ca-certificate-management#1-debian-distributions)
- Windows (DER, PEM): via Chrome browser or the management console
- iOS (DER, PEM): [Distribute certificates to Apple devices](https://support.apple.com/guide/deployment/distribute-certificates-depcdc9a6a3f/web) & [Trust manually installed certificate profiles in iOS, iPadOS, and visionOS](https://support.apple.com/en-us/102390)
- Android 10 (DER, PEM): Just click on the certificate file.

To install a CA certificate in software, see:

- Java (DER): [openHAB Docs :: Connect to InfluxDB via TLS](https://www.openhab.org/addons/persistence/influxdb/#connect-to-influxdb-via-tls) (don't wonder, the docs are about InfluxDB but that doesn't matter -- the approach is the same)
- Firefox (DER, PEM): Options -> Privacy & Security -> Certificates -> View Certificates
- Chrome (DER, PEM): Settings -> Privacy and Security -> Security -> Manage Certificates

### TLS Webserver Certificates

#### Creation of TLS Webserver Certificates

To create a certificate for a TLS webserver, use the `create_server` function:

```bash
./pki.bash create_server "hostname.local" "10.10.10.10"
```

You will be asked a number of questions, do NOT modify the organization name!
It MUST match with the organization name of the signing (and hence the root) CA.

After you have completed the CSR creation and signing process, you will find these files in the [certs/signing/](certs/signing/) folder:

- `hostname-local.cer`: The certificate in DER format - use the DER format to publish to format ([RFC 2585#section-3](https://datatracker.ietf.org/doc/html/rfc2585.html#section-3))
- `hostname-local.crt`: The certificate in PEM format.
_ `hostname-local-chain.pem`: The certificate chain containing the server certificate itself and the signing CA certificate.
- `hostname-local.key`: The private key - keep it safe!

The CSR will be located in the [reqs/signing/](/reqs/signing/) folder:

- `hostname-local.csr`: The certificate signing request (CSR) - keep it there, you need it for certificate renewal.

#### Deployment of TLS Webserver Certificates

When deploying your certificate to the server, remember to also deploy the intermediate certificate of the signing CA: `ca/signing-ca.cer`.

To deploy a webserver certificate to the server, you need these three files:

- `certs/signing/hostname-local.cer`: The server certificate itself.
- `ca/signing-ca.cer`: The intermediate certificate of the signing CA.
- `certs/signing/hostname-local.key`: The private key of the server certificate.

Note: You might also use the PEM (`.crt`) certificate versions instead of the DER (`.cer`) versions.

Alternatively, e.g. for nginx, you need instead:

- `certs/signing/hostname-local-chain.pem`: The certificate chain.
- `certs/signing/hostname-local.key`: The private key of the server certificate.

For the AVM Fritz!Box, you need the certificate chain and private key in a single PEM bundle:

```bash
cat cat certs/signing/fritz-box-chain.pem certs/signing/fritz-box.key > fritz-box.pem
```

#### Subdomain Wildcard Certificates

In case you need a wildcard certificate for a subdomain, e.g. for a reverse proxy, the procedure is very similar to the one above:

```bash
./pki.bash create_server_wildcard "subdomain.local"
```

The created certificate will be valid for both `subdomain.local` and `*.subdomain.local`.

### mTLS Client Certificate Authentication

#### Creation of mTLS Client Certificate

To create a mTLS authentication client certificate, use the `create_client` function:

```bash
./pki.bash create_client "User-Device"
```

After you have successfully created a client certificate, you need to bundle it with its private key into the PKCS#12 format:

```bash
./pki.bash build_client_p12 "User-Device"
```

or alternatively for iOS/iPadOS and Android devices:

```bash
./pki.bash build_client_p12_legacy "User-Device"
```

You will be prompted a password to encrypt the PKCS#12 bundle, which can be found in the [p12/](p12/) folder.

#### mTLS Client Certificate Authentication Server Setup

##### Creation of root & mTLS CRLs

You can create a CRL and the CRL chain for the mTLS CA using the following command:

```bash
./pki.bash create_crl "mtls"
```

This will automatically refresh the root CA CRL as this is required for the CRL chain.

You will find the CRL and the CRL chain in PEM format in the [crls/](crls/) folder, these are valid for 365 days:

- `root-ca.crl`
- `mtls-ca.crl`
- `mtls-ca-chain.crl`

##### Deployment to nginx

To enable mTLS client authentication on nginx, you need to specify these three directives either in the `http` or (more common) `server` block:

```
ssl_client_certificate    ca/mtls-chain.pem;      # The mTLS CA & root CA certificate chain in PEM format.
ssl_crl                   crl/mtls-ca-chain.crl;  # The root CA & mTLS CA CRL chain in PEM format.
ssl_verify_client         on;                     # Enables verification of client certificates.
```

Please note that you need to copy the two files above to a location where nginx can read them, and adjust the directives accordingly.
This is just an example to illustrate which file to use for what.

See [nginx: ngx_http_ssl_module](https://nginx.org/en/docs/http/ngx_http_ssl_module.html) for more information.

### General Certificate Management

#### Certificate Revocation

```bash
./pki.bash revoke caName commonName reason
```

where `reason` is one of the following: `unspecified`, `keyCompromise`, `CACompromise`, `affiliationChanged`, `superseded`, `cessationOfOperation`, `certificateHold`.

Remember to regenerate the CRL afterwards!

#### Generate a Certificate Revocation List (CRL)

```bash
./pki.bash create_crl intermediateCa
```

or for the `root` CA:

```bash
./pki.bash create_root_crl
```

If an intermediate CA CRL is generated, the root CA CRL will automatically be regenerated to properly build the CRL chain.

#### Certificate Renewal

For TLS webserver certificates:

```bash
./pki.bash renew_server "hostname.localnet"
```

For mTLS client certificates:

```
./pki.bash renew_client "User-Device"
```

Remember to regenerate the CRL afterwards!

#### View a CA Certificate

```bash
./pki.bash view_ca_cert caName
```

#### View a Single Certificate

```bash
./pki.bash view_cert caName commonName
```

#### List all Certificates of a CA

This is especially useful to check the expiration dates of the certificates of this CA.

```bash
./pki.bash view_certs_of_ca caName
```

The output format is as follows:

1. Certificate status flag (V=valid, R=revoked, E=expired).
1. Certificate expiration date in [YY]YYMMDDHHMMSSZ format.
1. Certificate revocation date in [YY]YYMMDDHHMMSSZ[,reason] format. Empty if not revoked.
1. Certificate serial number in hex.
1. Certificate filename or literal string `unknown`.
1. Certificate subject DN.

## Acknowledgments

Many thanks to Stefan Holek for his excellent [Simple PKI Tutorial](https://pki-tutorial.readthedocs.io/en/latest/simple/)!
