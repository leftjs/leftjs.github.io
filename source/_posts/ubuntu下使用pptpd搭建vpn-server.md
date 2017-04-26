---
title: ubuntu下使用pptpd搭建vpn-server
date: 2016-11-27 12:58:48
categories: 零碎的记录
tags:
  - 桂林电子科技大学
  - pptpd
  - vpn
  - 网络配置
---

## 简介
由于桂电实验室上网需要 **高额的** 费用，考虑到实验室和宿舍的网络都是基于校园网来进行内网连接的，基于这一点，考虑在宿舍通过笔记本或树莓派设备来启动一个vpn服务器连接内网，然后再在宿舍的设备上将内外网进行桥接，即该设备可以同时上内外网，由于大多数笔记本或者树莓派都具有双网卡，所以采用无线网卡来接入一个已经具备上网功能的路由器，使用有线网卡连接内网。

> 2016-12-15 update
> 最近又被2.4g网络恶心到了，宿舍那边的无线信号真的是太多了，各种信道拥堵，而英产树莓派又不支持**伟大的**13信道和大陆5g频段，所以ping一直都有波动，考虑买大陆的usb无线网卡或有线网卡来接入5g网络或者有线网路提高树莓派外网连接的稳定性，当然宿舍不缺电(桂电宿舍普通用电还要自己交钱,无力吐槽)准备使用笔记本搭建的童鞋当我没说，因为新一点的笔记本都支持5g吧....

## 配置 _永不掉线_ 的路由器
桂电宿舍的网线都是单网线三网通用的，其实原理很简单，就是在上层做了一下网关开放的工作，通过一个叫做出校器的工具，选择一个运营商，将拨号所用的mac地址和相关的运营商进行绑定，此后的所有的数据包都发往指定运营商的网关，基于这个原理，可以在实验室等能够连通内网的设备上定期进行相关端口开放的工作(前提是将该设备的mac修改为路由器的mac地址)，另外，这里推荐一个师大师兄编写的mac、linux、windows下的第三方出校器和端口开放工具[ipclient_gxnu][890a6ed5],项目中的有详细的介绍和文档，推荐学习。此外，如果仅仅是需要进行运营商和mac地址的绑定的话，推荐使用桂电某学生编写的在线端口绑定网站:[http://sec.guet.edu.cn/open/][72f609bc]将路由mac地址和某运营商端口进行绑定

## ubuntu发行版本选择
由于之前我用的是ubuntu的最新的发行版本，在dhcp的时候经常会出现三个默认路由的情况，我也没有深入研究原因，后来换到16.04LTS上后，同样的配置，dhcp后默认只有一个默认路由，所以推荐16.04LTS

## 配置
安装pptpd
```bash
sudo apt-get install pptpd
```
~~配置虚拟ip，编辑/etc/pptpd.conf~~ ( _这一步可以不用做，因为python脚本中会覆盖_ ):
~~```
localip 10.20.39.111 # 本机ip
remoteip 10.100.123.2-100 # 分配的ip段
```~~
在/etc/ppp/pptpd-options下中设置dns:
```
#根据实际情况设置dns
ms-dns 192.168.199.1
ms-dns 114.114.114.114
```
在/etc/ppp/chap-secrets中配置vpn账号:
```
"user"  pptpd   "user"  * #星号是不限制ip的意思
```
不过默认情况下，pptpd无法给vpn连接分配ip，所以如果是多用户的话需要手动分配ip具体配置类似:
```
"user1"   pptpd   "user1"   10.100.123.2
"user2"   pptpd   "user2"   10.100.123.3
```
重启pptpd服务:
```
sudo /etc/init.d/pptpd restart
```
在/etc/sysctl.conf中配置ip转发(取消该行注释):
```
net.ipv4.ip_forward=1
```
使配置立即生效:
```
sudo sysctl -p
```
安装iptables,这个是用于配置NAT映射的:
```
sudo apt-get install iptables
```
建立一个外网NAT（外网走无线）:
```
sudo iptables -t nat -A POSTROUTING -s 10.100.123.0/24 -o wlo1 -j MASQUERADE
```
建立一个内网NAT（内网走有线）:
```
sudo iptables -t nat -A POSTROUTING -s 10.100.123.0/24 -d 172.16.0.0/16 -o eno1 -j MASQUERADE
```
其中-o参数配置的是流量的出口,也就是我这的外网出口
设置MTU,防止包过大而丢包:
```
sudo iptables -A FORWARD -s 10.100.123.0/24 -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200
```
保存规则(此处需要root权限):
```
sudo iptables-save >/etc/iptables-rules
```
编辑/etc/network/interfaces,在末尾加一行,使网卡加载时自动加载规则:
```
per-up iptables-restore </etc/iptables-rules
```
配置到此为止

## 网卡名称修改问题
新版的ubuntu网卡名称都是随机的，可以自定义修改网卡的名称，比如本文章中，有线网卡为:wlo1， 无线网卡为:eno1。
默认，/etc/udev/rules.d/只有一个README文件。
新建了70-persistent-net.rules文件，然后编辑：
```
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="44:33:4c:07:ad:48", NAME="eno1"

```

## 服务器启动代码
### 目的
1. 修改dhcp后的ip为静态ip
2. 添加内外网路由，默认情况下重启后路由表会丢失
3. 持续ping对方网关，保证服务器的稳定，推荐使用tmux来维持会话状态，保证程序存活
### 启动方式
安装 `nodejs` 和 `npm` :
```
sudo apt install nodejs npm
```
切换到 `/usr/bin` 目录下，建立一个 `node` 到 `nodejs` 的软连接，因为 `pm2` 启动时需要执行 `node` 命令:
```
cd /usr/bin
sudo ln -s ./node.js node
```
使用 `npm` 安装 `pm2` :
```
sudo npm install -g pm2
```
使用 `pm2` 启动项目，并将其加入开启自启中:
```
cd vpn_launch
pm2 start ./launch.py --name="vpn"
pm2 save
pm2 startup
```
至此，您的pc、树莓派已经具有了开机自启、daemon、线路自检等功能，理论上已经不会掉线
代码地址:[leftjs/vpn_launch][911b7533]

launch.py:
```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
import socket, fcntl, struct
import commands
import time
import smtplib
from email.mime.text import MIMEText
from email.header import Header

# 第三方 SMTP 服务
mail_host="smtp.qq.com"  #设置服务器
mail_user="leftjs@foxmail.com"    #用户名
mail_pass="xxxx"   #授权口令
sender = 'leftjs@foxmail.com'
receivers = ['lefttjs@gmail.com']  # 接收邮件，可设置为你的QQ邮箱或者其他邮箱



def send_email(text):
    message = MIMEText('%s' % text, 'plain', 'utf-8')
    message['From'] = Header("leftjs", 'utf-8')
    message['To'] =  Header("jason zhang", 'utf-8')
    subject = 'vpn 报告'
    message['Subject'] = Header(subject, 'utf-8')
    try:
        smtpObj = smtplib.SMTP_SSL()
        smtpObj.connect(mail_host, 465)    # 25 为 SMTP 端口号
        smtpObj.login(mail_user,mail_pass)
        smtpObj.sendmail(sender, receivers, message.as_string())
        print "邮件发送成功"
    except smtplib.SMTPException:
        print "Error: 无法发送邮件"

def get_local_ip(ifname):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        inet = fcntl.ioctl(s.fileno(), 0x8915, struct.pack('256s', ifname[:15]))
        return socket.inet_ntoa(inet[20:24])
    except Exception, e:
        send_email(str(e))
        return

def check_ping():
    (ping_state, res) = commands.getstatusoutput('ping 202.193.75.254 -c 2')
    # (ping_state_baidu, res_baidu) = commands.getstatusoutput('ping www.baidu.com -c 2')
    # ping_state == 0 when ping is ok
    # return True if ping_state == 0 and ping_state_baidu == 0 else False
    return True if ping_state == 0 else False

def read_file_content(file_name):
    the_file = open(file_name, 'r')
    file_content = the_file.read()
    the_file.close()
    return file_content

def write_file_content(file_name, new_content):
    out_file = open(file_name, 'w')
    out_file.write(new_content)
    out_file.close()

def create_interfaces_static_file(intranet_ip):
    raw_content = read_file_content('./interfaces/raw_interfaces_static')
    write_file_content('./interfaces/interfaces_static', raw_content % intranet_ip)

def restart_pptpd(intranet_ip):
    raw_content = read_file_content('./pptpd/raw_pptpd.conf')
    write_file_content('./pptpd/pptpd.conf', raw_content % intranet_ip)
    commands.getstatusoutput('cp -f ./pptpd/pptpd.conf /etc/pptpd.conf')
    commands.getstatusoutput('sudo /etc/init.d/pptpd restart')

def restart_dnsmasq(intranet_ip):
    raw_content = read_file_content('./dns/raw_dnsmasq.conf')
    write_file_content('./dns/dnsmasq.conf', raw_content % intranet_ip)
    commands.getstatusoutput('cp -f ./dns/dnsmasq.conf /etc/dnsmasq.conf')
    commands.getstatusoutput('service dnsmasq restart')

def add_route_item():
    commands.getstatusoutput('ip route add 202.193.0.0/16 via 10.20.40.254 dev eno1')
    commands.getstatusoutput('ip route add 10.100.123.0/24 via 10.20.40.254 dev eno1')
    commands.getstatusoutput('ip route add 10.20.0.0/16 via 10.20.40.254 dev eno1')
    commands.getstatusoutput('ip route add 172.16.0.0/16 via 10.20.40.254 dev eno1')


if __name__ == '__main__':
    old_ip = None
    while True:
        ping_state = check_ping()
        if ping_state == False or old_ip == None:
            print time.strftime('%Y-%m-%d %H:%M:%S',time.localtime(time.time()))
            print 'network restart.'
            commands.getstatusoutput('dhclient eno1')
            # get new intranet ip
            intranet_ip = get_local_ip("eno1")
            if intranet_ip is None:
                continue
            if old_ip == None or old_ip != intranet_ip:
                print 'new ip: ',intranet_ip
                send_email('新的ip为: %s' % intranet_ip)
                create_interfaces_static_file(intranet_ip)
                restart_pptpd(intranet_ip)
                # restart_dnsmasq(intranet_ip)
                old_ip = intranet_ip
            commands.getstatusoutput('cp -f ./interfaces/interfaces_static /etc/network/interfaces')
            commands.getstatusoutput('/etc/init.d/networking restart')
            # commands.getstatusoutput('dhclient eth0')
            # commands.getstatusoutput('cp -f ./dns/resolv.conf /etc/resolv.dnsmasq.conf')
            add_route_item()
            time.sleep(1)
            print 'config complete.'
            continue
        time.sleep(2)


```

## 寝室ip扫描代码
在实验室连入校内网批量ping宿舍ip，找出自己的主机。

ping_test.py:
```python
from threading import Thread
import subprocess
from Queue import Queue

num_threads = 100
queue = Queue()
ips = ['10.20.38.' + str(a) for a in range(1, 255)] + ['10.20.39.' + str(a) for a in range(1, 255)]

#wraps system ping command
def pinger(i, q):
    """Pings subnet"""
    while True:
        ip = q.get()
        print "Thread %s: Pinging %s" % (i, ip)
        ret = subprocess.call("ping -c 1 %s" % ip,
            shell=True,
            stdout=open('/dev/null', 'w'),
            stderr=subprocess.STDOUT)
        if ret == 0:
            print "%s: is alive" % ip
            with open('./result.txt', 'a') as f:
                f.write("%s: is alive\n" % ip)
        else:
            print "%s: did not respond" % ip
        q.task_done()
#Spawn thread pool
for i in range(num_threads):
    worker = Thread(target=pinger, args=(i, queue))
    worker.setDaemon(True)
    worker.start()
#Place work in queue
for ip in ips:
    queue.put(ip)
#Wait until worker threads are done to exit
queue.join()
```

  [911b7533]: https://github.com/leftjs/vpn_launch "leftjs/vpn_launch"
  [890a6ed5]: https://github.com/xuzhipengnt/ipclient_gxnu "ipclient_gxnu"
  [72f609bc]: http://sec.guet.edu.cn/open/ "http://sec.guet.edu.cn/open/"
