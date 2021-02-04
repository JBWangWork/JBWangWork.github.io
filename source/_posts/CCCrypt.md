---
title: 对称加密算法原理--OpenSSL演示、iOS代码运用及CCCrypt安全隐患
date: 2019-01-13 15:32:28
author: Vincent
categories: 
- 算法
- 加密原理
tags: 
- 算法
- 对称加密原理
- OpenSSL
- CCCrypt安全隐患
---

![对称加密算法原理](https://upload-images.jianshu.io/upload_images/5741330-6fba0bae30b20faa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

之前介绍了非对称加密算法，这篇文章介绍一下在非对称加密算法出现之前的对称加密算法，常见的对称加密算法、终端演示OpenSSL和iOS代码运用以及CCCrypt的安全隐患等。

> 对称加密算法：明文通过密钥加密得到密文，密文再通过这个密钥解密得到明文。所以在业务逻辑上相对没有非对称加密RSA的安全性高。



#### 常见的对称加密算法
- DES
数据加密标准，但由于强度不高，暴力破解难度不是很高，所以用的很少。
- 3DES
使用3个密钥，对数据进行三次加密，强度增强。虽然强度相对DES有所提高，但是对称加密算法密钥的保存就很难，3DES的3个密钥更麻烦，所以3DES也没有被广泛使用。
- AES
高级密码标准，加密强度非常高，被广泛使用，美国安全局和苹果钥匙串访问都是用了AES加密算法。

#### 常用的两种加密模式
- ECB（Electronic Code Book）：电子密码本模式（每一块数据独立加密）
最基本的加密模式，也就是通常理解的加密，相同的明文将永远加密成相同的密文，无初始向量，容易受到密码本重放攻击，一般情况下很少用。
- CBC（Cipher Block Chaining）：密码分组链接模式（使用一个密钥和一个初始化向量[IV]对数据执行加密。每一块数据加密都依赖上一块数据，有效的保证数据的完整性）
明文被加密前要与前面的密文进行异或运算后再加密，因此只要选择不同的初始向量，相同的密文加密后会形成不同的密文，这是目前应用最广泛的模式。CBC加密后的密文是上下文相关的，但明文的错误不会传递到后续分组，但如果一个分组丢失，后面的分组将全部作废(同步错误)。CBC可以有效的保证密文的完整性，如果一个数据块在传递时丢失或改变，后面的数据将无法正常解密。

当然了，除了对称加密和非对称加密外，我们肯定还听说过Hash
#### Hash概述
> Hash：一般翻译做“散列”，也有直接音译为“哈希”的，就是把任意长度的输入通过散列算法变换成固定长度的输出，该输出就是散列值。这种转换是一种压缩映射，也就是，散列值的空间通常远小于输入的空间，不同的输入可能会散列成相同的输出，所以不可能从散列值来确定唯一的输入值。简单的说就是一种将任意长度的消息压缩到某一固定长度的消息摘要的函数


之前介绍了RSA加密算法，在RSA算法之后也衍生出很多加密算法，典型的算法就有HASH函数（也称之为散列函数，严格意义上不算是加密算法，只不过是和加密一起用），还有在RSA出现之前的对称加密算法，这些算法都是公开的。

#### Hash特点
算法是公开的
相同的数据加密结果不变
不同的数据加密结果定长（MD5得到的结果默认是128位二进制，一般用16进制的32个字符来标识）
不可逆
信息摘要，信息“指纹”，用来做数据识别的

#### HASH用途
密码加密：服务器不需要知道用户真实密码，只需要匹配HASH值
搜索引擎
版权
数字签名

#### OpenSSL演示ECB和CBC的区别
- OpenSSL演示ECB模式加密
```
// 先创建一个待加密的message.txt文件，编辑内容
$ vi message.txt
$ cat message.txt
  Hello vincent!!!
// enc -des-ecb是对称加密算法DES的ECB模式，-K是密钥，616263就是ASCII码“abc”，-nosalt不加盐（OpenSSL默认会加盐），输出msg.bin
$ openssl enc -des-ecb -K 616263 -nosalt -in message.txt -out msg.bin
```
这时在文件夹内会多出一个加密过后的二进制文件msg.bin，修改message.txt文件的内容，再次进行加密
```
// 先修改message.txt的内容，vincent->Vincent
$ vi message.txt
$ cat message.txt
  Hello Vincent!!!
// 再次对message.txt进行同样的方式加密，输出msg1.bin
$ openssl enc -des-ecb -K 616263 -nosalt -in message.txt -out msg1.bin
// 查看下msg.bin和msg1.bin有什么不同
$ xxd msg.bin
00000000: 6d87 4097 d383 0bda a5bc d168 de16 688d  m.@........h..h.
00000010: b8db 0794 f9ed eca9
$ xxd msg1.bin
00000000: 20e2 8361 50a7 16a0 a5bc d168 de16 688d   ..aP......h..h.
00000010: b8db 0794 f9ed eca9
```
我们会发现，修改message.txt内容后ECB模式加密的结果只是修改部分不同，前后加密结果不变

- OpenSSL演示CBC模式加密
```
// 先编辑message.txt的内容
$ vi message.txt
$ cat message.txt
  Hello vincent!!!
// 再次对message.txt进行CBC方式加密，相对ECB模式除了修改-des-cbc，还会多一个iv参数，iv是初始化向量，输出msg2.bin
$ openssl enc -des-cbc -iv 0102030405060708 -K 616263 -nosalt -in message.txt -out msg2.bin
// 再次编辑message.txt的内容，vincent->Vincent
$ vi message.txt
$ cat message.txt
  Hello Vincent!!!
// 对修改后的文件加密，输出msg3.bin
$ openssl enc -des-cbc -iv 0102030405060708 -K 616263 -nosalt -in message.txt -out msg3.bin
// 查看下msg2.bin和msg3.bin有什么不同
$ xxd msg2.bin
00000000: d647 a33b 0389 dea5 3c81 02c9 ec05 44dd  .G.;....<.....D.
00000010: 467c a581 ab1a 415a
$ xxd msg3.bin
00000000: 9882 c1b6 3186 b465 b3be d08a 5ad5 2fd1  ....1..e....Z./.
00000010: 6032 add7 bdb2 07da                      `2......
```
经多次测试发现，修改message.txt内容后CBC模式加密的结果是修改部分不同以及后面的加密结果也会变化
- 终端测试指令
>   加密过程：先加密，再base64编码
     解密过程：先base64解码，再解密
```
//  DES(ECB)加密
$ echo -n hello | openssl enc -des-ecb -K 616263 -nosalt | base64
 
// DES(CBC)加密
$ echo -n hello | openssl enc -des-cbc -iv 0102030405060708 -K 616263 -nosalt | base64

//  AES(ECB)加密 128位
$ echo -n hello | openssl enc -aes-128-ecb -K 616263 -nosalt | base64
 
 //  AES(CBC)加密
$ echo -n hello | openssl enc -aes-128-cbc -iv 0102030405060708 -K 616263 -nosalt | base64
 
//  DES(ECB)解密  base64 -D进行解码成二进制 -d解密
$ echo -n HQr0Oij2kbo= | base64 -D | openssl enc -des-ecb -K 616263 -nosalt -d

//  DES(CBC)解密
$ echo -n alvrvb3Gz88= | base64 -D | openssl enc -des-cbc -iv 0102030405060708 -K 616263 -nosalt -d

//  AES(ECB)解密
$ echo -n d1QG4T2tivoi0Kiu3NEmZQ== | base64 -D | openssl enc -aes-128-ecb -K 616263 -nosalt -d

//  AES(CBC)解密
$ echo -n u3W/N816uzFpcg6pZ+kbdg== | base64 -D | openssl enc -aes-128-cbc -iv 0102030405060708 -K 616263 -nosalt -d
```
#### 对称加密算法代码演示
```
// AES加密、ECB模式对“hello vincent!!!”进行加密
    NSString *ECBEncryptStr = [[EncryptionTools sharedEncryptionTools] encryptString:@"hello vincent!!!" keyString:@"abc" iv:nil];
    NSLog(@"%@", ECBEncryptStr);
    // 解密
    NSString *ECBDecrypt = [[EncryptionTools sharedEncryptionTools] decryptString:ECBEncryptStr keyString:@"abc" iv:nil];
    NSLog(@"%@", ECBDecrypt);
    
    // 一个数组，和前面一样有8个数据
    uint8_t iv[8] = {1, 2, 3, 4, 5, 6, 7, 8};
    // 把数组包装才二进制NSdata 把数组的指针和长度传进去
    NSData *ivData = [NSData dataWithBytes:iv length:sizeof(iv)];
    // AES加密、CBC模式
    NSString *CBCEncryptStr = [[EncryptionTools sharedEncryptionTools] encryptString:@"hello vincent!!!" keyString:@"abc" iv:ivData];
    NSLog(@"%@", CBCEncryptStr);
    // 解密
    NSString *CBCDecrypt = [[EncryptionTools sharedEncryptionTools] decryptString:CBCEncryptStr keyString:@"abc" iv:ivData];
    NSLog(@"%@", CBCDecrypt);
```
打印结果：
```
LzWe4b6VMKHECZTg5GEoDvOJyUo3lvcCucS987KliFw=
hello vincent!!!
Vo04z90TAfQX07onyrvCie1SnRpsbHKMkYnaNhcEPP0=
hello vincent!!!
```
虽然将加密和解密封装成了两个方法，但是苹果内部加密和解密都是用的一个函数。先看下其中封装的一个方法内部实现
```
// 加密字符串并返回base64编码字符串
- (NSString *)encryptString:(NSString *)string keyString:(NSString *)keyString iv:(NSData *)iv {
    
    // 设置秘钥 将keyString转成二进制
    NSData *keyData = [keyString dataUsingEncoding:NSUTF8StringEncoding];
    uint8_t cKey[self.keySize];
    bzero(cKey, sizeof(cKey));
    [keyData getBytes:cKey length:self.keySize];
    
    // 设置iv
    uint8_t cIv[self.blockSize];
    bzero(cIv, self.blockSize);
    int option = 0;
    if (iv) {
        [iv getBytes:cIv length:self.blockSize];
        option = kCCOptionPKCS7Padding; // CBC加密
    } else {
        option = kCCOptionPKCS7Padding | kCCOptionECBMode;  // ECB加密
    }
    
    // 设置输出缓冲区 将原始数据转成二进制，并根据所使用的加密方式设置缓冲区
    NSData *data = [string dataUsingEncoding:NSUTF8StringEncoding];
    size_t bufferSize = [data length] + self.blockSize;
    void *buffer = malloc(bufferSize);
    
    // 开始加密
    size_t encryptedSize = 0;
    //加密解密都是它 -- CCCrypt
    CCCryptorStatus cryptStatus = CCCrypt(kCCEncrypt,
                                          self.algorithm,
                                          option,
                                          cKey,
                                          self.keySize,
                                          cIv,
                                          [data bytes],
                                          [data length],
                                          buffer,
                                          bufferSize,
                                          &encryptedSize);
    
    NSData *result = nil;
    if (cryptStatus == kCCSuccess) {
        result = [NSData dataWithBytesNoCopy:buffer length:encryptedSize];
    } else {
        free(buffer);
        NSLog(@"[错误] 加密失败|状态编码: %d", cryptStatus);
    }
    
    return [result base64EncodedStringWithOptions:0];
}

// 解密字符串
- (NSString *)decryptString:(NSString *)string keyString:(NSString *)keyString iv:(NSData *)iv {
    
    // 设置秘钥
    NSData *keyData = [keyString dataUsingEncoding:NSUTF8StringEncoding];
    uint8_t cKey[self.keySize];
    bzero(cKey, sizeof(cKey));
    [keyData getBytes:cKey length:self.keySize];
    
    // 设置iv
    uint8_t cIv[self.blockSize];
    bzero(cIv, self.blockSize);
    int option = 0;
    if (iv) {
        [iv getBytes:cIv length:self.blockSize];
        option = kCCOptionPKCS7Padding;
    } else {
        option = kCCOptionPKCS7Padding | kCCOptionECBMode;
    }
    
    // 设置输出缓冲区
    NSData *data = [[NSData alloc] initWithBase64EncodedString:string options:0];
    size_t bufferSize = [data length] + self.blockSize;
    void *buffer = malloc(bufferSize);
    
    // 开始解密
    size_t decryptedSize = 0;
    CCCryptorStatus cryptStatus = CCCrypt(kCCDecrypt,
                                          self.algorithm,
                                          option,
                                          cKey,
                                          self.keySize,
                                          cIv,
                                          [data bytes],
                                          [data length],
                                          buffer,
                                          bufferSize,
                                          &decryptedSize);
    
    NSData *result = nil;
    if (cryptStatus == kCCSuccess) {
        result = [NSData dataWithBytesNoCopy:buffer length:decryptedSize];
    } else {
        free(buffer);
        NSLog(@"[错误] 解密失败|状态编码: %d", cryptStatus);
    }
    
    return [[NSString alloc] initWithData:result encoding:NSUTF8StringEncoding];
}
```
代码已加入相应的注释，就不解释代码了。我们会发现苹果内部加密和解密都是用`CCCryptorStatus CCCrypt(
    CCOperation op, 
    CCAlgorithm alg, 
    CCOptions options, 
    const void *key,
    size_t keyLength,
    const void *iv, 
    const void *dataIn, 
    size_t dataInLength,
    void *dataOut, 
    size_t dataOutAvailable,
    size_t *dataOutMoved)`，这个函数是对称加密算法的核心函数。
![CCCrypt函数](https://upload-images.jianshu.io/upload_images/5741330-55512cbb2b8544e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


WTF!!!11个参数，这么多，这都是干嘛的？
> 参数意义
op：kCCEncrypt(加密)/ kCCDecrypt(解密)
alg：加密算法 kCCAlgorithmAES、kCCAlgorithmDES、kCCAlgorithmBlowfish等等
options：加密方式 kCCOptionPKCS7Padding（CBC方式）/kCCOptionECBMode（ECB方式）
key：加密密钥
keyLength：密钥长度
iv：初始化向量 ECB不需要指定（CBC多了这个参数就相当于加盐，加密强度更高了）
dataIn：加密的数据
dataInLength：加密数据的长度
dataOut：缓冲区（地址），存放密文
dataOutAvailable：缓冲区的大小
dataOutMoved：加密结果的大小

搞清楚每个参数的意义也就明白了，苹果这样设计还是挺人性化的。对称加密和解密所用的参数密钥都是一样的，所以加密和解密都是用同一个函数。苹果的加密算法也都在CommonCrypto.h这个库里面，这个库并不在macho中，是在系统中，所以我们大多数会认为这个加密会很安全，但是事实上并不是这样。


#### CCCrypt函数安全隐患

现在我们已经使用CCCrypt对数据进行加密和解密了，接下来看下我们用CCCrypt加密的数据是否真的安全。下面的内容涉及到逆向开发，可能有点跑偏，如果感兴趣的小伙伴也可以进一步研究一下。
我们加密数据就是为了防止中间人攻击，假设如果别人拿到我们的APP，别人肯定不会知道我们的源码，也不知道在数据核心加密的地方是不是用的CCCrypt，这时别人会进行符号断点，当然这个只要是没有去符号，或者系统的都是可以拦截到的。
![符号断点](https://upload-images.jianshu.io/upload_images/5741330-b73c172a7ca12ed7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 前方高能！！！

下了断点后，我们继续运行刚才的demo，程序果断进入断点，接下来要读寄存器了！！！

![WeChat112460611fd045da8d8d9a181882dc6d.png](https://upload-images.jianshu.io/upload_images/5741330-76d1819cc5f6e894.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


刚刚说过，CCCrypt第7个参数是我们加密的数据，所以在寄存器中X6（从X0开始）就是我们的加密数据，我们在lldb中读取X6的地址值，也就是指针，拿到X6的地址值是0x00000001c403ee00，再p (char *)0x00000001c403ee00查看X6的值，回车！！！我们刚刚加密的数据显示出来了，我们原以为很安全的手段就这样被别人拿到了！所以我们有很多核心的加密算法不能直接用。

那怎么防御呢？先想到去符号，前面也说了，这个库是系统的，所以没办法去符号，当然自己实现或者三方库，比如支付宝就是直接用的OpenSSL，可以去符号来避免被直接破解。最好的方式是加密之前不能直接使用关键数据，我们可以自己对关键数据处理一下比如异或，方法肯定不止一个，如果各位有什么好的解决办法欢迎交流。

后面可能会介绍下怎么隐藏函数调用，怎样保护核心数据，当然逆向大神还是很多的，这也仅仅是让逆向变的更难而已。该文章为记录本人的学习路程，希望能够帮助大家！！！

