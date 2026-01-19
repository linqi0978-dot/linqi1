# linqi1
项目简介
本项目基于 Cisco Packet Tracer 构建了一个模拟高校多校区互联的广域网安全架构。项目旨在解决传统物理专线成本高、灵活性差的问题，通过部署 Site-to-Site IPSec VPN 实现总校（HQ）与分校（Branches）间的安全加密通信，并引入 ASA 防火墙、AAA 认证、Syslog 日志审计及 NetFlow 流量分析，构建了一个可视化的纵深防御体系。
模块,技术/协议,描述
广域网互联,IPSec VPN,采用 ESP/AES 加密与 SHA 哈希，实现 Site-to-Site 安全隧道 
边界防护,Cisco ASA,部署 ASA 5506-X，配置状态化防火墙策略与 ICMP 穿越 
路由交换,Static Routing,配置静态路由与网关冗余，防止路由黑洞
身份认证,AAA (Radius),均配置本地与 Radius 服务器认证，保护管理平面安全 
态势感知,NetFlow & Syslog,实现流量可视化监控与关键事件日志留存，满足合规性要求 
IP 地址规划
ISP (Internet): 202.100.x.x
HQ (总校): WAN 202.100.1.1 | LAN 192.168.1.1 | DMZ (SOC) 192.168.10.x
Branch 1: 192.168.20.1
Branch 2: 192.168.30.1
部署与测试
前置要求
Cisco Packet Tracer (最新版本)
配置概览
本项目包含详细的设备配置脚本，位于 /configs 目录下。
VPN配置： 使用 crypto isakmp policy (Phase 1) 和 crypto ipsec transform-set (Phase 2) 。
防火墙策略： 默认拒绝所有流量，仅放行 VPN (UDP 500/4500) 和特定管理流量，并修复了 Syslog 被防火墙拦截的问题 
测试结果
连通性测试： 分校 PC 成功 Ping 通总校服务器 。
可视化审计： SOC 服务器成功接收并展示 NetFlow 流量饼图 。
日志审计： 攻击模拟产生的日志成功发送至 Syslog 服务器
配置代码部分
HQ Router Configuration (HQ_Router.txt)
此配置包含基础网络、IPSec VPN、AAA 认证以及 NetFlow/Syslog 审计配置。
!--- 基本设置 ---
hostname HQ
enable secret hnu123

!--- 接口配置 ---
interface s0/0/0
 description Connection to ISP
 ip address 202.100.1.1 255.255.255.252
 ip flow ingress
 ip flow egress
 no shutdown
exit

interface f0/0
 description Connection to HQ-Firewall
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit

!--- 静态路由 ---
! 默认路由指向 ISP
ip route 0.0.0.0 0.0.0.0 202.100.1.2
! 内网(SOC区)路由指向防火墙 Outside 接口
ip route 192.168.10.0 255.255.255.0 192.168.1.2

!--- IPSec VPN Phase 1 (ISAKMP) ---
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 5
exit

! 配置预共享密钥 (针对三个分校)
crypto isakmp key hnu123 address 202.100.2.1
crypto isakmp key hnu123 address 202.100.3.1
crypto isakmp key hnu123 address 202.100.4.1

!--- IPSec VPN Phase 2 ---
crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac

!--- 感兴趣流 ACL (定义哪些流量走 VPN) ---
! 对应 Branch 1
access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
! 对应 Branch 2
access-list 102 permit ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
! 对应 Branch 3
access-list 103 permit ip 192.168.10.0 0.0.0.255 192.168.40.0 0.0.0.255

!--- Crypto Map 映射 ---
crypto map HQ-MAP 10 ipsec-isakmp
 set peer 202.100.2.1
 set transform-set VPN-SET
 match address 101
exit

crypto map HQ-MAP 20 ipsec-isakmp
 set peer 202.100.3.1
 set transform-set VPN-SET
 match address 102
exit

crypto map HQ-MAP 30 ipsec-isakmp
 set peer 202.100.4.1
 set transform-set VPN-SET
 match address 103
exit

! 应用 Crypto Map 到外网接口
interface s0/0/0
 crypto map HQ-MAP
exit

!--- AAA 身份认证配置 ---
aaa new-model
radius-server host 192.168.10.100 key hnu123
! 定义登录认证列表：优先 Radius，失败则本地
aaa authentication login SOC_AUTH group radius local
username linqi privilege 15 secret hnu123
line vty 0 4
 login authentication SOC_AUTH
exit
!--- 审计与日志配置 (NetFlow & Syslog) ---
! NetFlow 导出配置
ip flow-export source f0/0
ip flow-export version 9
ip flow-export destination 192.168.10.100 9996

! Syslog 配置
logging 192.168.10.100
logging trap debugging
logging on
HQ Firewall Configuration (HQ_Firewall_ASA.txt)
此配置包含区域划分、安全级别、访问控制列表（ACL）及 ICMP 穿越检测。
!--- 基本设置 ---
hostname HQ-FW

!--- 接口与区域配置 ---
interface g1/1
 nameif outside
 security-level 0
 ip address 192.168.1.2 255.255.255.0
 no shutdown
exit

interface g1/2
 nameif inside
 security-level 100
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

!--- 路由配置 ---
! 默认路由指向 HQ 路由器 (下一跳)
route outside 0.0.0.0 0.0.0.0 192.168.1.1

!--- 访问控制列表 (ACL) ---
! 放行分校网段访问 SOC 服务器网段 (VPN解密后的流量)
access-list OUT_IN extended permit ip 192.168.20.0 255.255.255.0 192.168.10.0 255.255.255.0
access-list OUT_IN extended permit ip 192.168.30.0 255.255.255.0 192.168.10.0 255.255.255.0
access-list OUT_IN extended permit ip 192.168.40.0 255.255.255.0 192.168.10.0 255.255.255.0

! 放行 ICMP (用于 Ping 测试)
access-list OUT_IN extended permit icmp any any

! *注意：根据文档 5.4 节排错，这里实际上还需要放行 HQ 路由器的 Syslog 流量*
! 建议补充: access-list OUT_IN extended permit udp host 192.168.1.1 host 192.168.10.100 eq 514

! 应用 ACL 到 outside 接口入方向
access-group OUT_IN in interface outside

!--- 策略配置 (ICMP Inspection) ---
! 定义检测流量类
class-map inspection_default
 match default-inspection-traffic
exit

! 定义全局策略
policy-map global_policy
 class inspection_default
  inspect icmp
 exit
exit

! 应用全局策略
service-policy global_policy global
