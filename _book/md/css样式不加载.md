## nginx配置成功 图片加载但css样式不加载

尝试过包括修改 nginx.conf等众多方法后发现根本没有效果  如下：

<img src="img/cssmiss.png" alt="image-20191125181708924" style="zoom:33%;" />

去除顶部的

```html
<!DOCTYPE html>
```

就可以使用外联css文件了。