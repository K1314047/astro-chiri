---
title: 'nat主机，ip被墙怎么搭建Hysteria2'
pubDate: '2025-08-30'
---

 nat主机便宜量大，但是ip默认被墙，被墙之后该怎么使用？

1、一键搭建代码：

```
curl -fsSL https://raw.githubusercontent.com/eooce/ssh_tool/main/ssh_tool.sh -o ssh_tool.sh && chmod +x ssh_tool.sh && ./ssh_tool.sh
```

2、选择12.节点搭建合集

![image-20250830142721230](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20250830142721230.png)

3、这我选择的1（1.F佬Sing-box一键脚本）

![image-20250830142550095](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20250830142550095.png)

4、选择1

![image-20250830142813032](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20250830142813032.png)

5、选择a

![image-20250830142831585](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20250830142831585.png)

6、这个是重点，根据自己服务器提供的端口输入

![image-20250830143131629](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20250830143131629.png)

7、服务器NAT端口转发一定要设置

![image-20250830143202212](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20250830143202212.png)

8、自己有argo域名就写自己的，没有就直接默认

![image-20250830143244012](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20250830143244012.png)

9、复制，导入到对应软件就可以了

![image-20250830143322515](C:\Users\Administrator\Desktop\image-20250830143322515.png)
