---
title: "Step by Step"

---



## Step 0 —— Linux and scripting

- **阅读列表**
  - [Hypertext Transfer Protocol](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)
  - [Domain Name System](http://en.wikipedia.org/wiki/Domain_Name_System)
  - [Whois](http://en.wikipedia.org/wiki/Whois)
  - [Network socket](http://en.wikipedia.org/wiki/Network_socket)
- **动手实践**
  - 安装Linux:  选择使用一款虚拟机软件(VirtualBox, VM Player) 并且安装Linux。使用像Ubuntu那样的传统发行版，而不是侧重安全的发行版。
  - 学习一个脚本语言的基础:  从Ruby ([Try Ruby](http://tryruby.org/))，Python ([在线运行](http://repl.it/languages/Python))，或者Perl中选择一个，学习他的语法和数据类型。这是一个持之以恒的过程。



---------------



## Step 1 —— HTTP

- **阅读列表**
  - [TCP/IP](http://en.wikipedia.org/wiki/Internet_protocol_suite)
  - [Secure Sockets Layer](http://en.wikipedia.org/wiki/Secure_Sockets_Layer)
- **动手实践**
  - 在你的虚拟机内安装apache，用vim修改主机的首页。用你的浏览器访问这个页面。
  - 修改你的host文件，使得用"vulnerable"这个名称访问到Linux系统。
  - 用一个http库写一个HTTP客户端，获取你站点的首页。
  - 用socket写一个HTTP客户端，获取你网站的首页。
  - 下载[Burp Suite](http://www.portswigger.net/burp/downloadfree.html)，访问一个网站，看看发送了什么请求收到了什么响应。



-----------------



## Step 2 —— PHP and DNS

- **阅读列表**
  - [了解虚拟主机](http://en.wikipedia.org/wiki/Virtual_hosting)
  - [用apache配置虚拟主机](http://httpd.apache.org/docs/2.2/vhosts/name-based.html)
  - [区域传送](http://www.digininja.org/projects/zonetransferme.php)
- **动手实践**
  - PHP基础：
    - 在你之前装过apache的虚拟机中安装PHP，写一个脚本回显url的参数。比如访问http://vulnerable/hello.php?name=Louis会返回"Hello Louis"。
    - 安装mysql并写一个从中读取信息的脚本。比如article.php?id=1返回一本书，article.php?id=2返回一个电脑。
    - 创建一个页面，用POST请求发送数据到它自身。
  - DNS和whois：
    - 在你的虚拟机里安装命令行工具dig。
    - 找到PentesterLab所用的name servers和mail servers，和www.pentesterlab.com的ip地址。
    - 用whois获取pentesterlab.com的信息。

------------



## Step 3 —— SSL/TLS

- **阅读列表**
  - [SQL injection](http://en.wikipedia.org/wiki/SQL_injection)
  - [XSS](https://en.wikipedia.org/wiki/Cross-site_scripting)
  - [Remote File Inclusion](http://en.wikipedia.org/wiki/Remote_file_inclusion)
- **动手实践**
  - 安装SSL：
    - 在你的web服务器上启用HTTPs
    - 确保你禁用所有弱密码
  - 玩转SSL：
    - 用HTTP库写一个SSL客户端
    - 用socket写一个SSL客户端
    - 用之前的HTTP脚本访问你的SSL服务器，并用socat做"socket<--->ssl-socket"的连接



------------------

## Step 4 —— SQL injection & Local File Include

- **阅读列表**
  - [MIME](http://en.wikipedia.org/wiki/MIME)
- **动手实践**
  - 读[《From SQL injection to Shell》](https://pentesterlab.com/exercises/from_sqli_to_shell/)并在ISPO上测试
  - 读[《PHP Include And Post Exploitation》](https://pentesterlab.com/exercises/php_include_and_post_exploitation/)并在ISO上测试



-----------

## Step 5 —— More SQL injection

- **阅读列表**
  - [Antisec Movement](http://en.wikipedia.org/wiki/Antisec_Movement)
  - [DHCP](http://en.wikipedia.org/wiki/DHCP)
  - [FTP](http://en.wikipedia.org/wiki/FTP)
  - [Request for Comments](http://en.wikipedia.org/wiki/Request_for_Comments)
- **动手实践**
  - 用Burp帮助调试编写脚本，完成 [From SQL injection to Shell](https://pentesterlab.com/exercises/from_sqli_to_shell/)
  - 不阅读教程完成 [From SQL injection to shell: PostgreSQL edition](https://pentesterlab.com/exercises/from_sqli_to_shell_pg_edition/) 
  - 检查你在Step 2写的代码是否有SQL和XSS漏洞



-----------

## Step 6 —— FTP and traffic analysis

- **阅读列表**
  - [Phrack](http://en.wikipedia.org/wiki/Phrack)
  - [Phrack: Happy Hacking](http://phrack.org/issues.html?issue=68&id=7#article)
  - [Phrack profile on FX](http://phrack.org/issues.html?issue=68&id=2#article)
- **动手实践**
  - 安装使用wireshark：探测你的HTTP客户端和HTTPS客户端发送的流量
  - FTP：
    - 安装FTP
    - 用socket写一个FTP客户端

-------------



## Step 7 —— Linux review and Code Exec

- **阅读列表**
  - [Iptables](http://en.wikipedia.org/wiki/Iptables)
  - [Internet Control Message Protocol](http://en.wikipedia.org/wiki/Internet_Control_Message_Protocol)
  - [Cryptography](http://en.wikipedia.org/wiki/Cryptography)
  - [Cryptographic hash function](http://en.wikipedia.org/wiki/Cryptographic_hash_functions)
- **动手实践**
  - [Introduction to Linux Host Review](https://pentesterlab.com/exercises/linux_host_review/)阅读课程完成镜像
  - [CVE-2012-1823: PHP CGI](https://pentesterlab.com/exercises/cve-2012-1823/)阅读课程完成镜像

------------------



## Step 8 —— HTTP server and Firewalling

- **阅读列表**
  - [C (programming language)](http://en.wikipedia.org/wiki/C_(programming_language))
  - [Nmap](http://en.wikipedia.org/wiki/Nmap)
  - [Setuid](http://en.wikipedia.org/wiki/Setuid)
- **动手实践**
  - 写一个HTTP服务器（使用fork来处理连接）
  - 启用禁用iptables。用iptables拦截ICMP请求，测试是否ping的通



--------------

## Step 9 —— Nmap and Crypto Attacks

- **阅读列表**
  - [Wifi](http://en.wikipedia.org/wiki/Wifi)
  - [WEP](http://en.wikipedia.org/wiki/Wired_Equivalent_Privacy)
  - [WPA](http://en.wikipedia.org/wiki/Wi-Fi_Protected_Access)
- **动手实践**
  - nmap
    - 用nmap找到你虚拟机的开放端口
    - 用iptables拦截ICMP请求，再次找开放的端口
    - 用iptables关闭一个开放的端口，用nmap检查是否生效
  - 找一个本地安全沙龙去看看
  - [CVE-2008-1930: Wordpress 2.5 Cookie Integrity Protection Vulnerability](https://pentesterlab.com/exercises/cve-2008-1930/)阅读课程并完成镜像

------------------



## Step 10 —— WIFI

- **阅读列表**
  - [Environment Variables](https://wiki.archlinux.org/index.php/Environment_Variables)
  - [Network Time Protocol](http://en.wikipedia.org/wiki/Network_Time_Protocol)
  - [SMB](http://en.wikipedia.org/wiki/Server_Message_Block)
- **动手实践**
  - 用WEP建立一个WIFI并破解它
  -  [Rack Cookies and Commands Injection](https://pentesterlab.com/exercises/rack_cookies_and_commands_injection/)阅读课程并完成ISO

------------------



## Step 11 —— Linux Exploitation

- **阅读列表**
  - [Memory management](http://en.wikipedia.org/wiki/Memory_management)
  - [Stack](http://en.wikipedia.org/wiki/Call_stack)
  - [Stack protection](http://en.wikipedia.org/wiki/Stack_protection)
- **动手实践**
  - 解决[exploit-exercises](http://exploit-exercises.com/)的Nebula的levels 00 到04

------------------



## Step 12 —— SSL Pinning and Linux Exploitation

- **阅读列表**
  - [Public key pinning](https://www.imperialviolet.org/2011/05/04/pinning.html)
  - [Your app shouldn't suffer SSL's problems](http://www.thoughtcrime.org/blog/authenticity-is-broken-in-ssl-but-your-app-ha/)
  - [Guardian's StrongTrustManager Vulnerabilities](http://www.thoughtcrime.org/blog/strongtrustmanager-mitm/)
- **动手实践**
  - 解决[Nebula](http://exploit-exercises.com/nebula)  [exploit-exercises](http://exploit-exercises.com/)的levels 05 到09
  - 完成[From SQL injection to SHELL 2](https://pentesterlab.com/exercises/from_sqli_to_shell_II/)

----------------------



## Step 13 —— Web For Pentester

- **阅读列表**
  - 阅读https://pentesterlab.com/exercises/web_for_pentester/
- **动手实践**
  - 解决[Nebula](http://exploit-exercises.com/nebula)  [exploit-exercises](http://exploit-exercises.com/)的levels 10 到14
  - 完成[Web For Pentester](https://pentesterlab.com/exercises/web_for_pentester/)

------------------



## Step 14 —— Web For Pentester II

-  **动手实践**
  - 解决[Nebula](http://exploit-exercises.com/nebula)  [exploit-exercises](http://exploit-exercises.com/)的levels 15 到19
  - 完成[Web For Pentester II](https://pentesterlab.com/exercises/web_for_pentester_II/)



--------------



来自[pentesterlab](https://pentesterlab.com)
