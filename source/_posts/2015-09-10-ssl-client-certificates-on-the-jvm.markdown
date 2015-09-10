---
layout: post
title: "SSL client certificates on the JVM"
date: 2015-09-10 12:26:31 +0200
comments: true
categories:
---

## Background

The most common scenario when using SSL/TLS is the [basic handshake](https://en.wikipedia.org/wiki/Transport_Layer_Security#Basic_TLS_handshake) where the server is the only party that is authenticated with its certificate - the client remains unauthenticated. We may then connect to the server just knowing its address:

```bash
openssl s_client -connect google.com:443
```

In this post I'm going to deal with a less popular scenario - the [client-authenticated handshake](https://en.wikipedia.org/wiki/Transport_Layer_Security#Client-authenticated_TLS_handshake) - in which the client is required to present its certificate as well and use its private key.

Let's assume our secure server is `secure.server.com:443` and we already have the client's certificate in `client.crt` and the client's private key in `client.key`, both of them in the [PEM format](https://en.wikipedia.org/wiki/Privacy-enhanced_Electronic_Mail#Sample_PEM_format_x_509_cert). We can again use `s_client` to test the connection, but this time we need to present the certificate and private key:

```bash
openssl s_client -connect secure.server.com:443 -cert client.crt -key client.key
```

However, things get a little bit less straightforward on the JVM. Any secure HTTP connection on the JVM, no matter which library you use, boils down to using the `javax.net.ssl.HttpsURLConnection`, which is a part of the [Java Secure Socket Extension (JSSE)](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html).

## JSSE, key stores and trust stores

Among other stuff, JSSE has a concept of _key stores_ and _trust stores_. The former are containers for keys/certificates presented to the server, the latter let the JVM know whether a given server certificate is signed by a trusted [Certificate Authority (CA)](https://en.wikipedia.org/wiki/Transport_Layer_Security#Certificate_Authorities). The default format for both stores is JKS (which stands for _Java keystore_), but JSSE is also capable of reading the [PKCS #12](https://en.wikipedia.org/wiki/PKCS_12) format.

### Custom key store

As you may already have guessed, in order to use the aforementioned client's certificate and key, we need to store them in a keystore. We'll go for the PKCS #12 format and use `openssl` to do the necessary conversions:

```bash
openssl pkcs12 -export -out keystore.p12 -in client.crt -inkey client.key
```

Please make sure **not** to provide an empty password when `openssl` prompts you - not only is it unreasonable from the security point of view, but it will also make mysterious `NullPointerException`s fly around when you attempt to use a key store which has an empty password.

In order for the JVM to use the custom key store, you need to set the following system properties:

  ```
  -Djavax.net.ssl.keyStore=keystore.p12
  -Djavax.net.ssl.keyStoreType=pkcs12
  -Djavax.net.ssl.keyStorePassword=<password>
  ```

where `<password>` is the key store password you chose when prompted by `openssl`. You may of course set those properties at runtime by calling `System.getProperties().put(key, value)` (in Java) or `sys.props += key -> value` (in Scala).

Provided that the certificate of `secure.server.com` is signed by a trusted CA, the steps so far are enough to get up and running. However, if the server's certificate is a self-signed one, you need an additional step, which is telling JSSE to trust the self-signed certificate.

### Custom trust store

We're going to achieve this by creating a trust store containing the certificate of the CA (the untrusted one) which signed the server's certificate. But where do we take the CA's certificate from? Once again `openssl` comes to the rescue. After executing

```bash
openssl s_client -connect secure.server.com:443 -showcerts < /dev/null
```

you're going to see - among other output - a number of certificates in the PEM format, i.e. something like:

  ```
  -----BEGIN CERTIFICATE-----
  (some Base64 content)
  -----END CERTIFICATE-----
  ```

You're interested in the last certificate in the sequence, which is going to be the CA's certificate - you need to save it (including the `BEGIN/END CERTIFICATE` lines) into a file, e.g. `ca.crt`.

Now it's time to decide whether you want to import the CA's certificate into the global JSSE trust store or just to create a local trust store with a single certificate. The global trust store contains certificates of trusted CAs like VeriSign/Symantec, so it's necessary if you want to connect to most of the well-known servers like `google.com`. The tricky part is that when you tell JSSE to use a custom trust store, it won't be using the global one anymore, so you will only be able to connect to servers whose certificates are signed by the CA in the custom trust store.

Therefore, you have three options to choose from:

1. Extend the global trust store by importing the untrusted CA's certificate into it. This is the easiest solution, but you need to remember that it will affect all applications running on the given JVM, i.e. all of them will trust certificates signed by the CA in question.

2. Make a copy of the global trust store and import the CA's certificate into the copy, then use the copy as a custom trust store in your application. In this case your application will be able to connect both to the well-known servers and to `secure.server.com`.

3. Create a custom trust store with only the certificate of the untrusted CA. Your application is then only going to trust certificated signed by the selected CA and it won't be able to make a secure connection to a well-known server like `google.com`.

Let's now explore the above options in more detail.

#### 1. Extending the global trust store

The global trust store is located in `$JAVA_HOME/jre/lib/security/cacerts`. To import the `ca.crt` into the trust store, we're going to use JDK's `keytool` utility (if you have `java` in the `PATH`, you should have `keytool` as well):

```bash
keytool -import -file ca.crt -alias "CA alias of your choice" \
        -keystore $JAVA_HOME/jre/lib/security/cacerts
```

Note: the default password for the global trust store is `changeit` (yes, not the most secure one).

Since the global trust store is used by default in a JVM application, no further configuration is needed.

#### 2. Using an extended copy of the global trust store

First simply create a copy of the global trust store:

```bash
cp $JAVA_HOME/jre/lib/security/cacerts my-cacerts.jks
```

Then import `ca.crt` like in the previous case (again, the default password is `changeit`):

```bash
keytool -import -file ca.crt -alias "CA alias of your choice" -keystore my-cacerts.jks
```

Finally, you need to tell the JVM to use the custom trust store by setting the following system properties:

  ```
  -Djavax.net.ssl.trustStore=my-cacerts.jks
  -Djavax.net.ssl.trustStoreType=JKS
  -Djavax.net.ssl.trustStorePassword=changeit
  ```

#### 3. Using a single-certificate trust store

The first step here is to create a new key store (yes, a trust store is a actually a key store):

```bash
keytool -genkey -dname "cn=CLIENT" -alias truststorekey -keyalg RSA -keystore truststore.jks
```

The `cn` value in the `dname` parameter is an arbitrary name and doesn't really matter. The same applies to the `alias` parameter. And again, please remember not to set an empty password.

Then you import `ca.crt` into the newly created trust store:

```bash
keytool -import -file ca.crt -alias "CA alias of your choice" -keystore truststore.jks
```

Finally, you need to tell the JVM to use the custom trust store by setting the following system properties:

  ```
  -Djavax.net.ssl.trustStore=truststore.jks
  -Djavax.net.ssl.trustStoreType=JKS
  -Djavax.net.ssl.trustStorePassword=<password>
  ```

where `<password>` is the password you chose when creating your custom trust store.

## Summary

Hopefully, this post has shed some light on the not-so-common scenario of a secure JVM client authenticating itself with a certificate and private key. You should now be able to seamlessly implement this kind of authentication in your JVM application.
