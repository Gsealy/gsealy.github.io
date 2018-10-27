---
title: BC添加HMAC-SM3算法支持
tags:
  - java
  - BC
  - JCE
abbrlink: fbc1d256
date: 2018-09-25 15:20:07
---

> **注：**
>
> 1. 仅适用没有java JCE证书的情况，最好还是申请一张签名证书。申请地址：[获取JCE签名证书](https://www.oracle.com/technetwork/java/javase/tech/getcodesigningcertificate-361306.html)
> 2. 请提前了解BouncyCastle轻量级加密套件
> 3. 请提前了解HMAC
> 4. 了解[ClassLoader原理](<http://blog.csdn.net/xyang81/article/details/7292380>)

------

### 一、有JCE签名证书的情况

直接修改`SM3.java`文件，添加几个方法即可。

```java
 /**
   * SM3 HashMac
   */
  public static class HashMac extends BaseMac {
    public HashMac() {
      super(new HMac(new SM3Digest()));
    }
  }

  public static class KeyGenerator extends BaseKeyGenerator {
    public KeyGenerator() {
      super("HMACSM3", 256, new CipherKeyGenerator());
    }
  }

  public static class Mappings extends DigestAlgorithmProvider {
    private static final String PREFIX = SM3.class.getName();

    public Mappings() {}

    @Override
    public void configure(ConfigurableProvider provider) {
      addHMACAlgorithm(provider, "SM3", PREFIX + "$HashMac", PREFIX + "$KeyGenerator");
      addHMACAlias(provider, "SM3", GMObjectIdentifiers.hmac_sm3);
    }
  }
```

至此，从新打包BC，签名即可使用。

### 二、无JCE签名证书，外部挂载

1. 新建一个项目，导入bcprov的jar包

2. 新建`SM3`和`DigestAlgorithmProvider`类 

3. 将BC中的相同类源码复制到新建类中，主要是`DigestAlgorithmProvider`类不是`public`的，所以需要重写一下。

4. 编写测试类

   ```java
   import java.security.MessageDigest;
   import java.security.Security;
   import javax.crypto.Mac;
   import javax.crypto.SecretKey;
   import javax.crypto.spec.SecretKeySpec;
   import org.bouncycastle.jce.provider.BouncyCastleProvider;
   import org.bouncycastle.util.encoders.Hex;
   
   public class HmacTest {
   
     static byte[] keyBytes = Hex.decode("0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b");
     static byte[] message = Hex.decode("4869205468657265");
   
     public static void main(String[] args) throws Exception {
       BouncyCastleProvider bcp = new BouncyCastleProvider();
       Security.addProvider(bcp);
       new SM3.Mappings().configure(bcp);
       for (String s : Security.getAlgorithms("Mac")) {
         if (s.contains("HMACSM3")) {
           System.out.println(true);
           break;
         }
       }
       HmacTest hmactest = new HmacTest();
       hmactest.testHMac("HMAC-SM3");
     }
   
     public void testHMac(String hmacName) throws Exception {
       SecretKey key = new SecretKeySpec(keyBytes, hmacName);
       byte[] out,out1;
       
       // SM3 
       MessageDigest sm3 = MessageDigest.getInstance("SM3", "BC");
       out1 = sm3.digest(message);
       System.out.println(Hex.toHexString(out1));
       
       // HMAC-SM3 
       Mac mac = Mac.getInstance(hmacName, "BC");
       mac.init(key);
       mac.reset();
       mac.update(message, 0, message.length);
       out = mac.doFinal();
       System.out.println(Hex.toHexString(out));
       
     }
   }
   ```

------

### 其中遇到的问题：

1. 需要了解JCE的调用过程和BC实现的相关方法

2. `DigestAlgorithmProvider`抽象类中的方法都是`protected`方法，需要在外部重写。

3. bcprov.jar不能放到java/jre/ext中，因为会loadSM3$Hmac.class，不然会报如下Exception。所以jar要和class使用相同的classloader。

   ```java
   Exception in thread "main" java.security.NoSuchAlgorithmException: class configured for Mac (provider: BC) cannot be found.
   at java.security.Provider$Service.getImplClass(Provider.java:1649)
   at java.security.Provider$Service.newInstance(Provider.java:1592)
   at sun.security.jca.GetInstance.getInstance(GetInstance.java:236)
   at javax.crypto.JceSecurity.getInstance(JceSecurity.java:103)
   at javax.crypto.Mac.getInstance(Mac.java:222)
   at cn.com.infosec.HmacSM3.HmacTest.testHMac(HmacTest.java:37)
   at cn.com.infosec.HmacSM3.HmacTest.main(HmacTest.java:30)
   Caused by: java.lang.ClassNotFoundException: com.gsealy.HmacSM3.SM3X$HashMac
   at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
   at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
   at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
   at java.security.Provider$Service.getImplClass(Provider.java:1636)
   ... 6 more
   ```

   结束！🔚

------

