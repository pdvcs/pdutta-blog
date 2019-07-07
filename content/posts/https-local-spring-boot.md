---
title: "HTTPS for local Spring Boot development"
date: 2018-04-30T09:01:41+01:00
draft: false
tags: ["development"]
---

Cyber-security is a top-priority issue in software development ever since the early 2000s, and especially
so today. For web development in 2018, you should really be using HTTPS Everywhere -- including on your
local workstation.

This is because you almost certainly will be deploying over HTTPS in production, and using HTTPS locally
will help you catch a whole class of errors very early on -- such as the HSTS issue mentioned below.

This is a simple Spring Boot + Spring Security application sourced
[from here](https://github.com/spring-guides/gs-securing-web/tree/master/initial), to which we will add TLS
(sometimes erroneously called "SSL" even though that's a deprecated protocol) support.


## TLS with a self-signed certificate

This section is based on [this technique](https://drissamri.be/blog/java/enable-https-in-spring-boot/)
with some changes to accommodate Spring Boot 2.

Start by generating a `keystore.p12` with:
```
keytool -genkey -alias myspringboot -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore keystore.p12 -validity 3650
```

Set up a `src/main/resources/application.properties` file:

```properties
server.port: 8443
server.ssl.key-store: keystore.p12
server.ssl.key-store-password: ChangeMe
server.ssl.keyStoreType: PKCS12
server.ssl.keyAlias: myspringboot
```
... and add this code your `@SpringBootApplication` class:

```java
    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };
        tomcat.addAdditionalTomcatConnectors(redirectConnector());
        return tomcat;
    }

    private Connector redirectConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        connector.setPort(8080);
        connector.setSecure(false);
        connector.setRedirectPort(8443);
        return connector;
    }
```

This *does* work, but you get these unsightly "your connection is not private errors" in most browsers.

![Image of Chrome "your connection is not private" error](https://i.imgur.com/XzPd93a.png)

It would be much better if we could easily set up a certificate that our browser trusted -- ideally,
without paying any money.


## TLS with a trusted local certificate on Windows

This has been tested on Windows 10. Start by running this PowerShell Script to create your certificate,
and import it into the current user's trusted store of certificates.

```powershell
# LocalSSL.ps1

# setup certificate properties including the commonName (DNSName) property for Chrome 58+
$certificate = New-SelfSignedCertificate `
    -Subject localhost `
    -DnsName localhost `
    -KeyAlgorithm RSA `
    -KeyLength 2048 `
    -NotBefore (Get-Date) `
    -NotAfter (Get-Date).AddYears(3) `
    -CertStoreLocation "cert:CurrentUser\My" `
    -FriendlyName "For Development Only - TLS Certificate for localhost" `
    -HashAlgorithm SHA256 `
    -KeyUsage DigitalSignature, KeyEncipherment, DataEncipherment `
    -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.1")
$certificatePath = 'Cert:\CurrentUser\My\' + ($certificate.ThumbPrint)

# Path where certificates are created - Change this!
$tmpPath = "C:\Certificates"
If(!(test-path $tmpPath))
{
    New-Item -ItemType Directory -Force -Path $tmpPath
}

# set certificate password here
#                             Change this! ====v
$pfxPassword = ConvertTo-SecureString -String "ChangeMe" -Force -AsPlainText
$pfxFilePath = "$tmpPath\localhost.pfx"
$cerFilePath = "$tmpPath\localhost.cer"

# create pfx certificate
Export-PfxCertificate -Cert $certificatePath -FilePath $pfxFilePath -Password $pfxPassword
Export-Certificate -Cert $certificatePath -FilePath $cerFilePath

# import the pfx certificate
Import-PfxCertificate -FilePath $pfxFilePath Cert:\LocalMachine\My -Password $pfxPassword -Exportable

# trust the certificate by importing the pfx certificate into your trusted root
Import-Certificate -FilePath $cerFilePath -CertStoreLocation Cert:\CurrentUser\Root

# optionally delete the physical certificates (don't delete the pfx file as you need to copy this to your app directory)
# Remove-Item $pfxFilePath
# Remove-Item $cerFilePath
```

Tip: if you don't run PowerShell very often and need a temporary PowerShell session where you can run
unsigned scripts, type `powershell -ExecutionPolicy Bypass` at a `cmd` or `powershell` prompt. Of course,
if you use this script often, signing it is recommended!

You can view the generated `pfx` file using the JDK's `keytool` command, thus:
```
keytool -list -keystore C:\Certificates\localhost.pfx -storetype pkcs12 -storepass ChangeMe -v
```
If you just want to view the alias generated (which is usually a random UUID), use `findstr`:
```
keytool -list -keystore C:\Certificates\localhost.pfx -storetype pkcs12 -storepass ChangeMe -v | findstr /i alias
```

We can now import the generated `pfx` file into our keystore:
```
keytool -importkeystore -srckeystore C:\Certificates\localhost.pfx -srcstoretype pkcs12 ^
    -srcstorepass ChangeMe -srcalias change-this-to-whatever-alias-you-found ^
    -destkeystore keystore.p12 -deststoretype pkcs12 -deststorepass ChangeMe -destalias myspringboot
```
You can view the generated keystore thus:
```
keytool -list -keystore keystore.p12 -storetype pkcs12 -storepass ChangeMe -v
```

Ensure that your `application.properties` file has the correct values (as shown in the previous section),
and rebuild and run your application and you should be able to visit `https://localhost:8443`
and have a certificate that's recognized in Chrome and Edge. (For Firefox you'll still need to import the certificate,
unless [from Firefox 49 onwards](https://serverfault.com/questions/722563/how-to-make-firefox-trust-system-ca-certificates)
you have `security.enterprise_roots.enabled` set to `true` in about:config.)

## HSTS

In my initial testing, redirecting HTTP `http://localhost:8080` to HTTPS worked in Microsoft Edge
(v41.16299.371.0) but not in Chrome 66. Chrome complained that "localhost sent an invalid response" (`ERR_SSL_PROTOCOL_ERROR`).
The server log said:
```
2018-04-29 15:35:30.196 DEBUG 11440 --- [nio-8080-exec-1] o.a.coyote.http11.Http11InputBuffer      : Received [ <<unprintable characters>> ]
2018-04-29 15:35:30.197  INFO 11440 --- [nio-8080-exec-1] o.apache.coyote.http11.Http11Processor   : Error parsing HTTP request header
```
It turns out that Chrome was adding `localhost` to its [HSTS list](https://www.thesslstore.com/blog/clear-hsts-settings-chrome-firefox/)
because Spring Boot sent back a `Strict-Transport-Security: max-age=31536000 ; includeSubDomains` header back for
`https://localhost:8443`. Adding a `.headers().httpStrictTransportSecurity().disable();` in `WebSecurityConfig.configure` fixed the issue.
