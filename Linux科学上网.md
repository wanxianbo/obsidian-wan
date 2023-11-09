执行 cd && mkdir clash 在用户目录下创建 clash 文件夹

下载适合的 Clash 二进制文件并解压重命名为 clash

一般个人的64位电脑下载 clash-linux-amd64 的压缩包即可。（此处的“amd”并非指 CPU 品牌，无需担心 intel 的 CPU 无法使用）

https://github.com/Dreamacro/clash/releases

https://github.com/Dreamacro/maxmind-geoip/releases/latest/download/Country.mmdb

在 clash 文件夹路径中执行 wget -O config.yaml "https://www.thismiao.xyz/link/kr1LsX8mPQyKqgZA?clash=1&log-level=info" 下载 Clash 配置文件（可变）

接着执行chmod +x clash和./clash -d . 以启动 Clash。启动后请勿关闭命令行，否则Clash也会被一同关闭

在 Linux 中使用浏览器（注：Linux 自带的浏览器可能版本过低，请先升级浏览器版本，否则可能无法正常使用）访问 Clash Dashboard 选择节点开始使用

如果出现右图，则表示有步骤做错了

以 Ubuntu 19.04 为例，打开系统设置，选择网络，点击网络代理右边的 ⚙ 按钮，选择手动，填写 HTTP 和 HTTPS 代理为 127.0.0.1:7890，填写 Socks 主机为 127.0.0.1:7891，即可启用系统代理