---
title: 基于BouncyCastle，PKCS12添加CrlBag支持
tags:
  - BC
  - PKCS12
abbrlink: f01dc177
date: 2018-01-29 10:56:23
---

> BouncyCastle的KeyStore（PKCS12KeyStorespi）默认支持KeyBag、PKCS8ShroudedKeyBag、CertBag。现阶段还没有任何一个p12生成工具支持添加CrlBag的。

### 添加crlBag支持

直接对`PKCS12KeyStoreSpi.java`进行修改：

首先是`engineLoad`方法，直接在certbag的判断后添加对crlbag的判断：

```java
else if (b.getBagId().equals(crlBag)) {
                            org.bouncycastle.asn1.pkcs.CRLBag crlB =
                                    org.bouncycastle.asn1.pkcs.CRLBag.getInstance(b.getBagValue());
                            // TODO set the attributes on the key
                            X509CRL crlx509 = null;
                            try {
                                InputStream crlIn = new ByteArrayInputStream(
                                        ((ASN1OctetString) crlB.getCrlValue()).getOctets());
                                crlx509 = (X509CRL) certFact.generateCRL(crlIn);
                            } catch (Exception e) {
                                // TODO: handle exception
                                new Exception(e.toString());
                            }

                            //
                            // set the attributes
                            //
                            ASN1OctetString localId = null;
                            String alias = null;

                            if (b.getBagAttributes() != null) {
                                Enumeration e = b.getBagAttributes().getObjects();
                                while (e.hasMoreElements()) {
                                    ASN1Sequence sq = (ASN1Sequence) e.nextElement();
                                    ASN1ObjectIdentifier aOid =
                                            (ASN1ObjectIdentifier) sq.getObjectAt(0);
                                    ASN1Set attrSet = (ASN1Set) sq.getObjectAt(1);

                                    if (attrSet.size() > 0) {
                                        ASN1Primitive attr = (ASN1Primitive) attrSet.getObjectAt(0);

                                        if (crlx509 instanceof PKCS12BagAttributeCarrier) {
                                            PKCS12BagAttributeCarrier bagAttr =
                                                    (PKCS12BagAttributeCarrier) crlx509;
                                            ASN1Encodable existing = bagAttr.getBagAttribute(aOid);
                                            if (existing != null) {
                                                // OK, but the value has to be the same
                                                if (!existing.toASN1Primitive().equals(attr)) {
                                                    throw new IOException(
                                                            "attempt to add existing attribute with different value");
                                                }
                                            } else {
                                                bagAttr.setBagAttribute(aOid, attr);
                                            }
                                        }
                                        if (aOid.equals(pkcs_9_at_friendlyName)) {
                                            alias = ((DERBMPString) attr).getString();
                                            crls.put(alias, crlx509);
                                        } else if (aOid.equals(pkcs_9_at_localKeyId)) {
                                            localId = (ASN1OctetString) attr;
                                        }
                                    }
                                }
                            }

                            if (localId != null) {
                                String name = new String(Hex.encode(localId.getOctets()));

                                if (alias == null) {
                                    crls.put(name, crlx509);
                                } else {
                                    localIds.put(alias, name);
                                }
                            } else {
                                unmarkedCrl = true;
                                crls.put("unmarked", crlx509);
                            }
                        }
                    }
                }
```

在`engineLoad`中，还需要对`unmarkedCrl`判断，虽说P9扩展是可添加也可不添加，但是对pfx处理的时候，还是需要`localId`作为Key存储在`HashTable`中。

```java
if (unmarkedKey) {
　　if (keyCerts.isEmpty()) {
        String name = new String(Hex.encode(createSubjectKeyId(cert.getPublicKey()).getKeyIdentifier()));
        keyCerts.put(name, cert);
        keys.put(name, keys.remove("unmarked"));
        }
    } else if (unmarkedCrl) {
         String name = new String(Hex.encode(createSubjectKeyId(cert.getPublicKey()).getKeyIdentifier()));
         crls.put(name, crls.remove("unmarked"));
        }
```

在外部生成pfx时，我们读取一个crl文件，直接转换为X509CRL文件格式存储在Pfx文件中，然后通过`engineLoad`方法解析pfx中包含的所有内容。解析后，存储在`HashTable`和`IgnoresCaseHashtable`中。以备`doStore`方法使用。

在`doStore`中，处理CRL：  

```java
//
// handle the crl
//
ASN1EncodableVector crlSeq = new ASN1EncodableVector();
Enumeration crlbs = crls.keys();
while (crlbs.hasMoreElements()) {
	byte[] crlSalt = new byte[SALT_SIZE];
	random.nextBytes(crlSalt);
	String name = (String) crlbs.nextElement();
	X509CRL x509crl = (X509CRL) crls.get(name);
	PKCS12PBEParams crlParams = new PKCS12PBEParams(crlSalt, MIN_ITERATIONS);
	AlgorithmIdentifier crlAlgId =
	                    new AlgorithmIdentifier(keyAlgorithm, crlParams.toASN1Primitive());
	org.bouncycastle.asn1.pkcs.CRLBag crlbagInfo = null;
	try {
		crlbagInfo = new org.bouncycastle.asn1.pkcs.CRLBag(crlBag,
		                        new DEROctetString(x509crl.getEncoded()));
	}
	catch (CRLException e) {
		new CRLException(e.toString());
	}
	Boolean crlattrSet = false;
	ASN1EncodableVector crlName = new ASN1EncodableVector();
	if (x509crl instanceof PKCS12BagAttributeCarrier) {
		PKCS12BagAttributeCarrier bagAttrs = (PKCS12BagAttributeCarrier) x509crl;
		//
		// make sure we are using the local alias on store
		//
		DERBMPString nm = (DERBMPString) bagAttrs.getBagAttribute(pkcs_9_at_friendlyName);
		if (nm == null || !nm.getString().equals(name)) {
			bagAttrs.setBagAttribute(pkcs_9_at_friendlyName, new DERBMPString(name));
		}
		//
		// make sure we have a local key-id
		//
		if (bagAttrs.getBagAttribute(pkcs_9_at_localKeyId) == null) {
			Certificate ct = engineGetCertificate(name);
			bagAttrs.setBagAttribute(pkcs_9_at_localKeyId,
			                            createSubjectKeyId(ct.getPublicKey()));
		}
		Enumeration e = bagAttrs.getBagAttributeKeys();
		while (e.hasMoreElements()) {
			ASN1ObjectIdentifier oid = (ASN1ObjectIdentifier) e.nextElement();
			ASN1EncodableVector crlS = new ASN1EncodableVector();
			crlS.add(oid);
			crlS.add(new DERSet(bagAttrs.getBagAttribute(oid)));
			crlName.add(new DERSequence(crlS));
			crlattrSet = true;
		}
	}
	if (!crlattrSet) {
		//
		// set a default friendly name (from the key id) and local id
		//
		ASN1EncodableVector crlS = new ASN1EncodableVector();
		Certificate ct = engineGetCertificate(name);
		crlS.add(pkcs_9_at_localKeyId);
		crlS.add(new DERSet(createSubjectKeyId(ct.getPublicKey())));
		crlName.add(new DERSequence(crlS));
		crlS = new ASN1EncodableVector();
		crlS.add(pkcs_9_at_friendlyName);
		crlS.add(new DERSet(new DERBMPString(name)));
		crlName.add(new DERSequence(crlS));
	}
	SafeBag crlsBag =
	                    new SafeBag(crlBag, crlbagInfo.toASN1Primitive(), new DERSet(crlName));
	crlSeq.add(crlsBag);
}
byte[] CrlEncoded = new DERSequence(crlSeq).getEncoded(ASN1Encoding.DER);
BEROctetString CrlString = new BEROctetString(CrlEncoded);
```

至此，CRL基本处理完成，添加到`ContentInfo`中即可：

```java
 // safebag 按顺序放入contentinfo
 // keyBag | pkcs8ShroudedKeyBag | certBag | crlBag | secretBag | safeContentsBag
 ContentInfo[] info = new ContentInfo[] {new ContentInfo(data, keyString), 
　　　　　　　　　　　　　　　　　　　　　　　　new ContentInfo(encryptedData, cInfo.toASN1Primitive()),
　　　　　　　　　　　　　　　　　　　　　　　　new ContentInfo(data, CrlString)};
```

###  测试Pfx

编写测试类，输出一个pfx：

测试类太大了，放到Gist上地址：[PKCS12.java](https://gist.github.com/Gsealy/30704c7c098f3f8620a065cb61fd68bb)

使用`openssl`查看pfx文件

```bash
OpenSSL> pkcs12 -in D:\test.pfx -info
Enter Import Password:
MAC:sha1 Iteration 1024
PKCS7 Data
Shrouded Keybag: pbeWithSHA1And3-KeyTripleDES-CBC, Iteration 51200
Bag Attributes
    localKeyID: F3 32 9A 1E EC 9C A8 E7 87 E2 73 28 74 AC E5 A7 8A 19 C2 A4
    friendlyName: f3329a1eec9ca8e787e2732874ace5a78a19c2a4
Key Attributes: <No Attributes>
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIIC1DBOBgkqhkiG9w0BBQ0wQTApBgkqhkiG9w0BBQwwHAQIXykO/vFWOcMCAggA
MAwGCCqGSIb3DQIJBQAwFAYIKoZIhvcNAwcECMtSfWOttU3iBIICgBd5cr9SQN9l
jdxNRqgqeb+Q8seSo2sDQVbhIggc/kUQgDuXgR71kNxnlqU/qtZul/DWZbCZTgGX
2V/vRO9bd3Y4/YNuxWBDRqxsD3rWWq0YNrkKyh0pApW/R5t1/AeQgADCSOzN5FLz
fOLN76ZbDpxtQVitt09wu1I1F8ui7QicS3kwiCV6TQZHyoere7dv1QYt5mGmZnj9
luPRB6TcXMVKRnGYZki4AVx9Yc7XudC9pd5QPSlD4wJj3gap1mreOqvryxbU4dcl
YLh+itTgeB4DIRzhj3liooJic+iNuICyULPpr3Tqj9JKbQFkICxZpQGhZ8L/AuXu
aBC1jNdSLaMb+7PtCCXcH667A2zyUuD+/LLwqnvpfHFrYeelyFOY5nKC4UgsueHP
t+MHldLYeJh/EzqoulbdTNTz90MocOgPgc6TBkfwlaiWOiLfLkFjkDjF1P64RxM9
PX1KoQDrsgIA2bEa28ZG0qUx/I6ENz9Nn/9IOV0uyWxct/WCEpdDIUSBJHbVHvX4
Nm+3rfOdK3av4vJTqGzaqRAGZQ7iAoHfOGhlnq6T1Q/Xs6SjCnLM7nARh0o0JE3q
B7HDs2miXVzjZvYaT5or5tIsYm3WpLybx1gLtw4Bi98srnLQwuAQiQcwWL65+OT/
F9fBr3rQOzgQ8fauo/s6YWX/qwMuGPeK4KpZ7/F36rUGVzd5SiehZ0LIUJo+t6te
iHT6q3JD+lADojXD5+Dvf5zGM35nnf+VjwnriBveVppenCm/nshwxo/gHWoBIRFL
Q/VI3//vkl5mrfHyrt8fBCsp1mgkR3E9dL9lFKt3vw615bJm7olgWKvLjXHqyrPD
gD1mYgrCH+E=
-----END ENCRYPTED PRIVATE KEY-----
PKCS7 Encrypted data: pbeWithSHA1And40BitRC2-CBC, Iteration 51200
Certificate bag
Bag Attributes
    localKeyID: F3 32 9A 1E EC 9C A8 E7 87 E2 73 28 74 AC E5 A7 8A 19 C2 A4
    friendlyName: f3329a1eec9ca8e787e2732874ace5a78a19c2a4
subject=/C=AU/O=The Legion of the Bouncy Castle/L=Melbourne/CN=Eric H. Echidna/emailAddress=feedback-crypto@bouncycastle.org
issuer=/C=AU/O=The Legion of the Bouncy Castle/OU=Bouncy Intermediate Certificate/emailAddress=feedback-crypto@bouncycastle.org
-----BEGIN CERTIFICATE-----
MIIC4jCCAkugAwIBAgIBAzANBgkqhkiG9w0BAQUFADCBkjELMAkGA1UEBhMCQVUx
KDAmBgNVBAoMH1RoZSBMZWdpb24gb2YgdGhlIEJvdW5jeSBDYXN0bGUxKDAmBgNV
BAsMH0JvdW5jeSBJbnRlcm1lZGlhdGUgQ2VydGlmaWNhdGUxLzAtBgkqhkiG9w0B
CQEWIGZlZWRiYWNrLWNyeXB0b0Bib3VuY3ljYXN0bGUub3JnMB4XDTE3MTIzMDAy
MDU0NFoXDTE4MDIyODAyMDU0NFowgZYxCzAJBgNVBAYTAkFVMSgwJgYDVQQKDB9U
aGUgTGVnaW9uIG9mIHRoZSBCb3VuY3kgQ2FzdGxlMRIwEAYDVQQHDAlNZWxib3Vy
bmUxGDAWBgNVBAMMD0VyaWMgSC4gRWNoaWRuYTEvMC0GCSqGSIb3DQEJARYgZmVl
ZGJhY2stY3J5cHRvQGJvdW5jeWNhc3RsZS5vcmcwgZ8wDQYJKoZIhvcNAQEBBQAD
gY0AMIGJAoGBAIxalHTxlstm+f3wwL7R9LmTKqz3VGQsMxQU0uLybDMBgeWdpatm
yvHud+0oOzrfwaGzcduRUx7+0B1cnzMCvM3snVxUcGJmH/gcF5+pBXOPIbBlfyKY
gwnUx/B4QyOxZKkoZ93yf/fhlldWNkWwjfN3YqSGGpPie8nWCSPX0iPNAgMBAAGj
QjBAMB0GA1UdDgQWBBTzMpoe7Jyo54ficyh0rOWnihnCpDAfBgNVHSMEGDAWgBTz
Mpoe7Jyo54ficyh0rOWnihnCpDANBgkqhkiG9w0BAQUFAAOBgQBImpjBAY5P7ol0
Dfnu4jTgaedgpss5oC9zsi4RC8NOan040o1WVNif2924TMaSv5B5oyiZWUGJLt1r
JLCfYtZX3dAwWpIFnKPSXPcezcTorUWTD78f7+Qs6aax5arN6inxC8LzEWOzeyRw
MpOmsgTMhDJjltNNAnF6jOe6rxDALg==
-----END CERTIFICATE-----
Certificate bag
Bag Attributes
    friendlyName: Bouncy Primary Certificate
subject=/C=AU/O=The Legion of the Bouncy Castle/OU=Bouncy Primary Certificate
issuer=/C=AU/O=The Legion of the Bouncy Castle/OU=Bouncy Primary Certificate
-----BEGIN CERTIFICATE-----
MIICLDCCAZWgAwIBAgIBATANBgkqhkiG9w0BAQUFADBcMQswCQYDVQQGEwJBVTEo
MCYGA1UECgwfVGhlIExlZ2lvbiBvZiB0aGUgQm91bmN5IENhc3RsZTEjMCEGA1UE
CwwaQm91bmN5IFByaW1hcnkgQ2VydGlmaWNhdGUwHhcNMTcxMjMwMDIwNTQ0WhcN
MTgwMjI4MDIwNTQ0WjBcMQswCQYDVQQGEwJBVTEoMCYGA1UECgwfVGhlIExlZ2lv
biBvZiB0aGUgQm91bmN5IENhc3RsZTEjMCEGA1UECwwaQm91bmN5IFByaW1hcnkg
Q2VydGlmaWNhdGUwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBAKZeos3VmEH5
ArZnt5y5XwvHmdUPg0WNxSAe33xwipx33ge1tT9MYyDkXOvmywk234D9uAyblkY/
HvMAapNrGbVk6C5NRCpFjwW1YWa92DMWi9RDetMnDw1cv+WkaQqLbeaaeEhoZ2OB
HrvzKJhiB8d02c6T3KO6araImqMBD9d5AgMBAAEwDQYJKoZIhvcNAQEFBQADgYEA
axNdt0JvRmdw67VvoioAiY8c0iTy0/Ic2cINDhBeMlyX6eiseCpovDzcIZVS8C57
o1eSjwuaBy5WwDQOvQbZ74pVO1setHo6tfRpmigwc1u6gaDxxKL50PyZ22PS550u
46f3rw+XjhGfoQwileXMPJ8hJHqMZQuHjsAy4+eJS3c=
-----END CERTIFICATE-----
Certificate bag
Bag Attributes
    friendlyName: Bouncy Intermediate Certificate
subject=/C=AU/O=The Legion of the Bouncy Castle/OU=Bouncy Intermediate Certificate/emailAddress=feedback-crypto@bouncycastle.org
issuer=/C=AU/O=The Legion of the Bouncy Castle/OU=Bouncy Primary Certificate
-----BEGIN CERTIFICATE-----
MIIDIzCCAoygAwIBAgIBAjANBgkqhkiG9w0BAQUFADBcMQswCQYDVQQGEwJBVTEo
MCYGA1UECgwfVGhlIExlZ2lvbiBvZiB0aGUgQm91bmN5IENhc3RsZTEjMCEGA1UE
CwwaQm91bmN5IFByaW1hcnkgQ2VydGlmaWNhdGUwHhcNMTcxMjMwMDIwNTQ0WhcN
MTgwMjI4MDIwNTQ0WjCBkjELMAkGA1UEBhMCQVUxKDAmBgNVBAoMH1RoZSBMZWdp
b24gb2YgdGhlIEJvdW5jeSBDYXN0bGUxKDAmBgNVBAsMH0JvdW5jeSBJbnRlcm1l
ZGlhdGUgQ2VydGlmaWNhdGUxLzAtBgkqhkiG9w0BCQEWIGZlZWRiYWNrLWNyeXB0
b0Bib3VuY3ljYXN0bGUub3JnMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCM
WpR08ZbLZvn98MC+0fS5kyqs91RkLDMUFNLi8mwzAYHlnaWrZsrx7nftKDs638Gh
s3HbkVMe/tAdXJ8zArzN7J1cVHBiZh/4HBefqQVzjyGwZX8imIMJ1MfweEMjsWSp
KGfd8n/34ZZXVjZFsI3zd2KkhhqT4nvJ1gkj19IjzQIDAQABo4G9MIG6MB0GA1Ud
DgQWBBTzMpoe7Jyo54ficyh0rOWnihnCpDCBhAYDVR0jBH0we4AULrvQaIUwE+yo
EpZVQuovt3Dj2mGhYKReMFwxCzAJBgNVBAYTAkFVMSgwJgYDVQQKDB9UaGUgTGVn
aW9uIG9mIHRoZSBCb3VuY3kgQ2FzdGxlMSMwIQYDVQQLDBpCb3VuY3kgUHJpbWFy
eSBDZXJ0aWZpY2F0ZYIBATASBgNVHRMBAf8ECDAGAQH/AgEAMA0GCSqGSIb3DQEB
BQUAA4GBADkgMlIUOSub1ypi8RlKfBl54SennqwOSfDu63W0cbkLF2uCOxRTTLQo
gCoTwMOUrO/9xWEnY78iS1KXO8+yhZuTFKRjzO4DXUOLgSVQdfoxi9rZtZIfjCaT
wHLSzOEYWO3lOAov61uZDzijrzJdQidocbAxdMHdSR5jSJ2M1xtn
-----END CERTIFICATE-----
PKCS7 Data
Warning unsupported bag type: crlBag
```

OPENSSL现阶段是无法读取crlBag的，现阶段可能实际场景应用极少。加就加上了。🔚

------

