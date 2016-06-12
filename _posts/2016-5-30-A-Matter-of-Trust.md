---
published: true
---
The JSSE [TrustManager](https://docs.oracle.com/javase/8/docs/api/javax/net/ssl/TrustManager.html) is used to establish trust with peer when making an TLS/SSL connection. The only implementation I've ever worked with is the [X509TrustManager](https://docs.oracle.com/javase/8/docs/api/javax/net/ssl/X509TrustManager.html) which uses X509 certificates to establish trust. Typically, applications are deployed with a TrustStore which contains one or more certificate chains and when making a connection, they'll accept any of the certs from that store or ones they see that are signed by certificates in the store.

## Singleton TrustManager

I recently had a requirement to create a secure connection with a remote host and I wanted to ensure that its identity was exactly the same as what was presented before. This would allow the admin to be sure that nothing had changed with the remote endpoint.

This differed from the built in support from the Java HTTP client library I was using so I wound up writing a little code to create a one-off TrustManager with access to a single certificate chain in it.

The actual TrustManager implementation is very simple:


    public class SingletonTrustManager implements X509TrustManager {

        private final X509Certificate[] expectedChain;

        public SingletonTrustManager(X509Certificate[] expected) {
            if (expected == null) throw new IllegalArgumentException("you must have an expected chain");
            this.expectedChain = expected;
        }

        @Override
        public void checkClientTrusted(java.security.cert.X509Certificate[] x509Certificates, String s)
                throws java.security.cert.CertificateException {
        }

        @Override
        public void checkServerTrusted(java.security.cert.X509Certificate[] x509Certificates, String s)
                throws java.security.cert.CertificateException {

            if (x509Certificates.length == 0) {
                throw new CertificateException("No certificates returned by the remote endpoint");
            }

            if (!Arrays.equals(expectedChain, x509Certificates)) {
                throw new CertificateException("Cert didn't match the expected chain");
            }
        }

        @Override
        public java.security.cert.X509Certificate[] getAcceptedIssuers() {
            return new java.security.cert.X509Certificate[0];
        }
    }

In the example above, the X509Certificate difference check is implemented easily by the Arrays helper class. You may also want to include a CRL check as well to see if a certificate in the chain was revoked but that could be done outside of this class. Here we only care if the cert doesn't match exactly with what we're expecting to see.

## Using the TrustManager in a HttpClient

You can install your own trust manager in the HTTP Client 4's ConnectionManager like so:

    public static HttpClientConnectionManager createConnectionManager(TrustManager tm) 
    throws NoSuchAlgorithmException, KeyManagementException {

        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, new TrustManager[] {tm}, new SecureRandom());

        Registry<ConnectionSocketFactory> registry = RegistryBuilder.<ConnectionSocketFactory>create()
                .register("https", new SSLConnectionSocketFactory(sslContext,
                                   new BrowserCompatHostnameVerifier()))
                .build();

        return new BasicHttpClientConnectionManager(registry);
    }
    
    ...
    
    HttpClient client = HttpClients.custom()
    			.setConnectionManager(createConnectionManager(mySingletonTrustManager))
                .build();


## Getting an X509 certificate chain

There are lots of examples on the web for using openssl and other tools for generating, signing, and importing a cert chain. In my scenario, I need to get the cert chain and present the details to the end user at runtime as part of the application so one approach is to make a connection with the remote server using the HttpClient that has a TrustManager configured that will record the cert chain it's presented. After the call is done, the code making the call could simply ask for the cert chain and there you have it.


    public class MyCertChainRecorder implements X509TrustManager {

        private X509Certificate[] chain;

        @Override
        public void checkClientTrusted(java.security.cert.X509Certificate[] x509Certificates, String s)
                throws java.security.cert.CertificateException {
        }

        @Override
        public void checkServerTrusted(java.security.cert.X509Certificate[] x509Certificates, String s)
                throws java.security.cert.CertificateException {
            this.chain = x509Certificates;
        }

        @Override
        public java.security.cert.X509Certificate[] getAcceptedIssuers() {
            return new java.security.cert.X509Certificate[0];
        }
        
        public X509Certificate[] getChain() { return chain; }
    }

    ...


    HttpClient client = HttpClients.custom()
    			.setConnectionManager(createConnectionManager(myCertChainRecorder))
                .build();
                
    client.execute(new HttpGet(url));
    
    X509Certificate[] chain = myCertChainRecorder.getChain();
    
## Storing a Cert Chain in a Truststore

The Java [KeyStore](https://docs.oracle.com/javase/8/docs/api/java/security/KeyStore.html) class can be used to store private keys, key pairs, or certificates or a mix of each. I like to separate them into keystore and truststore where the keystore has all of the private information for the identities I want to present and the truststore has the public identities for the peers that I trust. This is similar to how Tomcat and Jetty do it.

The KeyStore class doesn't have any calls for storing the full chain but you can store them individually. To do this, simply walk the chain and add each entry into the KeyStore perhaps using its CN as an alias.

    /**
     * Imports a chain into the keystore and returns the alias for the head of the
     * chain.
     * @param trustStore
     * @param x509Chain
     * @throws KeyStoreException
     */
    public static String importChain(KeyStore trustStore, X509Certificate[] x509Chain) 
    throws KeyStoreException {
        String baseAlias = null;
        Set<String> aliases = new HashSet<>(Collections.list(trustStore.aliases()));
        for(X509Certificate cert : x509Chain) {
            String alias = trustStore.getCertificateAlias(cert);
            if (alias == null) {
                alias = makeAlias(cert, aliases);
                aliases.add(alias);
                trustStore.setCertificateEntry(alias, cert);
            }

            // only set the endpoint for the first cert in the chain
            if (baseAlias == null) {
                baseAlias = alias;
            }
        }
        return baseAlias;
    }

    /**
     * Helper method that makes an alias for the given cert by using its CN
     * and a unique suffix if the CN alone isn't unique within the given
     * set of aliases.
     * @param cert
     * @param aliases
     */
    private static String makeAlias(X509Certificate cert, Set<String> aliases) {
        String cn = cert.getSubjectDN().toString();
        int counter = 1;
        String alias = cn;
        while(aliases.contains(alias)) {
            alias = cn + "-" + counter++;
        }
        return alias;
    }

## Loading a cert chain from the KeyStore

Loading the cert chain back seems easy since there's a call on [KeyStore](https://docs.oracle.com/javase/8/docs/api/java/security/KeyStore.html#getCertificateChain-java.lang.String-) for doing this in one step!

    public final Certificate[] getCertificateChain(String alias)
                                        throws KeyStoreException
                                        
However, upon futher review (i.e. reading the actual Javadoc duh!) I learned that this was not the right method for me. This only fetches the Public Key portion of a previously entered Private Key. This is not what we need.

Unfortunately, you have to write a little code to do this. It's basically just reading the head entry certificate from the chain using the alias you got back from the import method above and then you need to traverse into that cert's issuer until you don't find any more certs/issuers to load.

    public static X509Certificate[] readCertChain(KeyStore trustStore, String alias) 
    throws KeyStoreException {
        List<X509Certificate> chain = new ArrayList<>();
        Set<String> processedCertDN = new HashSet<>();
        X509Certificate cert = (X509Certificate) trustStore.getCertificate(alias);
        while(cert != null) {
            String dn = cert.getSubjectDN().toString();
            chain.add(cert);
            processedCertDN.add(dn);
            Principal issuerDN = cert.getIssuerDN();

            // clear the cert to stop the loop
            cert = null;

            if (issuerDN != null) {
                // there's an issuer, let's get their cert too
                // I don't think there'd be a cycle here but let's avoid one
                // anyway.
                String issuerDNString = issuerDN.toString();
                if (!processedCertDN.contains(issuerDNString)) {
                    cert = findCert(trustStore, issuerDNString);
                }
            }
        }
        X509Certificate[] retval = new X509Certificate[chain.size()];
        chain.toArray(retval);
        return retval;
    }
