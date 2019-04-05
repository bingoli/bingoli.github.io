---
title: Nodejs的DES加密解密实现
date: 2018-11-15 23:11:45
tags:
- crypto
- Node.js
- DES
---

## 引言
为了保证敏感数据的安全，客户端和服务器除了使用加密协议（如https）传输之外，一般还会再对敏感数据进行加密处理，比如使用DES算法进行加密。客户端和服务器会约定使用相同的加密解密算法，但由于使用不同语言不同基础库，加密解密过程中的一些细节处理会有差异，比如填充值的值。一般客户端和服务器是由不同人的开发，且都是各自负责自己的代码，对对方的代码不太了解，就容易导致在调试过程中花费较多的时间。这时候，就体现出了全栈工程师的优势了，能够较快的把接口调通。今天，我先总结下在服务器使用Nodejs实现DES解密中的一些需要明确的细节。
## DES加密解密示例
Nodejs的Crypto模块封装了各种加密解密的算法，可以非常方便的使用。我今天选用了DES-CBC算法。
加密算法的调用过程中如下：
1. 通过crypto.createCipheriv创建一个加密对象。
2. 通过cipher.update对原始数据进行加密，并根据设定的编码方式输出部分密文。update可以调用多次，并需要把每次输出的密文合并到一起。
3. 调用cipher.final对最后剩余的数据进行加密，并输出剩余的密文，和之前update输出的密文合并，就是最终的完整密文。
解密过程跟加密过程是类似的步骤。

加密解密的示例代码如下所示
``` javascript
var crypto = require('crypto');

function encrypt(plainText, key, iv) {
    var cipher = crypto.createCipheriv('des-cbc', key, iv);
    cipher.setAutoPadding(true);
    var encryptedText = cipher.update(plainText, 'utf8', 'base64');
    encryptedText += cipher.final('base64');
    return encryptedText;
}

function decrypt(encryptedText, key, iv) {
    var decipher = crypto.createDecipheriv('des-cbc', key, iv);
    decipher.setAutoPadding(true);
    var plainText = decipher.update(encryptedText, 'base64', 'utf8');
    plainText += decipher.final('utf8');
    return plainText;
}

function testEncryptAndDecrypt() {
    var key = "12345678";
    var iv = "abcdefgh";

    var plainText = "Hello World!";
    console.log("Origin Text: %s", plainText);

    var encryptedText = encrypt(plainText, key, iv);
    console.log("Encrypted Text: %s", encryptedText);
    
    var decryptedText = decrypt(encryptedText, key, iv);
    console.log("Decrypted Text: %s", decryptedText);
}

testEncryptAndDecrypt();
```

## 密码的使用
DES的密码长度必须是8个字节，否则，会抛出异常。在创建加密对象时，如果输入的密码是utf8编码的字符串。如果在密码中使用了一些特殊的字节，并把ASCII码转换为字符串，有可能导致抛出密码长度不对的异常。原因是，转换后的字符串不符合utf8的编码规则。错误的用法示例代码如下。
``` javascript
var key = "\xff\xff\xff\xff\xff\xff\xff\xff";
```

如果要使用这些字节作为密码，那输入密码的时候，就需要改用TypedArray，比如Int8Array。示例代码如下所示。
``` javascript
var key = new Int8Array([0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff]);
```

其中，iv的长度必须是8个字节，其使用方法跟密码相似。

## 自动填充字段
DES解密算法是对称性的块加密算法，每个块的长度为8个字节。可通过cipher设置是否使用自动填充补齐8个字节，如果不使用自动填充，就需要保证输入数据的字节数是8的倍数，否则会抛出异常。如果使用自动填充，在调用cipher.final的时候，会采用PCKS5 Padding模式对数据进行自动填充。填充数据的值是一共填充的字节数量对应的ASCII码，即缺几个字节，填充的值就是几。需要注意的是，如果输入的数据的字节数正好是8的倍数，那样也会填充8个字节。填充的数据只会是下面的8种，其中*为原来的数据。
*******1
******22
*****333
****4444
***55555
**666666
*7777777
88888888
## Buffer的使用
如果单纯使用加密算法转化出来的密文，可能会存在一些不可见的字符，比如填充值。而Nodejs的Buffer类正好可以帮助解决这个问题。通过Buffer可以实现多种不同编码的转换。而在cipher.update和final函数中设置了输出参数的编码方式，就会直接转换为字符串。Buffer可以作为字符串编码转换的工具，如base64的编码和解码。
``` javascript
function base64Encode(utf8String) {
    var buffer = new Buffer(utf8String, 'utf8');
    return buffer.toString('base64');
}

function base64Decode(base64String) {
    var buffer = new Buffer(base64String, 'base64');
    return buffer.toString('utf8');
}
```
## 结语
Nodejs使用起来非常方便，尤其是C++程序员感受明显。同样是加密解密，用C++实现就需要费一番周折，而且一不小心，就容易写出问题。在使用C++编程的时候，就特别希望也能多存在一些这种类似库的封装。

## 参考文档
- [NodeJS开发文档](https://nodejs.org/api/crypto.html)

## 关于作者
微信公众号：程序员bingo
![微信公众号：程序员bingo](https://bingoli.github.io/wechat.jpeg)
Blog: https://bingoli.github.io/
GitHub: https://github.com/bingoli