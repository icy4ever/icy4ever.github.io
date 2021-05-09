## Http与Https

![img](https://upload-images.jianshu.io/upload_images/627325-dc83fef6ac2e6c88.png)

根据这张图片，分析下Https：

首先客户端发起请求，服务端返回crt公钥（证书），客户端检查下这个证书是否可信，如果不可信发出警告⚠️，然后生成一个随机密钥，将密钥通过证书加密 传给服务器，服务器解密后，用密钥加密信息返回给客户端，客户端根据密钥解密得到信息。