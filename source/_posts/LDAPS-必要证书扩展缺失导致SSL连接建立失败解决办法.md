---
title: 'LDAPS: 必要证书扩展缺失导致SSL连接建立失败解决办法'
date: 2020-01-03 10:19:54
tags:
- LDAP
- workaroud
---

## 问题复现

#### 环境

AD由测试部署在Windows Server 2008上面，服务端证书也是Windows签发的

------------------------------------------------------------------------------------------------------------------

***可以直接跳到后面的`解决方案`一节查看处理***

通过SSL连接LDAP时，会抛出如下异常（精简后）

```
javax.net.ssl|DEBUG|01|main|2020-01-05 13:14:47.338 CST|SSLCipher.java:437|jdk.tls.keyLimits:  entry = AES/GCM/NoPadding KeyUpdate 2^37. AES/GCM/NOPADDING:KEYUPDATE = 137438953472
....省略
Caused by: java.security.cert.CertificateException: No subject alternative names matching IP address 10.20.61.26 found
	at java.base/sun.security.util.HostnameChecker.matchIP(HostnameChecker.java:160)
	at java.base/sun.security.util.HostnameChecker.match(HostnameChecker.java:96)
	at java.base/sun.security.ssl.X509TrustManagerImpl.checkIdentity(X509TrustManagerImpl.java:463)
	at java.base/sun.security.ssl.X509TrustManagerImpl.checkIdentity(X509TrustManagerImpl.java:434)
	at java.base/sun.security.ssl.X509TrustManagerImpl.checkTrusted(X509TrustManagerImpl.java:233)
	at java.base/sun.security.ssl.X509TrustManagerImpl.checkServerTrusted(X509TrustManagerImpl.java:129)
	at java.base/sun.security.ssl.CertificateMessage$T12CertificateConsumer.checkServerCerts(CertificateMessage.java:626)
	... 26 more
```

主要就是因为检查服务端证书的特定扩展失败，证书中没有对应的扩展。`LDAPS`对应的SSL证书需要验证IP或者DNS扩展才可以

再看下SSL流的追踪

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/protocol/ldaps_wrong_handshake.png)

最后显示`未知证书`，同时也印证了上面的异常堆栈信息。从而我们知道是证书出的问题

## 问题分析

### 定位

证书检查异常，回过头去翻一下LDAPS的RFC文档，在[RFC4519]( https://tools.ietf.org/html/rfc4513#page-9)第3.1.3.服务端身份认证一节，存在三种认证方式：

- 比较DNS
- 比较IP
- 比较其他SN类型

都是提取Extensions里面的`subjectAlternativeName`(oid: `2.5.29.17`)，主要涉及`GeneralName`的`DNSName`和`iPAddress`两种类型。当证书中不存在相应扩展，或者对应扩展的类型有误，都会校验失败

## 解决方案

重新实现一个`SSLSocketFactory`，不验证证书等信息即可：

LdapsNoVerifySSLSocketFactory.java

```java
import java.io.IOException;
import java.net.InetAddress;
import java.net.Socket;
import java.net.UnknownHostException;
import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.security.Security;
import java.security.cert.CertificateException;
import java.security.cert.X509Certificate;

import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLEngine;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManager;
import javax.net.ssl.X509ExtendedTrustManager;

/**
 * Do not verify cert IP or DNS Extensions.
 *
 * @author <a href="mailto:gsealy@outlook.com">Gsealy</a>
 */
public class LdapsNoVerifySSLSocketFactory extends SSLSocketFactory {

    static {
        Security.setProperty("ssl.SocketFactory.provider",
                LdapsNoVerifySSLSocketFactory.class.getName());
    }

    private final SSLContext sslContext;

    private final SSLSocketFactory socketFactory;

    public LdapsNoVerifySSLSocketFactory() throws NoSuchAlgorithmException, KeyManagementException {
        NoVerificationTrustManager noVerificationTrustManager = new NoVerificationTrustManager();
        sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, new TrustManager[]{noVerificationTrustManager}, new SecureRandom());
        socketFactory = sslContext.getSocketFactory();
        SSLContext.setDefault(sslContext);
		}

    @Override
    public String[] getDefaultCipherSuites() {
        return socketFactory.getDefaultCipherSuites();
    }

    @Override
    public String[] getSupportedCipherSuites() {
        return socketFactory.getSupportedCipherSuites();
    }

    @Override
    public Socket createSocket(Socket s, String host, int port, boolean autoClose)
            throws IOException {
        return socketFactory.createSocket(s, host, port, autoClose);
    }

    @Override
    public Socket createSocket(String host, int port) throws IOException {
			SSLSocketFactory socketFactory = sslContext.getSocketFactory();
			return this.socketFactory.createSocket(host, port);
		}

    @Override
    public Socket createSocket(String host, int port, InetAddress localHost, int localPort)
            throws IOException, UnknownHostException {
        return socketFactory.createSocket(host, port, localHost, localPort);
    }

    @Override
    public Socket createSocket(InetAddress host, int port) throws IOException {
        return socketFactory.createSocket(host, port);
    }

    @Override
    public Socket createSocket(InetAddress address, int port, InetAddress localAddress,
            int localPort) throws IOException {
        return socketFactory.createSocket(address, port, localAddress, localPort);
    }

    static class NoVerificationTrustManager extends X509ExtendedTrustManager {
        @Override
        public void checkClientTrusted(X509Certificate[] x509Certificates, String authType,
            Socket socket) throws CertificateException {
        }

        @Override
        public void checkClientTrusted(X509Certificate[] x509Certificates, String authType,
            SSLEngine engine) throws CertificateException {
        }


        public void checkServerTrusted(X509Certificate[] x509Certificates, String authType,
            Socket socket) throws CertificateException {
        }

        @Override
        public void checkServerTrusted(X509Certificate[] x509Certificates, String authType,
            SSLEngine engine) throws CertificateException {
        }

        @Override
        public void checkClientTrusted(X509Certificate[] x509Certificates, String s)
            throws CertificateException {
        }

        @Override
        public void checkServerTrusted(X509Certificate[] x509Certificates, String s)
            throws CertificateException {
        }

        @Override
        public X509Certificate[] getAcceptedIssuers() {
            return new X509Certificate[0];
        }
    }
}
```

JNDI连接LDAP示例代码：

```java
import javax.naming.Context;
import javax.naming.NamingException;
import javax.naming.ldap.InitialLdapContext;
import javax.naming.ldap.LdapContext;

import org.apache.directory.api.ldap.model.constants.JndiPropertyConstants;

import com.sun.jndi.ldap.LdapCtxFactory;

public class LdapsJNDITest {

    public static void main(String[] args) {
			Hashtable<String, String> env = new Hashtable<>();
			String ldapURL = "ldaps://10.20.61.26:636";
			String adminName = "CN=Administrator,CN=Users,DC=aaa,DC=com";
			String adminPassword = "11111111";
			env.put(Context.INITIAL_CONTEXT_FACTORY, LdapCtxFactory.class.getName());
			env.put(Context.SECURITY_PRINCIPAL, adminName);
			env.put(Context.SECURITY_CREDENTIALS, adminPassword);
			env.put(JndiPropertyConstants.JNDI_FACTORY_SOCKET, LdapsNoVerifySSLSocketFactory.class.getName());

			try {env.put(Context.PROVIDER_URL, ldapURL);
				LdapContext ctx = new InitialLdapContext(env, null);
			} catch (NamingException e) {
				e.printStackTrace();
			}
    }

}
```

## 附：JNDI LDAP连接流程

先创建一个配置的Map，这里用的是`HashTable`，因为上下文初始化的方法签名是`hashtable`

```java
// 初始化的方法签名
javax.naming.InitialContext#InitialContext(java.util.Hashtable<?,?>)
```

若选择LDAPS，最少需要如下参数

```properties
# 初始化上下文工厂类, javax内部类
Context.INITIAL_CONTEXT_FACTORY=LdapCtxFactory.class.getName()
# 用户名
Context.SECURITY_PRINCIPAL
# 口令
Context.SECURITY_CREDENTIALS
# SSL Socket工厂类
JndiPropertyConstants.JNDI_FACTORY_SOCKET
# ldaps地址
Context.PROVIDER_URL
```

其他参数都是可以不传，因为内部有相应的判断，可以省去部分的配置，如

```java
// LdapCtx.java L2723-2742
if (envprops != null) {
	user = (String)envprops.get(Context.SECURITY_PRINCIPAL);
	passwd = envprops.get(Context.SECURITY_CREDENTIALS);
	ver = (String)envprops.get(VERSION);
	// 这里的useSsl全局变量是在前面就已经判断过了，先判断链接，也就是{@code Context.PROVIDER_URL}的scheme是什么，如果是ldaps，就会使用ssl;
    // 还存在冗余判断，当端口号是636时，默认也使用SSL
	secProtocol =
		useSsl ? "ssl" : (String)envprops.get(Context.SECURITY_PROTOCOL);
	socketFactory = (String)envprops.get(SOCKET_FACTORY);
	authMechanism =
		(String)envprops.get(Context.SECURITY_AUTHENTICATION);
	usePool = "true".equalsIgnoreCase((String)envprops.get(ENABLE_POOL));
}

// 当{@code JndiPropertyConstants.JNDI_FACTORY_SOCKET}没有配置，且使用SSL时，使用默认Sun的SSL
if (socketFactory == null) {
	socketFactory =
	"ssl".equals(secProtocol) ? DEFAULT_SSL_FACTORY : null;
}

// 身份认证方式， 通过判断是否有用户名来确定
if (authMechanism == null) {
	authMechanism = (user == null) ? "none" : "simple";
}
```

在创建SSLSocket时，是在`Connection.java`中，方法签名如下：

```java
com.sun.jndi.ldap.Connection#createSocket(String host, int port, String socketFactory,
            int connectTimeout)
```

不能通过OOP的方式建立`SSLSocket`，因为会通过反射的方式创建SSL

```java
// Connection.java L273-L278
@SuppressWarnings("unchecked")
// 这里的socketFactory就是我们自定义的socket工厂类
Class<? extends SocketFactory> socketFactoryClass =
	(Class<? extends SocketFactory>)Obj.helper.loadClass(socketFactory);
// 通过反射的方式获取默认的SSL引擎
Method getDefault =	socketFactoryClass.getMethod("getDefault", new Class<?>[]{});
SocketFactory factory = (SocketFactory) getDefault.invoke(null, new Object[]{});
```

SSLSocketFactory内会创建默认的SSLSocket，除非我们指定SSLSocketFactory

```java
// SSLSocketFactory L95
String clsName = getSecurityProperty("ssl.SocketFactory.provider");
... 初始化操作
```

所以我们在LdapsNoVerifySSLSocketFactory里面通过静态代码块初始化配置了需要加载的类

### 另一种实现

所以这里引出另一种实现方式，可以减少代码量。但是耦合度较高，那就是在JNDI初始化前，初始化SSLContext，并设置为默认

注：还是需要NoVerificationTrustManager.class(定义在了LdapsNoVerifySSLSocketFactory内部)

```java
import java.security.SecureRandom;
import java.util.Hashtable;

import javax.naming.Context;
import javax.naming.NamingException;
import javax.naming.ldap.InitialLdapContext;
import javax.naming.ldap.LdapContext;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManager;

import com.sun.jndi.ldap.LdapCtxFactory;

import cn.com.LdapsNoVerifySSLSocketFactory.NoVerificationTrustManager;

public class LdapsJNDIV2Test {

    public static void main(String[] args) throws Exception{
			NoVerificationTrustManager noVerificationTrustManager = new NoVerificationTrustManager();
			SSLContext sslContext = SSLContext.getInstance("TLS");
			sslContext.init(null, new TrustManager[]{noVerificationTrustManager}, new SecureRandom());
			SSLContext.setDefault(sslContext);
			Hashtable<String, String> env = new Hashtable<>();
			String ldapURL = "ldaps://10.20.61.26:636";
			String adminName = "CN=Administrator,CN=Users,DC=aaa,DC=com";
			String adminPassword = "11111111";
			env.put(Context.INITIAL_CONTEXT_FACTORY, LdapCtxFactory.class.getName());
			env.put(Context.SECURITY_PRINCIPAL, adminName);
			env.put(Context.SECURITY_CREDENTIALS, adminPassword);

			try {env.put(Context.PROVIDER_URL, ldapURL);
				LdapContext ctx = new InitialLdapContext(env, null);
				System.out.println(ctx.getEnvironment());
			} catch (NamingException e) {
				e.printStackTrace();
			}
    }

}
```

两种方式都可，选择适合自己的就可以啦！🔚

------

