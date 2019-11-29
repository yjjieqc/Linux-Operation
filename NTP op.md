## NTP 服务器维护

在Linux系统中，为了避免主机时间因为在长时间运行下所导致的时间偏差，进行时间同步(synchronize)的工作是非常必要的。Linux系统下，一般使用ntp服务来同步不同机器的时间。NTP 是网络时间协议（Network Time Protocol）的简称，干嘛用的呢？就是通过网络协议使计算机之间的时间同步化。

### 时间和时区

```
date

Fri Nov 29 10:35:26 CST 2019

cat /etc/sysconfig/clock

ZONE="Asia/Shanghai"
```

格林威治时间(GMT), 它也就是0时区时间. 但是我们在计算机中经常看到的是UTC. 它是Coordinated Universal Time的简写. 虽然可以认为UTC和GMT的值相等(误差相当之小),但是UTC已经被认定为是国际标准,所以我们都应该遵守标准只使用UTC那么假如现在中国当地的时间是晚上8点的话,我们可以有下面两种表示方式：

20:00 CST

12:00 UTC

这里的CST是Chinese Standard Time,也就是我们通常所说的北京时间了. 因为中国处在UTC+8时区

## NTP的配置

1. 查看是否安装了ntp服务

    ```
    rpm -qa | grep ntp

    ntp-4.2.6p5-15.el6.centos.x86_64
    ntpdate-4.2.6p5-15.el6.centos.x86_64
    fontpackages-filesystem-1.41-1.1.el6.noarch
    ```

2. ntp的配置

    ntp服务器的配置保存在`/etc/ntp.conf`中

    ```vb
    # cat /etc/ntp.conf

    restrict default kod nomodify notrap nopeer noquery 
    restrict -6 default kod nomodify notrap nopeer noquery 
    restrict 11.107.13.100                         //允许该NTP服务器进入
    restrict 11.80.81.1                            //没有任何何參數的話，這表示『该IP或网段不受任何限制』
    restrict 202.112.1.199 
    restrict 127.0.0.1 
    restrict -6 ::1 
    restrict 192.168.0.0 mask 255.255.0.0 nomodify  //该网段可以进行校时
    restrict 0.0.0.0 mask 0.0.0.0 notrust           //拒绝没有认证的用户端
    server time-nw.nist.gov prefer                  //prefer 该服务器优先
    server 0.rhel.pool.ntp.org  iburst        //设定时间服务器
    server 1.rhel.pool.ntp.org  iburst
    server 2.rhel.pool.ntp.org  iburst
    fudge   127.127.1.0 stratum 6 
    driftfile /var/lib/ntp/drift 
    keys /etc/ntp/keys 
    broadcastdelay 0.008
    ```

    上边只是简单设置，没有考虑安全方面如认证等等。

    权限管理使用`restrict`公式如下：
    ```
    restrict IP mask [参数] / restrict 192.168.0.0 mask 255.255.0.0 nomodify
    ```
    其中参数主要有底下这些：

        * ignore：拒绝所有类型的NTP的连线;
        * nomodfiy：用户端不能使用NTPC与ntpq这两支程式来修改伺服器的时间参数，但使用者端仍可透过这部主机来进行网路校时的;
        * noquery：用户端不能够使用ntpq，NTPC等指令来查询发表伺服器，等于不提供的NTP的网路校时幂;
        * notrap：不提供陷阱这个远端事件邮箱（远程事件日志）的功能。
        * notrust：拒绝没有认证的用户端。

    配置文件中的driftfile是什么?

    我们每一个system clock的频率都有小小的误差,这个就是为什么机器运行一段时间后会不精确. NTP会自动来监测我们时钟的误差值并予以调整.但问题是这是一个冗长的过程,所以它会把记录下来的误差先写入driftfile.这样即使你重新开机以后之前的计算结果也就不会丢失了。

    启动ntp服务

    ```shell
    [root@localhost ~]# /etc/init.d/ntp start

    [root@localhost ~]# chkconfig ntpd on #在运行级别2、3、4、5上设置为自动运行

    [root@localhost ~]# chkconfig --list ntpd

    ntpd 0:off 1:off 2:on 3:on 4:on 5:on 6:off
    ```

    查看ntp服务器有无和上层ntp连通

    ```shell
    [root@localhost ~]# ntpstat
    synchronised to NTP server (192.168.7.49) at stratum 6
    time correct to within 440 ms
    polling server every 128 s
    ```

    查看ntp服务器与ntp的状态

    ```shell
    [root@localhost ~]# watch ntpq -p
        remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
    192.168.7.49    192.168.7.50     5 u   13   64    3    5.853  1137178   2.696

    [root@localhost ~]# ntpq -p

    remote           refid               st     t when  poll  reach     delay     offset     jitter

    ==================================================================

    *10.247.160.31 10.240.241.5     4     u 53      64   377     0.240     0.374     0.240
    ```

    remote：响应这个请求的NTP服务器的名称。“+”表示优先，“*”表示次优先

    refid：NTP服务器使用的上一级ntp服务器。

    st ：remote远程服务器的级别。由于NTP是层型结构,有顶端的服务器,多层的Relay Server再到客户端.所以服务器从高到低级别可以设定为1-16.为了减缓负荷和网络堵塞,原则上应该避免直接连接到级别为1的服务器的。

    when: 上一次成功请求之后到现在的秒数。

    poll : 本地机和远程服务器多少时间进行一次同步(单位为秒).在一开始运行NTP的时候这个poll值会比较小,那样和服务器同步的频率也就增加了,可以尽快调整到正确的时间范围，之后poll值会逐渐增大,同步的频率也就会相应减小

    reach:这是一个八进制值,用来测试能否和服务器连接.每成功连接一次它的值就会增加

    delay:从本地机发送同步要求到ntp服务器的round trip time

    offset：主机通过NTP时钟同步与所同步时间源的时间偏移量，单位为毫秒（ms）。offset越接近于0,主机和ntp服务器的时间越接近

    jitter:这是一个用来做统计的值.它统计了在特定个连续的连接数里offset的分布情况.简单地说这个数值的绝对值越小，主机的时间就越精确

3. ntpd的配置

    配置/etc/sysconfig/ntpd文件

    ntp服务，默认只会同步系统时间。如果想要让ntp同时同步硬件时间，可以设置`/etc/sysconfig/ntpd`文件，在`/etc/sysconfig/ntpd`文件中，添加 `SYNC_HWCLOCK=yes` 这样，就可以让硬件时间与系统时间一起同步。

    #允许BIOS与系统时间同步，也可以通过`hwclock -w` 命令

4. 时间同步

    利用crontab可以让LINUX NTP定时更新时间

    注：让linux运行ntpdate更新时间时，linux不能开启NTP服务，否则会提示端口被占用：如下

    ```shell
    [root@ESXI ~]# ntpdate 1.rhel.pool.ntp.org                                 
    20 May 09:34:14 ntpdate[6747]: the NTP socket is in use, exiting
    ```

    如果想定时进行时间校准，可以使用crond服务来定时执行。

    编辑 /etc/crontab 文件，加入下面一行：

    ```sh
    30 8 * * * root /usr/sbin/ntpdate 192.168.0.1; /sbin/hwclock -w    #192.168.0.1是NTP服务器的IP地址
    ```

    然后重启crond服务

    ```sh
    service crond restart
    ```

    这样，每天 8:30 Linux 系统就会自动的进行网络时间校准。

    如果是WINDOWS ，则需要打开windows time服务和RPC的二个服务，如果在打开windows time服务时报错误1058，进行下面操作:

    1. 运行 cmd 进入命令行，然后键入`w32tm /register`,进行注册，正确的响应为：W32Time 成功注册。

    2. 如果上一步正确，用`net start "windows time"`或`net start w32time`启动服务。

5. Linux的硬件时间

    Linux硬件时间的设置

    硬件时间的设置，可以用hwclock或者clock命令。其中，clock和hwclock用法相近，只用一个就行，只不过clock命令除了支持x86硬件体系外，还支持Alpha硬件体系。

    查看硬件时间 可以是用`hwclock`，`hwclock --show`或者 `hwclock -r`

    ```shell
    [root@localhost ~]# hwclock --show
    2008年12月12日 星期五 06时52分07秒  -0.376932 seconds

    #设置硬件时间
    [root@localhost ~]# hwclock --set --date="1/25/09 00:00" #月/日/年时:分:秒
    [root@localhost ~]# hwclock
    2009年01月25日 星期日 00时00分06秒  -0.870868 seconds
    ```

### ntpd、ntpdate的区别

下面是网上关于ntpd与ntpdate区别的相关资料。如下所示所示：

使用之前得弄清楚一个问题，ntpd与ntpdate在更新时间时有什么区别。ntpd不仅仅是时间同步服务器，它还可以做客户端与标准时间服务器进行同步时间，而且是平滑同步，并非ntpdate立即同步，在生产环境中慎用ntpdate，也正如此两者不可同时运行。

时钟的跃变，对于某些程序会导致很严重的问题。许多应用程序依赖连续的时钟——毕竟，这是一项常见的假定，即，取得的时间是线性的，一些操作，例如数据库事务，通常会地依赖这样的事实：时间不会往回跳跃。不幸的是，ntpdate调整时间的方式就是我们所说的”跃变“：在获得一个时间之后，ntpdate使用settimeofday(2)设置系统时间，这有几个非常明显的问题：

第一，这样做不安全。ntpdate的设置依赖于ntp服务器的安全性，攻击者可以利用一些软件设计上的缺陷，拿下ntp服务器并令与其同步的服务器执行某些消耗性的任务。由于ntpdate采用的方式是跳变，跟随它的服务器无法知道是否发生了异常（时间不一样的时候，唯一的办法是以服务器为准）。

第二，这样做不精确。一旦ntp服务器宕机，跟随它的服务器也就会无法同步时间。与此不同，ntpd不仅能够校准计算机的时间，而且能够校准计算机的时钟。

第三，这样做不够优雅。由于是跳变，而不是使时间变快或变慢，依赖时序的程序会出错（例如，如果ntpdate发现你的时间快了，则可能会经历两个相同的时刻，对某些应用而言，这是致命的）。因而，唯一一个可以令时间发生跳变的点，是计算机刚刚启动，但还没有启动很多服务的那个时候。其余的时候，理想的做法是使用ntpd来校准时钟，而不是调整计算机时钟上的时间。

NTPD 在和时间服务器的同步过程中，会把 BIOS 计时器的振荡频率偏差——或者说 Local Clock 的自然漂移(drift)——记录下来。这样即使网络有问题，本机仍然能维持一个相当精确的走时。