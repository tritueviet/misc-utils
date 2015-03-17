# How to bypass SSL security check #
Do you always get the same nerve-racking exception when you try to get the content of a resource available through https protocol? An exception looking like this?
```
javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
    ...
Caused by: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target2009-02-17 12:49:36 - Check failed for tomcat2
    ...
Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
    ...
```

**Here is an easy solution but be warned!** The key and certificate system was not created for you to bypass it. But if you just want it to work the easy way, here is how you can do it:

```
// Imports: javax.net.ssl.TrustManager, javax.net.ssl.X509TrustManager
try {
    // Create a trust manager that does not validate certificate chains
    final TrustManager[] trustAllCerts = new TrustManager[] { new X509TrustManager() {
        @Override
        public void checkClientTrusted( final X509Certificate[] chain, final String authType ) {
        }
        @Override
        public void checkServerTrusted( final X509Certificate[] chain, final String authType ) {
        }
        @Override
        public X509Certificate[] getAcceptedIssuers() {
    	    return null;
        }
    } };
    
    // Install the all-trusting trust manager
    final SSLContext sslContext = SSLContext.getInstance( "SSL" );
    sslContext.init( null, trustAllCerts, new java.security.SecureRandom() );
    // Create an ssl socket factory with our all-trusting manager
    final SSLSocketFactory sslSocketFactory = sslContext.getSocketFactory();
    
    
    // All set up, we can get a resource through https now:
    final URLConnection urlCon = new URL( "https://someserver.yo/resource" ).openConnection();
    // Tell the url connection object to use our socket factory which bypasses security checks
    ( (HttpsURLConnection) urlCon ).setSSLSocketFactory( sslSocketFactory );
    
    final InputStream input = urlCon.getInputStream();
    int c;
    while ( ( c = input.read() ) != -1 ) {
        // Do something...
    }
    input.close();
} catch ( final Exception e ) {
    e.printStackTrace();
}
```

Alternatively you can call
```
HttpsURLConnection.setDefaultSSLSocketFactory( sc.getSocketFactory() );
```
and after that you don't have to use the `setSSLSocketFactory()` method.


## Another solution ##
You can call `XTrustProvider.install()` to turn off SSL check.
```
// Source: http://devcentral.f5.com/weblogs/joe/archive/2005/07/06/1345.aspx

/*
 * The contents of this file are subject to the "END USER LICENSE AGREEMENT FOR F5
 * Software Development Kit for iControl"; you may not use this file except in
 * compliance with the License. The License is included in the iControl
 * Software Development Kit.
 *
 * Software distributed under the License is distributed on an "AS IS"
 * basis, WITHOUT WARRANTY OF ANY KIND, either express or implied. See
 * the License for the specific language governing rights and limitations
 * under the License.
 *
 * The Original Code is iControl Code and related documentation
 * distributed by F5.
 *
 * Portions created by F5 are Copyright (C) 1996-2004 F5 Networks
 * Inc. All Rights Reserved.  iControl (TM) is a registered trademark of
 * F5 Networks, Inc.
 *
 * Alternatively, the contents of this file may be used under the terms
 * of the GNU General Public License (the "GPL"), in which case the
 * provisions of GPL are applicable instead of those above.  If you wish
 * to allow use of your version of this file only under the terms of the
 * GPL and not to allow others to use your version of this file under the
 * License, indicate your decision by deleting the provisions above and
 * replace them with the notice and other provisions required by the GPL.
 * If you do not delete the provisions above, a recipient may use your
 * version of this file under either the License or the GPL.
 */

import java.security.AccessController;
import java.security.InvalidAlgorithmParameterException;
import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.PrivilegedAction;
import java.security.Security;
import java.security.cert.X509Certificate;
  
import javax.net.ssl.ManagerFactoryParameters;
import javax.net.ssl.TrustManager;
import javax.net.ssl.TrustManagerFactorySpi;
import javax.net.ssl.X509TrustManager;

public final class XTrustProvider extends java.security.Provider {
	
	private final static String NAME = "XTrustJSSE";
	private final static String INFO = "XTrust JSSE Provider (implements trust factory with truststore validation disabled)";
	private final static double VERSION = 1.0D;
	
	public XTrustProvider() {
		super(NAME, VERSION, INFO);
		
		AccessController.doPrivileged(new PrivilegedAction() {
			public Object run() {
				put("TrustManagerFactory." + TrustManagerFactoryImpl.getAlgorithm(), TrustManagerFactoryImpl.class.getName());
				return null;
			}
		});
	}
	
	public static void install() {
		if(Security.getProvider(NAME) == null) {
			Security.insertProviderAt(new XTrustProvider(), 2);
			Security.setProperty("ssl.TrustManagerFactory.algorithm", TrustManagerFactoryImpl.getAlgorithm());
		}
	}
	
	public final static class TrustManagerFactoryImpl extends TrustManagerFactorySpi {
		public TrustManagerFactoryImpl() { }
		public static String getAlgorithm() { return "XTrust509"; }
		protected void engineInit(KeyStore keystore) throws KeyStoreException { }
		protected void engineInit(ManagerFactoryParameters mgrparams) throws InvalidAlgorithmParameterException {
			throw new InvalidAlgorithmParameterException( XTrustProvider.NAME + " does not use ManagerFactoryParameters");
		}
		
		protected TrustManager[] engineGetTrustManagers() {
			return new TrustManager[] {
				new X509TrustManager() {
					public X509Certificate[] getAcceptedIssuers() { return null; }
					public void checkClientTrusted(X509Certificate[] certs, String authType) { }
					public void checkServerTrusted(X509Certificate[] certs, String authType) { }
				}
			};
		}
	}
}
```


## Adding the certificate that is not accepted ##
As a 3rd solution you can get around this by adding the certificate that is not accepted to your keystore.
To do that, save the certificate of the server that you want to access. Import this certificate to a keystore with the following command:
```
keytool -importcert -file downloaded.cer -alias somealias -keystore keystore_file -storepass somepass
```
And tell your Java/socket factory to use this "trust store":
```
System.setProperty( "javax.net.ssl.trustStore", "keystore_file" );
System.setProperty( "javax.net.ssl.trustStorePassword", "somepass" );
```

The certificate can be saved by web browsers, or with the `openssl` program.

...or the following program does that for you (saves the certificate and adds it to the keystore; you might want to change the keystore that it uses):

Source: http://blogs.sun.com/andreas/entry/no_more_unable_to_find

```
/*
 * Copyright 2006 Sun Microsystems, Inc.  All Rights Reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 *
 *   - Redistributions of source code must retain the above copyright
 *     notice, this list of conditions and the following disclaimer.
 *
 *   - Redistributions in binary form must reproduce the above copyright
 *     notice, this list of conditions and the following disclaimer in the
 *     documentation and/or other materials provided with the distribution.
 *
 *   - Neither the name of Sun Microsystems nor the names of its
 *     contributors may be used to endorse or promote products derived
 *     from this software without specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS
 * IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO,
 * THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
 * PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
 * CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
 * EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
 * PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
 * PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
 * LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
 * SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 */

import java.io.*;
import java.net.URL;

import java.security.*;
import java.security.cert.*;

import javax.net.ssl.*;

public class InstallCert {

    public static void main(String[] args) throws Exception {
	String host;
	int port;
	char[] passphrase;
	if ((args.length == 1) || (args.length == 2)) {
	    String[] c = args[0].split(":");
	    host = c[0];
	    port = (c.length == 1) ? 443 : Integer.parseInt(c[1]);
	    String p = (args.length == 1) ? "changeit" : args[1];
	    passphrase = p.toCharArray();
	} else {
	    System.out.println("Usage: java InstallCert <host>[:port] [passphrase]");
	    return;
	}

	File file = new File("jssecacerts");
	if (file.isFile() == false) {
	    char SEP = File.separatorChar;
	    File dir = new File(System.getProperty("java.home") + SEP
		    + "lib" + SEP + "security");
	    file = new File(dir, "jssecacerts");
	    if (file.isFile() == false) {
		file = new File(dir, "cacerts");
	    }
	}
	System.out.println("Loading KeyStore " + file + "...");
	InputStream in = new FileInputStream(file);
	KeyStore ks = KeyStore.getInstance(KeyStore.getDefaultType());
	ks.load(in, passphrase);
	in.close();

	SSLContext context = SSLContext.getInstance("TLS");
	TrustManagerFactory tmf =
	    TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
	tmf.init(ks);
	X509TrustManager defaultTrustManager = (X509TrustManager)tmf.getTrustManagers()[0];
	SavingTrustManager tm = new SavingTrustManager(defaultTrustManager);
	context.init(null, new TrustManager[] {tm}, null);
	SSLSocketFactory factory = context.getSocketFactory();

	System.out.println("Opening connection to " + host + ":" + port + "...");
	SSLSocket socket = (SSLSocket)factory.createSocket(host, port);
	socket.setSoTimeout(10000);
	try {
	    System.out.println("Starting SSL handshake...");
	    socket.startHandshake();
	    socket.close();
	    System.out.println();
	    System.out.println("No errors, certificate is already trusted");
	} catch (SSLException e) {
	    System.out.println();
	    e.printStackTrace(System.out);
	}

	X509Certificate[] chain = tm.chain;
	if (chain == null) {
	    System.out.println("Could not obtain server certificate chain");
	    return;
	}

	BufferedReader reader =
		new BufferedReader(new InputStreamReader(System.in));

	System.out.println();
	System.out.println("Server sent " + chain.length + " certificate(s):");
	System.out.println();
	MessageDigest sha1 = MessageDigest.getInstance("SHA1");
	MessageDigest md5 = MessageDigest.getInstance("MD5");
	for (int i = 0; i < chain.length; i++) {
	    X509Certificate cert = chain[i];
	    System.out.println
	    	(" " + (i + 1) + " Subject " + cert.getSubjectDN());
	    System.out.println("   Issuer  " + cert.getIssuerDN());
	    sha1.update(cert.getEncoded());
	    System.out.println("   sha1    " + toHexString(sha1.digest()));
	    md5.update(cert.getEncoded());
	    System.out.println("   md5     " + toHexString(md5.digest()));
	    System.out.println();
	}

	System.out.println("Enter certificate to add to trusted keystore or 'q' to quit: [1]");
	String line = reader.readLine().trim();
	int k;
	try {
	    k = (line.length() == 0) ? 0 : Integer.parseInt(line) - 1;
	} catch (NumberFormatException e) {
	    System.out.println("KeyStore not changed");
	    return;
	}

	X509Certificate cert = chain[k];
	String alias = host + "-" + (k + 1);
	ks.setCertificateEntry(alias, cert);

	OutputStream out = new FileOutputStream("jssecacerts");
	ks.store(out, passphrase);
	out.close();

	System.out.println();
	System.out.println(cert);
	System.out.println();
	System.out.println
		("Added certificate to keystore 'jssecacerts' using alias '"
		+ alias + "'");
    }

    private static final char[] HEXDIGITS = "0123456789abcdef".toCharArray();

    private static String toHexString(byte[] bytes) {
	StringBuilder sb = new StringBuilder(bytes.length * 3);
	for (int b : bytes) {
	    b &= 0xff;
	    sb.append(HEXDIGITS[b >> 4]);
	    sb.append(HEXDIGITS[b & 15]);
	    sb.append(' ');
	}
	return sb.toString();
    }

    private static class SavingTrustManager implements X509TrustManager {

	private final X509TrustManager tm;
	private X509Certificate[] chain;

	SavingTrustManager(X509TrustManager tm) {
	    this.tm = tm;
	}

	public X509Certificate[] getAcceptedIssuers() {
	    throw new UnsupportedOperationException();
	}

	public void checkClientTrusted(X509Certificate[] chain, String authType)
		throws CertificateException {
	    throw new UnsupportedOperationException();
	}

	public void checkServerTrusted(X509Certificate[] chain, String authType)
		throws CertificateException {
	    this.chain = chain;
	    tm.checkServerTrusted(chain, authType);
	}
    }

}
```

## Final notice ##
If after all these you get an error message of
```
HelloRequest followed by an unexpected  handshake message
```
then set the following system property:
```
System.setProperty( "sun.security.ssl.allowUnsafeRenegotiation", "true" );
```