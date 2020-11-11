### 1. 解决链接
* https://blog.csdn.net/kris_fei/article/details/85100971
* 不能显示的原因：
* 因为 DRM pannel 一开始没有初始化，直接 显示 LOGO 会出现问题。
* 这应该算是 DRM 的一个 BUG。
```
K神，我最近也在调试这个IC，panel-simple.c中的panel_simple_prepare函数一直不调用，导致初始命令没发过去，是因为没打你文中所提到的patch吗？请问你这个patch可以分享一下吗？另外上电时序那块儿的时间是如何测出来的呢？我也想检查一下我的上电时序对不对。2年前回复
点赞
kris_fei
KrisFei回复:嗯，调试前需要把uboot logo关闭掉。2年前回复
点赞
weixin_43245081
weixin_43245081回复:多谢K神，我尝试把dts中route-dsi节点 disabled掉了，panel-simple.c可以正常发送命令了。多谢指点。2年
```
* 如果出现 RK 的问题，可以找这个人。

### 2. 现象
* 一开始 DTS 打开了 hdmi 之后。
* Backlight 先是被使能，然后一段时间后被关闭。
* 后面我只留下了 dsi 显示部分的配置。
* 这次背光能打开，也能控制，但是发现没有显示信号。
* 经过打印日志，在 drivers/gpu/drm/panel/panel-simple.c 跟踪代码.
* 发现 panel-init-sequence 命令的解析函数执行了，但是发送命令的函数没有执行 。
* 也就是说，TC358775 都没有被初始化，那么肯定有问题。

### 3. 解决
* 有对比过 Firefly 的代码，也把一些 Firefly 的代码 copy 过去。但是我感觉都是没啥用的。
* 最后我去翻 CDSN 里面 K神的博客，找到了关于这个问题的描述。
* 需要将 route_dsi 先关闭，不能在U-boot阶段显示 LOGO，不然就会出现上面的情况。
