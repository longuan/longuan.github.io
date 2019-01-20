---
title: "another-laptop-with-manjaro"
---


好长时间没更新了。一是因为自己越来越咸鱼，二是自己好像迷失在了这花花世界。不同于其他人，我的眼睛看到世界是多重的，时常令我目接不暇，眼花缭乱。所以我大多数时间都是闭着眼在消化刚才的匆匆一瞥。这些事情让我觉得自己非同寻常，痛苦也非同寻常。而这又常常令我苦恼，这是指自觉非同寻常。因为每个人都是这样想的，每个人都觉得自己非同寻常。他们认识到自己在觉得自己非同寻常吗？他们知道其他人也是非同寻常的吗？他们怎么看非同寻常的我？如果他们认识到之后，他们怎么想的？这个世界容的下这么多非同寻常吗？这么多的非同寻常应该怎么相处这些个问题快烦死我了。再进一步，不过好在我有自知，还能意识自己正处于这样的状态，还能慢慢调整。希望下下周去看医生之后会有好转。PS.不是心理医生。

回到题目，十一国庆，为了给祖国母亲庆生，同时为了助力AMD吊捶intel，剁手买了一个magicbook。让人出乎意料的是，颜值相当的高，除了把macbook换成magicbook，咬一口的苹果换成honor，其他跟mbp一毛一样。超值有木有！（滑稽）

笔记本拿到手，第一想法是系统换成manjaro。结果悲剧了，竟然装不上。又下载了i3版本的manjaro，还是不行。又试了kali，不行。mint，可以，不过卡卡的，而且也不是滚动更新的。opensuse，可以，就凑合着用了几天，发现字体渲染不如manjaro，就又换到了win10。不过身在曹营心在汉，还是忘不了manjaro。就又开始google解决方法，忽然发现有帖子说ryzen 2500的apu比较新，linux内核4.15以上才支持。这时才明白自己之前安装一直选的4.14稳定版本的内核。恍然大悟，欣然起行，行云流水，开机登录，BINGO！

主要是两个坑：
1. 制作好的u盘进不去live系统，提示u盘未挂载。解决方法就是拔掉再插上。
2. 内核版本的选择，manjaro的mini在线安装可以选择安装的内核版本，选4.18。

剩下的就是系统配置：
1. 增加archlinuxcn的源
2. 安装chrome，sublime，vim，shadowsocks-libev，keepassxc
3. 登录google帐号同步，配置switchyomega，在chrome里面创建网易云、微信、evernote的桌面快捷方式并且move到`~/.local/share/applications/`目录下
4. 安装字体，consolas、monaco、yahei consolas、wqy、noto、source-han、dejavu
5. 其他软件，albert、ohmyzsh、virtualbox、googlepinyin、docker、papirus-icon-theme、aria2、uget

然后就可以愉快的开始工作了。

![desktop](https://i.loli.net/2018/10/12/5bc085e219388.png)