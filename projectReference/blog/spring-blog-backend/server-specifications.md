# 온프레미스 서버 스펙

## 하드웨어 사양
- **CPU**: Intel(R) N100 (4 Cores / 4 Threads)
- **Memory**: 16 GB
- **Swap**: 4.0 GiB
- **Architecture**: x86_64

## 소프트웨어 사양
- **OS**: Ubuntu 24.04.3 LTS
- **Kernel**: Linux 6.8.0-62-generic

## 네트워크 구성
- **DDNS**: No-IP 
  - 공유기 public IP 고정 불가(환경)에 따른 외부 접속 도메인 관리
- **SSL/TLS**: HTTPS 
  - Let's Encrypt(CA) 인증서 적용
- **Local IP**: DHCP Static Lease 
  - 공유기 내 사설 IP 고정
- **Port Forwarding**
  - 80(HTTP), 443(HTTPS) 등 공유기 포트포워딩을 통한 외부 서비스 포트 노출



---



## misc

### 원본



```bash
jin@jin:~$ hostnamectl | grep -E "Operating System|Kernel|Architecture|Hardware"
Operating System: Ubuntu 24.04.3 LTS
          Kernel: Linux 6.8.0-62-generic
    Architecture: x86-64
 Hardware Vendor: Firebat_Computer
  Hardware Model: ZY-AK2PLUS
jin@jin:~$ sudo dmidecode -t memory | grep -E "Size|Type|Speed|Manufacturer" | grep -v "No Module Installed"
[sudo] password for jin: 
        Error Correction Type: None
        Size: 16 GB
        Type: DDR4
        Type Detail: Synchronous
        Speed: 2667 MT/s
        Manufacturer: 0x0E2A
        Configured Memory Speed: 2667 MT/s
        Module Manufacturer ID: Bank 15, Hex 0x2A
        Memory Subsystem Controller Manufacturer ID: Unknown
        Non-Volatile Size: None
        Volatile Size: 16 GB
        Cache Size: None
        Logical Size: None
        Type: Unknown
        Type Detail: None
jin@jin:~$ lscpu | grep -E 'Architecture|Model name|CPU\(s\)|Thread\(s\) per core|Core\(s\) per socket|L1d|L1i|L2|L3|Virtualization'
Architecture:                         x86_64
CPU(s):                               4
On-line CPU(s) list:                  0-3
Model name:                           Intel(R) N100
Thread(s) per core:                   1
Core(s) per socket:                   4
CPU(s) scaling MHz:                   28%
Virtualization:                       VT-x
L1d cache:                            128 KiB (4 instances)
L1i cache:                            256 KiB (4 instances)
L2 cache:                             2 MiB (1 instance)
L3 cache:                             6 MiB (1 instance)
NUMA node0 CPU(s):                    0-3
```

