---
layout: post
title: Netty服务器自签证书
excerpt: 记一次在服务器热更中发现的问题
---

### 问题发现

在打开内网测试服后台GM准备热更文件时, 发现打开本地文件夹报错: SecurityError. 

此时得知热更网页使用的window.showDirectoryPicker()在最新chrome下, 强制使用https连接. 由于之前一直在本地开发, 若通过本地访问, 即使是http连接也被浏览器认为是安全的访问, 遂直到部署到内网服后问题才暴露.

接下来对服务器http服务添加证书.

### 服务器的HTTP服务

服务通过Netty启动, 使用内置的http编解码器解码

```java
ServerBootstrap bootstrap = new ServerBootstrap()
    .option(ChannelOption.SO_BACKLOG, 128)
    .group(bossGroup, workerGroup)
    .channel(System.getProperty("os.name").startsWith("Win") ? NioServerSocketChannel.class : EpollServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(@NonNull SocketChannel ch) {
            ch.pipeline()
                .addLast(new HttpServerCodec())
                .addLast(new HttpObjectAggregator(Integer.MAX_VALUE))
                .addLast(new HttpHandler())
                .addLast(new WsHandler());
        }
});
```

### 添加证书

证书可以向CA机构申请获得, 或者由公司的运维部门签发. 由于项目还没有上线, 暂时使用自签证书来满足日常开发.

```java
SslContext sslContext;
try {
    SelfSignedCertificate certificate = new SelfSignedCertificate();
    sslContext = SslContextBuilder.forServer(certificate.certificate(), certificate.privateKey())
            .clientAuth(ClientAuth.NONE)
            .build();
} catch (CertificateException | SSLException e) {
    Log.http.error("自签证书生成失败", e);
}
ServerBootstrap bootstrap = new ServerBootstrap()
    .option(ChannelOption.SO_BACKLOG, 128)
    .group(bossGroup, workerGroup)
    .channel(System.getProperty("os.name").startsWith("Win") ? NioServerSocketChannel.class : EpollServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(@NonNull SocketChannel ch) {
            //cors
            CorsConfig corsConfig = CorsConfigBuilder.forAnyOrigin().allowNullOrigin().allowCredentials().build();
            ch.pipeline()
                .addLast(sslContext.newHandler(ch.alloc()));
                .addLast(new HttpServerCodec())
                .addLast(new HttpObjectAggregator(Integer.MAX_VALUE))
                .addLast(new CorsHandler(corsConfig))
                .addLast(new HttpHandler(httpWorkStore))
                .addLast(new WsHandler());
        }
});
```

### 自定义加密

若服务器部署在公网上, 可以考虑使用 BouncyCastle 自定义加密.

添加依赖

```kotlin
implementation("org.bouncycastle:bcprov-jdk18on:1.77")
implementation("org.bouncycastle:bcpkix-jdk18on:1.77")
```

注册Bouncy Castle提供者并生成自签证书

```java
// 注册Bouncy Castle提供者
Security.addProvider(new BouncyCastleProvider());
// 生成自签名证书和密钥对
KeyPair keyPair = generateKeyPair();
X509Certificate cert = generateSelfSignedCertificate(keyPair);

SslContext sslContext = SslContextBuilder
    .forServer(keyPair.getPrivate(), cert)
    .build();
```
