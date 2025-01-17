## 功能背景
随着数据安全保护的要求越来越严格，各国各地域的信息安全保护法都提到要求数据库进行加密存储。通过数据加密防止意外数据文件遗失导致的数据泄露问题。

## 功能介绍
云数据库 PostgreSQL 提供透明数据加密（Transparent Data Encryption，TDE）功能，透明加密指数据的加解密操作对用户透明，支持对数据文件进行实时 I/O 加密和解密，在数据写入磁盘前进行加密，从磁盘读入内存时进行解密，可满足静态数据加密的合规性要求。加密使用的密钥由密钥管理服务 KMS 产生和管理。

[KMS](https://cloud.tencent.com/document/product/573) 是腾讯云一项保护数据及密钥安全的密钥服务，服务涉及的各个流程均采用高安全性协议通信，保证服务高安全，提供分布式集群管理和热备份，保证服务高可靠和高可用。

KMS 采用的是两层密钥体系，涉及两类密钥，即用户主密钥（CMK）与数据密钥（DATA KEY）。用户主密钥用于加密数据密钥或密码、证书、配置文件等小包数据（最多4KB）。数据密钥用于加密业务数据。海量的业务数据在存储或通信过程中使用数据密钥以对称加密的方式加密，而数据密钥又通过用户主密钥采用非对称加密方式加密保护。通过两层密钥体系，确保数据文件加密。

## 支持版本
内核版本 v11.12_r1.3。

## 适用场景
透明数据加密指数据的加解密操作对用户透明，支持对数据文件进行实时 I/O 加密和解密，在数据写入磁盘前进行加密，从磁盘读入内存时进行解密，可满足静态数据加密的合规性要求。

## 使用说明
开启透明数据加密功能与数据库透明加密的详细介绍，请参见 [透明数据加密](https://cloud.tencent.com/document/product/409/71749)。
