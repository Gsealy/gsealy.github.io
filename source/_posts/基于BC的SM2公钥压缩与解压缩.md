---
title: 基于BC的SM2公钥压缩与解压缩
date: 2019-01-22 13:40:49
tags:
- BouncyCastle
- SM2
- EC
---

> `ECC`的公钥是支持压缩的，`SM2椭圆曲线公钥密码算法`中也给出了压缩的方法，这里就不讨论如何计算了，直接看看BC是如何实现的。

### 编码压缩

```java
/**
* encode EC PublicKey
* 
* @param publicKey EC公钥
* @return 压缩的公钥
* @throws InvalidKeySpecException 不是EC公钥，抛出异常
*/
public static byte[] encodePoint(PublicKey publicKey) throws InvalidKeySpecException {
	if (publicKey instanceof ECPublicKey) {
		BCECPublicKey bcPubKey = (BCECPublicKey) publicKey;
		return bcPubKey.getQ().normalize().getEncoded(true);
	}
	throw new InvalidKeySpecException("cannot identify EC public key.");
}
```

### 解码解压

```java
/**
* decode 16进制的Point X
* 
* @param compressedKey
* @param curveName
* @throws InvalidKeySpecException
* @throws NoSuchAlgorithmException
*/
public static PublicKey decodePoint(String compressedKey, String curveName)
		throws NoSuchAlgorithmException, InvalidKeySpecException {
	X9ECParameters spec = ECNamedCurveTable.getByName(curveName);
	// 根据X恢复点Y
	ECPoint W = spec.getCurve().decodePoint(Hex.decode(compressedKey));
	return toAsn1PublicKey(W.normalize(), curveName);
}

```

调用的方法在Gist上，给出了一个EC 工具包，包含计算公钥，格式转换等：[EC Utils](https://gist.github.com/Gsealy/778efb420d325582541a1ec67a98b73e)🔚

------

