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
