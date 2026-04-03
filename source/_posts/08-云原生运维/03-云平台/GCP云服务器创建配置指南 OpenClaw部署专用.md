---
tags:
  - GCP
  - OpenClaw
  - 云服务器
  - 成本优化
title: GCP云服务器创建配置指南 OpenClaw部署专用
categories:
  - 08-云原生运维
  - 03-云平台
description: 详细介绍如何在GCP上创建适合OpenClaw部署的VM实例，包含机器配置、操作系统选择、网络配置及成本优化建议，帮助在$300预算内稳定运行3个月。
abbrlink: 5949fa3d
date: 2026-04-03 10:30:00
---

# GCP云服务器创建配置指南：OpenClaw部署专用

> [!NOTE]
> 本指南旨在指导用户在 GCP 上配置一台性能稳定、安全且符合预算（3个月 $300）的虚拟机，用于运行 OpenClaw 智能体。

---

## 1. Machine Configuration (机器配置)

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **Machine Family** | General-purpose (通用型) | - |
| **Series** | E2 | 性价比之王，适合 Agent 任务 |
| **Machine type** | `e2-standard-4` (4 vCPU, 16 GB RAM)<br>或 `e2-standard-2` (2 vCPU, 8 GB RAM) | 16GB 内存是运行浏览器自动化和多任务调度的安全线，无浏览器自动化可选 8GB |
| **Provisioning model** | Standard | 不要选 Spot，确保 24/7 在线 |
| **Confidential VM Service** | ❌ 不勾选 | 避免性能损耗和额外费用 |

### Advanced Configurations (高级配置)

| 配置项 | 推荐值 |
|--------|--------|
| **On host maintenance** | Migrate VM instance (自动迁移) |
| **Automatic restart** | On (意外关机自动重启) |

---

## 2. Operating System & Storage (操作系统与存储)

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **Operating system** | Ubuntu | - |
| **Version** | Ubuntu 24.04 LTS (x86/64) | 不要选 Pro 版，避免额外订阅费 |
| **Boot disk type** | Balanced persistent disk | 平衡持久性磁盘 |
| **Size (GB)** | 50 | 为 Docker 镜像、浏览器内核及日志预留充足空间 |
| **Encryption** | Google-managed encryption key | 免费且高效 |
| **Deletion rule** | Delete disk | 实例删除时同步释放，避免隐形成本 |

---

## 3. Data Protection (数据保护)

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **Deletion protection** | ✅ 勾选开启 | 防止手滑误删实例 |
| **Backups** | ❌ 不勾选 (No backups) | 避免高昂的企业级托管费用，推荐后续手动创建快照 |
| **Replication** | None 或 Zonal | 避免存储费用翻倍 |

---

## 4. Networking (网络配置)

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **Network tags** | `openclaw-server` | 用于精确防火墙控制 |
| **External IPv4 address** | RESERVE STATIC IP ADDRESS | 命名为 `openclaw-ip` 并保留 |
| **Network Service Tier** | Premium | 保证访问 Gemini API 的低延迟 |
| **Firewall (HTTP/HTTPS)** | ❌ 不要勾选 | 通过自定义防火墙规则只允许你的个人 IP 访问 `18789` 端口，实现真正的安全隔离 |

---

## 5. Security (安全配置)

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **Secure Boot** | ✅ 勾选 | - |
| **vTPM** | ✅ 勾选 | - |
| **Integrity Monitoring** | ✅ 勾选 | 以上三项为 Shielded VM 功能 |
| **Access scopes** | Allow full access to all Cloud APIs | 确保 OpenClaw 调用 Gemini API 时拥有完整权限 |
| **VM access (IAM)** | ❌ 不勾选 | 直接在 **SSH Keys** 中点击 **ADD ITEM** 并粘贴你的 Xshell 公钥 |

---

## 6. Advanced Options (高级选项)

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **Reservations** | Don't use any reservation | 避免产生占位费 |
| **Data encryption** | Google-managed key | 再次确认选择 |

---

## 💰 预算与配额优化建议

### 运行策略

1. **首月运行**: 按照 `e2-standard-4` 配置运行。
2. **模型选择**: 推荐以 **Gemini 3.1 Flash** 为主（利用 Tier 1 每天 1500 次的免费额度），仅在复杂逻辑时切换到 3.1 Pro。
3. **降级观察**: 若首月内存占用率长期低于 40%，次月可在关机状态下将机器改为 `e2-standard-2`，费用可立省一半。

### 预估成本

| 配置 | 月费用 | 3 个月费用 |
|-----|--------|----------|
| `e2-standard-4` | ~$75-90 | ~$225-270 |
| `e2-standard-2` | ~$40-50 | ~$120-150 |

---

> [!TIP]
> 这份指南涵盖了创建界面的每一个关键参数，按照此配置可在 $300 预算内稳定运行 3 个月。
