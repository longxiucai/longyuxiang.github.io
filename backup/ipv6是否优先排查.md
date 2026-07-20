## 异常情况
`getent ahosts xxx`时IPV4排前面
`ip a `有`dadfailed tentative`
```
[root@nginx1 ~]# getent ahosts  lyx.nginx.com
10.44.70.210    STREAM lyx.nginx.com
10.44.70.210    DGRAM  
10.44.70.210    RAW    
2003:db8::11    STREAM 
2003:db8::11    DGRAM  
2003:db8::11    RAW    
[root@nginx1 ~]# ip -6 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 2003:db8::18/64 scope global dadfailed tentative noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fec5:cf5a/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
[root@nginx1 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:c5:cf:5a brd ff:ff:ff:ff:ff:ff
    inet 10.44.70.219/24 brd 10.44.70.255 scope global noprefixroute ens3
       valid_lft forever preferred_lft forever
    inet6 2003:db8::18/64 scope global dadfailed tentative noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fec5:cf5a/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
## 正常情况
IPV6在前面且IPV6地址正常
```
[root@master2 ~]# getent ahosts lyx.nginx.com
2003:db8::11    STREAM lyx.nginx.com
2003:db8::11    DGRAM  
2003:db8::11    RAW    
10.44.70.210    STREAM 
10.44.70.210    DGRAM  
10.44.70.210    RAW    
[root@master2 ~]# ip -6 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 2003:db8::12/64 scope global noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fec8:ba36/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
## 验证V6优先
curl命令`-v`输出`* Trying <ipv6地址>`
```
[root@master2 ~]# curl -v http://lyx.nginx.com:33116
*   Trying 2003:db8::11:33116...
* Connected to lyx.nginx.com (2003:db8::11) port 33116 (#0)
> GET / HTTP/1.1
> Host: lyx.nginx.com:33116
> User-Agent: curl/7.71.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.23.1
< Date: Wed, 15 Jul 2026 02:28:35 GMT
< Content-Type: text/html
< Content-Length: 615
< Last-Modified: Tue, 19 Jul 2022 14:05:27 GMT
< Connection: keep-alive
< ETag: "62d6ba27-267"
< Accept-Ranges: bytes
< 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host lyx.nginx.com left intact
```