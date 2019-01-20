

I found a stack-based buffer overflow vulnerability in the Tenda router's webserver. While processing the `list` parameters for a post request to `/goform/SetStaticRouteCfg`, the value is directly `sscanf` to local variables placed on the stack, which can override the return address of the function. The attackers can construct an exploit to execute arbitrary binary code. The information about this vulnerability is shown below:


## Overview

Vendor: Tenda

Product:
- AC6V1.0 device, Firmware V15.03.05.16 and earlier version
- AC6V2.0 device, Firmware V15.03.06.51 and earlier version
- AC18 device, Firmware V15.03.05.19(6318) and earlier version
- AC10UV1.0 device, Firmware V15.03.06.49 and earlier version
- AC9V1.0 device, Firmware V15.03.05.19 and earlier version
- AC9V3.0 device, Firmware V15.03.06.42 and earlier version

Vulnerability type: buffer overflow

Discoverer: longuan@iie 


## Details

1. `websGetVar()` retrieves the content of `list` from a HTTP POST request, then pass it to `sub_785D0()`.

![image](/vulns/Tenda/images/details-1-1.png)

2. `sub_785D0()` parse the content of `list` directly using `sscanf()` to local variables placed on the stack, causing buffer overflow.

![image](/vulns/Tenda/images/details-1-2.png)


## PoC

1. The exploitable POST request is show below:

![image](/vulns/Tenda/images/poc-1-1.png)

2. The process of `httpd` crash and restart. 

![image](/vulns/Tenda/images/poc-1-2.png)


