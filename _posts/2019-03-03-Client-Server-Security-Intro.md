---
layout: post
title: 客户端与服务端数据加密传输方案
categories: [Security]
description: 客户端与服务端的传输数据加密，是网络通信中必不可少的。
keywords: Security, 网络安全
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 前言

从前一篇[网络安全基础要点知识介绍]({{site.url}}/2019/03/02/Net-Security-Intro/)中可以知道，在网络通信中，通信传输数据容易被截取或篡改，如果在传输用户隐私数据过程中，被不法分子截取或篡改，就可能导致用户受到伤害，比如被诈骗，所以对客户端与服务端的传输数据加密，是网络通信中必不可少的。

## 数据加密方案

首先，客户端与服务端商量好数据加密协议，对传输数据做到安全保护。

安全保护至少需要有下面两点：

1. 采用HTTPS协议
2. 采用公钥密码体制RSA算法对数据加密

现在安全是保证了，但还要考虑到性能问题，由于RSA算法对数据加密时**运算速度慢**，所以直接把所有传输数据都用RSA加密，会导致网络通信慢，这对用户将是不好的体验。由于对称密钥密码体制中的AES**运算速度快且安全性高**，所以结合AES对传输数据加密是非常好的方案。

下面是对客户端与服务端通信数据加密比较通用的方案：

1.  客户端生成AES密钥，并保存AES密钥
2.  客户端用AES密钥对请求传输数据进行加密
3.  客户端使用RSA公钥对AES密钥加密，然后把值放到自定义的一个请求头中
4.  客户端向服务端发起请求
5.  服务端拿到自定义的请求头值，然后使用RSA私钥解密，拿到AES密钥
6.  服务端使用AES密钥对请求数据解密
7.  服务端对响应数据使用AES密钥加密
8.  服务端向客户端发出响应
9.  客户端拿到服务端加密数据，并使用之前保存的AES密钥解密

注意：**传输数据使用AES密钥加密，RSA公钥对AES密钥加密。RSA公钥和私钥由服务端生成，公钥放在客户端，私钥放在服务端。公钥私钥要私密保护，不能随便给人。**

具体流程图如下：


<img src="{{site.url}}/images/posts/2019-03-03-Client-Server-Security-Intro/p1.png" />



上面网络通信过程是安全的，可以保证通信数据**即使被截取了，也无法获得任何有效信息；即使被篡改了，也无法被客户端和服务端验证通过。**

## 数据加密细节

### AES加解密

生成AES密钥和使用AES密钥加密、解密，有下面重要的几点：

1. 密钥长度的选择：AES能支持的密钥长度可以为128,192,256位(也即16,24,32个字节)，这里选择128位。

2. 算法/模式/填充的选择：

<table border="1">
<tr>
<th>算法/模式/填充</th>
<th>字节加密后数据长度</th>
<th>不满16字节加密后长度 </th>
</tr>
<tr>
<td>AES/CBC/NoPadding</td>
<td>16 </td>
<td> 不支持 </td>
</tr>
<tr>
<td>AES/CBC/PKCS5Padding </td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/CBC/ISO10126Paddind</td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/CFB/NoPadding  </td>
<td> 16</td>
<td> 原始数据长度  </td>
</tr>
<tr>
<td>AES/CFB/PKCS5Padding</td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/CFB/ISO10126Padding </td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/ECB/NoPadding </td>
<td> 16</td>
<td> 不支持 </td>
</tr>
<tr>
<td>AES/ECB/PKCS5Padding</td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/ECB/ISO10126Padding</td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/ECB/ISO10126Padding</td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/OFB/NoPadding</td>
<td> 16</td>
<td> 原始数据长度 </td>
</tr>
<tr>
<td>AES/OFB/PKCS5Padding </td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/OFB/ISO10126Padding</td>
<td> 32</td>
<td> 16 </td>
</tr>
<tr>
<td>AES/PCBC/NoPadding</td>
<td> 16</td>
<td> 不支持 </td>
</tr>
<tr>
<td>AES/PCBC/PKCS5Padding</td>
<td> 32</td>
<td> 16</td>
</tr>
<tr>
<td>AES/PCBC/ISO10126Padding </td>
<td> 32</td>
<td> 16</td>
</tr>
</table>

这里选择AES/CBC/PKCS5Padding。

3.添加向量 IvParameterSpec：增强算法强度。

4.编码格式选择：UTF-8。

下面为具体代码实现：

```
    private final int AES_KEY_LENGTH = 16;//密钥长度16字节，128位
    private final String AES_ALGORITHM = "AES";//算法名字
    private final String AES_TRANSFORMATION = "AES/CBC/PKCS5Padding";//算法/模式/填充
    private final String AES_IV = "0112030445060709";//使用CBC模式，需要一个向量iv，可增加加密算法的强度
    private final String AES_STRING = "abcdefghijklmnopqrstuvwxyzABCDEFGHIGKLOP";
    private final Charset UTF_8 = Charset.forName("UTF-8");//编码格式

    /**
     * 使用AES加密
     *
     * @param aesKey AES Key
     * @param data   被加密的数据
     * @return AES加密后的数据
     */
    private byte[] encodeAES(byte[] aesKey, String data) {
        if (aesKey == null || aesKey.length != AES_KEY_LENGTH) {
            return null;
        }
        SecretKeySpec keySpec = new SecretKeySpec(aesKey, AES_ALGORITHM);
        try {
            Cipher cipher = Cipher.getInstance(AES_TRANSFORMATION);
            IvParameterSpec iv = new IvParameterSpec(AES_IV.getBytes(UTF_8));
            cipher.init(Cipher.ENCRYPT_MODE, keySpec, iv);
            return cipher.doFinal(data.getBytes(UTF_8));
        } catch (Exception e) {
            Log.d(TAG, e.getMessage(), e);
        }
        return null;
    }

    /**
     * 使用AES解密
     *
     * @param aesKey AES Key
     * @param data   被解密的数据
     * @return AES解密后的数据
     */
    private String decodeAES(byte[] aesKey, byte[] data) {
        if (aesKey == null || aesKey.length != AES_KEY_LENGTH) {
            return null;
        }
        SecretKeySpec keySpec = new SecretKeySpec(aesKey, AES_ALGORITHM);
        try {
            Cipher cipher = Cipher.getInstance(AES_TRANSFORMATION);
            IvParameterSpec iv = new IvParameterSpec(AES_IV.getBytes(UTF_8));
            cipher.init(Cipher.DECRYPT_MODE, keySpec, iv);
            return new String(cipher.doFinal(data), UTF_8);
        } catch (Exception e) {
            Log.d(TAG, e.getMessage(), e);
        }
        return null;
    }

    private int getRandom(int count) {
        return (int) Math.round(Math.random() * (count));
    }

    /**
     * 生成AES key
     *
     * @return AES key
     */
    private String initAESKey() {
        StringBuilder sb = new StringBuilder();
        int len = AES_STRING.length();
        for (int i = 0; i < AES_KEY_LENGTH; i++) {
            sb.append(AES_STRING.charAt(getRandom(len - 1)));
        }
        return sb.toString();
    }
```

现在AES密钥和AES加密、解密都有了，在通常情况下，还会对加密、解密过程进行Base64 编码、解码。

Base64编码，选择 URL_SAFE 标识，也就是  "-" 和 “_” 会被替换为 "+" 和 "/"，：

```
    /**
     * 对数据进行Base64编码，使用的是{@link android.util.Base64}，而且flags需要使用 {@link android.util.Base64#URL_SAFE,android.util#Base64.NO_PADDING,android.util.Base64#NO_WRAP}。
     *
     * @param input 来源数据
     * @return Base64编码的数据
     */
    private String encodeBase64(byte[] input) {
        return new String(Base64.encode(input, Base64.URL_SAFE | Base64.NO_PADDING | Base64.NO_WRAP), UTF_8);
    }
```

Base64解码，和编码对应：

```
    /**
     * 对数据进行Base64解码，使用的是{@link android.util.Base64}，而且flags需要使用 {@link android.util.Base64#URL_SAFE,android.util.Base64#NO_WRAP}，主要是为了和Base64加密对应。
     *
     * @param str 需要解码的数据
     * @return Base64解码后的数据
     */
    private byte[] decodeBase64(String str) {
        return Base64.decode(str.getBytes(UTF_8), Base64.URL_SAFE | Base64.DEFAULT);
    }
```

### RSA公钥加密

RSA公钥是从服务端拿到的，这个公钥不能被泄漏，必须做到安全保护。

使用RSA公钥加密，也有几个重要点：

1. 拿到的公钥是Base64 编码后的，所以首先需要对公钥Base64解码。

2. 算法/模式/填充的选择：RSA/ECB/PKCS1Padding

3. 编码格式选择：UTF-8。

注意：**使用RSA公钥加密的流程对应的就是服务端使用RSA私钥解密的流程，所以需要和服务端沟通商量好。**

具体代码实现：

```
    private final String RSA_PUB_KEY = "服务端给的公钥";
    private final String RSA_TRANSFORMATION = "RSA/ECB/PKCS1Padding";

    /**
     * 公钥加密
     *
     * @param data           要加密的数据
     * @param key            公钥
     * @param transformation 算法/模式/填充
     * @return 加密后的数据
     */
    public byte[] encryptByPublicKey(byte[] data, String key, String transformation)
            throws GeneralSecurityException {
        byte[] keyBytes = Base64.decode(key.getBytes(UTF_8), Base64.NO_WRAP);
        X509EncodedKeySpec keySpec = new X509EncodedKeySpec(keyBytes);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        PublicKey pubKey = keyFactory.generatePublic(keySpec);

        Cipher cipher = Cipher.getInstance(transformation);
        cipher.init(Cipher.ENCRYPT_MODE, pubKey);
        return cipher.doFinal(data);
    }
```

 ---

## 总结

1. 为了保证网络通信中的通信数据安全，首先采用HTTPS协议和公钥密钥体制中的RSA加密。

2. 因为是RSA运算速度慢，所以采用运算速度快且安全性高的对称密钥密码体制中的AES对所
   有传输数据进行加密，然后再用RSA对AES密钥加密，这样既能保证安全又能保证性能。

3. RSA公钥和私钥由服务端生成，公钥放在客户端，私钥放在服务端。

4. 数据加密后采用Base64编码，数据解密前采用Base64解码。

5. 编码格式同一采用UTF-8。
