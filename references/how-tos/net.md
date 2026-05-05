# 网络/SOCK/COM/WMI — 写法示例

> **版本兼容性：** 部分示例使用 DLL 调用（`CALL $--qd`）。DLL 调用需要 PECMD2012 v1.88+ 完整版支持（32-bit 和 64-bit 行为一致）。
> 如果 DLL 调用不工作，可使用 `EXEC*`（外部命令）、`REGI`（注册表）等内置命令替代。

## 16. 网络操作

### 获取网卡 IP

```wcs
EXEC* &&ipcfg=!ipconfig                          // EXEC* 捕获标准输出; ! = 隐藏执行
FORX *NL &ipcfg,&&line,
{
    SED &&found=1,IPv4,,%&line%
    IFEX $1=%&found%,
    {
        MSTR &&ip=<-1>%&line%                   // last field = IP address
        MSTR &&ip=<2>%&ip%                       // strip leading space
        MESS IP: %&ip%
    }
}
```

### WiFi 扫描与连接

```wcs
ADSL-wlan ,,list,&&wifiInfo                     // scan nearby networks
ADSL-wlan ,,scan,&&detail                        // detailed scan
ADSL-wlan MyNetwork,mypassword,WPA2PSK          // connect (plain text SSID/password)
ADSL-wlan -start MyNetwork,mypassword           // connect and start
```

### Ping 检测

```wcs
EXEC* &result=!ping -n 1 192.168.1.1
FIND TTL=,%&result%,MESS Host reachable             // substring search (no $ = contains)
```

---

## 29. NET 与网卡操作

```wcs
// Full network adapter query
PCIP ?* IP,MASK,GW,DNS,%NIC%?NAME,MAC,LINK,DHCP,bDHCP,STATUS,MEDIA,DESC,TYPE

// Set static IP
PCIP 192.168.1.100,255.255.255.0,192.168.1.1,192.168.1.1

// DHCP
PCIP DHCP
```

---

## 65. SOCK 网络（TCP/UDP 客户端-服务器）

### TCP 服务器（带 accept 循环）

```wcs
SOCK sk                                    // server listen socket (TCP default)
ENVI @sk.sock=&&err                        // create socket
ENVI#$ &&v=1
ENVI @sk.setsockopt=;;%&SO_REUSEADDR%,&&v  // SO_REUSEADDR
ENVI @sk.bind=&&err;%&MYIP%;%&MYPORT%      // bind to IP:port
ENVI @sk.listen=&&err;1                     // listen (backlog=1)

THREAD* CALL Server

_SUB Server
    SOCK sr                                // accept socket
    ENVI @sk.fd=&&fd                       // get listen fd
    ENVI @sr.accept=&&err;%&fd%            // accept connection
    ENVI @sr.getname=;1;&&remoteIP         // get remote IP

    // Read loop
    ENVI @sr.read=&&err;&Len;&BRMSG        // read data
    IFEX $%&Len%>0, ENVI @this.SENDMSG=#1  // notify main thread

    // Send response
    ENVI @sr.write=&&err;&Len;&Response    // write data
_END
```

### TCP 客户端与 UDP

```wcs
SOCK sc                                    // client socket
ENVI @sc.sock=&&err
ENVI @sc.connect=&&err;%&TOIP%;%&TOPORT%   // connect
ENVI @sc.write=&&err;&Len;&MSG             // send data
ENVI @sc.read=&&err;&Len;&recvBuf          // receive
ENVI @sc.shutdown=                         // graceful shutdown
ENVI @sc.close=                            // close

// UDP
SOCK su;;%&SOCK_DGRAM%;%&IPPROTO_UDP%     // UDP socket
ENVI @su.write=&&err;&Len;&data;;;&destIP  // sendto
ENVI @su.read=&&err;&Len;&recvBuf;;;&srcIP // recvfrom
```

### 共享内存与命名管道

```wcs
SOCK --shm shm1;w;MySharedMem;1024         // writable shared memory
ENVI @shm1.mem=&&addr                      // get memory address

SOCK --pipe pip1;MyPipe;5000;4096;0x1      // named pipe, immediate connect
ENVI @pip1.write=;;&data                   // write
ENVI @pip1.read=;;&buf                     // read
```

---

## 66. COM/WMI 对象自动化

### 通过 CoCreateInstance 创建 COM 对象

```wcs
LOCK .com**                                // initialize COM
SOCK --unknown &&pObj                      // IUnknown pointer (auto-release)

SET$# &CLSID=*16 0
SET$# &IID=*16 0
CODE *,%&CLSID_HEX%,*HEX,&CLSID
CODE *,%&IID_HEX%,*HEX,&IID

CALL $--qd --16 OLE32.DLL,CoCreateInstance,*&CLSID,#0,#1,*&IID,*&pObj

// Call vtable method (index 3)
CALL $--16 --ret:&&hr #,*&pObj.%&iMethod%,arg1,arg2
```

### WMI 查询模式

```wcs
LOCK .com**
SOCK --unknown &&pLoc                       // IWbemLocator
SOCK --unknown &&pSvc                       // IWbemServices

CALL $--qd --16 OLE32.DLL,CoCreateInstance,*&CLSID_WbemLocator,#0,#1,*&IID_IWbemLocator,*&pLoc

CALL $--16 --qd #,*&pLoc.%&iConnectServer%,$ROOT\CIMV2,#0,#0,#0,#0,#0,#0,*&pSvc

CALL $--16 --ret:&&hr --qd #,*&pSvc.%&iExecQuery%,$WQL,*&wqlCmd,#0x30,#0,*&pEnum

SOCK --BSTR &&bstrProp,,PropertyName
SOCK --unknown &&pRow
CALL $--16 --qd #,*&pEnum.%&iNext%,#0xFFFFFFFF,#1,*&pRow,*&count
SET$# &vProp=*24 0
CALL $--16 --ret:&&hr --qd #,*&pRow.%&iGet%,#%&bstrProp?ptr%,#0,*&vProp,#0,#0
SET &value=%&vProp?ptr:8
```

### ITaskbarList3（任务栏进度条）

```wcs
SOCK --unknown &&pTaskbar
CALL $--qd --16 --ret:&&hr OLE32.DLL,CoCreateInstance,*&CLSID_TaskbarList,#0,#1,*&IID_ITaskbarList3,*&pTaskbar
CALL $--ret:&&r #,*&pTaskbar.%&iHrInit%
CALL $--ret:&&r --qd# #,*&pTaskbar.%&iSetValue%,%&hwnd%,0,%&total%
CALL $--ret:&&r #,*&pTaskbar.%&iRelease%
```

---

### 45. GetIfTable — 网络速率监控

```wcs
ENVI$ &&buf=*0x1000 0
SET$# &&dwSize=*4 0
CALL $--qd --ret:&&bret Iphlpapi.dll,GetIfTable,*&buf,*&dwSize,#0
// If GetLastError==122 (ERROR_INSUFFICIENT_BUFFER), re-allocate with returned size
ENVI-addr ;&&bufsize=&&buf
// MIB_IFENTRY: 860 bytes per entry, name at offset 0, dwInOctets at offset 344, dwOutOctets at offset 356
// Filter: skip dwType==24 (loopback), keep dwOperStatus==5 (operational)
// Delta: save previous values, subtract for speed
// CalcSize helper: B→KB→MB→GB→TB→PB→EB→ZB
```

---

### 48. WiFi 连接 + 托盘 UI 模式

Combined ADSL-wlan + TABL + minimize-to-tray typical pattern:

```wcs
// Scan WiFi networks
ADSL-wlan ,,scan,&&result
// result format: one per line, TAB-delimited fields
// Parse into TABL for display
TABL TABL1,L10T10W400H200,SSID:150 Signal:60 Flags:80 Type:60
FORX *NL &result,&&line,
{   MSTR &&ssid,&&signal,&&flags,&&type=<1><2><3><4>%&line%
    ENVI @TABL1.ADD=%&ssid%;%&signal%;%&flags%;%&type%
}

// Connect to selected SSID
ADSL-wlan %&ssid%,%&password%,,

// Minimize to tray
_SUB OnClose
    ENVI @@Visible=::0        // hide window
    // Show tray icon with notification
_END
```

---


