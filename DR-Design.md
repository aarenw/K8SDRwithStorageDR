# K8S DR with Storage DR — 完整灾难恢复方案设计

## 1. 方案概述

### 1.1 业务背景
制造业工厂环境，IT 规模有限但对业务连续性要求极高。所有系统都必须能在灾难发生后快速恢复。

### 1.2 设计目标

| 指标 | 目标值 |
|------|--------|
| RPO（数据丢失容忍）| 0（Powerstore Metro 同步复制） |
| RTO（恢复时间目标）| ≤ 1 小时 |
| Failover 模式 | 半自动（管理员通过 AAP 确认后自动执行） |
| VM 规模 | 200+ 虚拟机 |
| 容器应用规模 | 20+ 有状态应用 |
| 自动化平台 | Ansible Automation Platform 2.7 |

### 1.3 架构总览

```
                         ┌─── GSLB / F5 GTM ───┐
                         │   (DNS 自动切换)      │
                         ▼                      ▼
              ┌──────────────────┐    ┌──────────────────┐
              │      DC1 (主)     │    │    DC2 (备-温备)   │
              │                  │    │                  │
              │  OpenShift 4.x   │    │  OpenShift 4.x   │
              │  + OCP Virt      │    │  + OCP Virt      │
              │                  │    │                  │
              │  ┌────────────┐  │    │  ┌────────────┐  │
              │  │ 200+ VMs   │  │    │  │ VM 定义预置  │  │
              │  │ 20+ 容器应用 │  │    │  │ (未启动)    │  │
              │  └────────────┘  │    │  └────────────┘  │
              │                  │    │                  │
              │  Dell CSI Driver │    │  Dell CSI Driver │
              │       (FC)       │    │       (FC)       │
              └────────┬─────────┘    └────────┬─────────┘
                       │                       │
              ┌────────┴─────────┐    ┌────────┴─────────┐
              │  Dell Powerstore  │◄──►│  Dell Powerstore  │
              │  (Active)         │    │  (Passive/Mirror) │
              │  Metro 同步复制    │    │  LUN ID 一致     │
              │  RPO = 0         │    │                  │
              └──────────────────┘    └──────────────────┘

                    ┌──────────────────────────┐
                    │  Ansible Automation       │
                    │  Platform 2.7             │
                    │  ┌────────────────────┐   │
                    │  │ Workflow Template   │   │
                    │  │ DR-Failover        │   │
                    │  │  ├─ 存储切换        │   │
                    │  │  ├─ VM恢复(分批)    │   │
                    │  │  ├─ 容器恢复        │   │
                    │  │  ├─ 网络切换        │   │
                    │  │  └─ 验收            │   │
                    │  └────────────────────┘   │
                    └──────────────────────────┘
```

## 2. 基础架构设计

### 2.1 存储层

#### Dell Powerstore Metro 配置
- **复制模式**：Metro（同步复制），RPO = 0
- **连接协议**：FC（光纤通道）
- **LUN 映射**：两端 LUN ID 保持一致
- **CSI 驱动**：Dell CSI PowerStore Driver，两个集群均安装配置

#### 存储卷分类

| 卷类型 | 用途 | Metro 复制 | 预估数量 |
|--------|------|-----------|---------|
| VM rootdisk | 虚拟机系统盘（DataVolume/PVC） | ✅ | 200+ |
| VM data disk | 虚拟机数据盘 | ✅ | ~100 |
| Container PV | 有状态容器应用持久卷 | ✅ | 20+ |

#### 关键约束
- Powerstore Metro 模式下，DC1 为 Active 端，DC2 为 Passive 端
- Failover 时，DC2 端 LUN 提升为 Active（read-write）
- LUN ID 不变，CSI Driver 通过相同 LUN ID 发现并挂载卷

### 2.2 计算层

#### DC1 — 主集群
- OpenShift 4.x + OpenShift Virtualization
- 承载所有生产工作负载（200+ VM、20+ 容器应用）
- Dell CSI PowerStore Driver 已配置

#### DC2 — 温备集群
- OpenShift 4.x + OpenShift Virtualization（已部署运行）
- Dell CSI PowerStore Driver 已配置
- **预置内容**：
  - Namespace 已创建（与 DC1 一致）
  - 网络配置（NetworkAttachmentDefinition、Service、Route 等）已就位
  - RBAC、Secret、ConfigMap 等已同步
  - VM 定义和容器应用 Deployment 已导入但**未启动**
- **容量规划**：DC2 计算资源 ≥ DC1（CPU、内存需满足 200+ VM 并发运行）

### 2.3 VM 优先级分层

200+ VM 不可能同时启动，必须按业务优先级分批恢复：

| 优先级 | 分层名称 | 典型应用 | VM 数量（估） | 恢复顺序 | 目标完成时间 |
|--------|---------|---------|-------------|---------|------------|
| P1 | 核心基础设施 | AD/DNS/DHCP、数据库主节点、核心网关 | ~15 | 第 1 批 | 0:15-0:22 |
| P2 | 关键业务系统 | ERP、MES、SCADA、质量管理 | ~35 | 第 2 批 | 0:22-0:30 |
| P3 | 重要业务系统 | OA、邮件、文件服务器、监控 | ~50 | 第 3 批 | 0:30-0:38 |
| P4 | 一般业务系统 | 开发/测试环境、报表、归档 | ~100+ | 第 4 批 | 0:38-0:50 |

> 每批内部通过 AAP forks 并行执行（forks=50），单批 200+ VM 中取 50 并发。

### 2.4 网络层

#### IP 地址变更策略
两个 DC 不在同一子网，VM 迁移后需要修改 IP 地址。

**方案：IP 地址映射表 + 自动化修改**

```yaml
# inventory/host_vars/ 结构 — 每个 VM 一个文件，便于 200+ 规模管理
# inventory/host_vars/erp-server.yaml
vm_name: erp-server
namespace: erp-production
dc1_ip: "10.10.1.100"
dc2_ip: "10.20.1.100"
os_type: linux
priority: 2
recovery_group: critical_business
health_check_port: 8080
```

**IP 修改方式**：
- **Linux VM**：通过 `virtctl ssh` 或 `cloud-init` 网络配置注入修改 IP
- **Windows VM**：通过 `virtctl` 执行 PowerShell 脚本修改网卡配置
- **容器应用**：由 OpenShift SDN/OVN-Kubernetes 自动分配 Pod IP，Service/Route 保持不变

#### DNS 管理 — GSLB/GTM
- F5 GTM 或同类 GSLB 解决方案
- Health Check 监控 DC1 应用端点可用性
- DC1 不可用时，自动将 DNS 解析切换至 DC2 的 VIP/Route
- 应用 DNS name 保持不变，用户无感知切换

## 3. Ansible Automation Platform 2.7 架构设计

### 3.1 AAP 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                 Ansible Automation Platform 2.7                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Automation Controller                      │    │
│  │                                                         │    │
│  │  Workflow Template: DR-Failover-Master                  │    │
│  │  ┌───────┐  ┌───────┐  ┌───────┐  ┌──────┐  ┌──────┐  │    │
│  │  │Storage│─►│VM-P1  │─►│VM-P2  │─►│VM-P3 │─►│VM-P4 │  │    │
│  │  │Failovr│  │核心基础│  │关键业务│  │重要   │  │一般   │  │    │
│  │  └───────┘  └───┬───┘  └───┬───┘  └──┬───┘  └──┬───┘  │    │
│  │       │         │          │         │         │       │    │
│  │       │    ┌─────┴──────────┴─────────┴─────────┘       │    │
│  │       │    │  (P1完成后 P2/P3/P4 与容器恢复并行)          │    │
│  │       │    │                                            │    │
│  │       ▼    ▼                                            │    │
│  │  ┌───────────┐  ┌──────────┐  ┌──────────┐             │    │
│  │  │Container  │  │ Network  │  │Validation│             │    │
│  │  │App Recover│─►│ Cutover  │─►│& Report  │             │    │
│  │  └───────────┘  └──────────┘  └──────────┘             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │ Credentials  │  │  Inventories │  │ Execution Environ.  │   │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │                     │   │
│  │ │OpenShift │ │  │ │DC1-VMs   │ │  │ dr-execution-env    │   │
│  │ │DC1 Token │ │  │ │(200+ VM) │ │  │ ├─ python 3.11      │   │
│  │ ├──────────┤ │  │ ├──────────┤ │  │ ├─ kubernetes.core   │   │
│  │ │OpenShift │ │  │ │DC2-VMs   │ │  │ ├─ oc / kubectl     │   │
│  │ │DC2 Token │ │  │ │(映射)    │ │  │ ├─ virtctl           │   │
│  │ ├──────────┤ │  │ ├──────────┤ │  │ └─ Dell PowerStore   │   │
│  │ │Powerstore│ │  │ │Container │ │  │     Python SDK       │   │
│  │ │API Cred  │ │  │ │Apps      │ │  └─────────────────────┘   │
│  │ ├──────────┤ │  │ └──────────┘ │                             │
│  │ │GSLB/GTM │ │  └──────────────┘                             │
│  │ │API Cred  │ │                                               │
│  │ └──────────┘ │                                               │
│  └──────────────┘                                               │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Notification Templates                      │   │
│  │  ┌─────────┐  ┌──────────┐  ┌──────────────────┐        │   │
│  │  │ Email   │  │ Webhook  │  │ Slack/Teams      │        │   │
│  │  │ (报告)   │  │ (监控集成)│  │ (实时进度通知)    │        │   │
│  │  └─────────┘  └──────────┘  └──────────────────┘        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Execution Environment（执行环境）

为 DR Playbook 构建自定义执行环境，包含所有必需工具：

```yaml
# execution-environment/execution-environment.yml
---
version: 3

images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:2.7

dependencies:
  galaxy:
    collections:
      - kubernetes.core
      - community.general
      - ansible.posix
      - dellemc.powerstore
  python:
    - PyPowerStore>=3.0.0
    - openshift-client>=1.0.0
    - jmespath
  system:
    - openshift-clients     # oc CLI
    - kubevirt-virtctl       # virtctl CLI

additional_build_steps:
  append_final:
    - RUN pip3 install --upgrade pip
```

构建并推送 EE 镜像：

```bash
ansible-builder build \
  -t registry.example.com/dr-execution-env:2.7 \
  -f execution-environment/execution-environment.yml

podman push registry.example.com/dr-execution-env:2.7
```

### 3.3 Credential Types（凭据类型）

#### AAP 中配置的凭据

| 凭据名称 | 类型 | 用途 |
|---------|------|------|
| `ocp-dc1-token` | OpenShift/Kubernetes API Token | 访问 DC1 集群 API |
| `ocp-dc2-token` | OpenShift/Kubernetes API Token | 访问 DC2 集群 API |
| `powerstore-api` | Custom Credential Type | Powerstore REST API 认证 |
| `gslb-api` | Custom Credential Type | GSLB/GTM API 认证 |
| `vm-ssh-key` | Machine Credential | SSH 登录 VM 修改 IP |

#### 自定义凭据类型 — Powerstore

```yaml
# AAP -> Credential Types -> 新建
# Input Configuration:
fields:
  - id: powerstore_host
    type: string
    label: Powerstore Management IP
  - id: powerstore_user
    type: string
    label: Username
  - id: powerstore_password
    type: string
    label: Password
    secret: true
required:
  - powerstore_host
  - powerstore_user
  - powerstore_password

# Injector Configuration:
extra_vars:
  powerstore_host: '{{ powerstore_host }}'
  powerstore_user: '{{ powerstore_user }}'
  powerstore_password: '{{ powerstore_password }}'
```

### 3.4 Inventory 设计

200+ VM 规模下，采用**分组 Inventory** 按优先级和 OS 类型组织：

```yaml
# inventory/hosts.yaml
all:
  vars:
    dc2_gateway: "10.20.1.1"
    dc2_netmask: "255.255.255.0"
    dc2_prefix: 24
    dc2_dns:
      - "10.20.1.10"
      - "10.20.1.11"
    dc2_ocp_api: "https://api.dc2.ocp.factory.local:6443"
    dc1_ocp_api: "https://api.dc1.ocp.factory.local:6443"

  children:
    # ── 按优先级分组 ──
    priority_1_infra:
      hosts:
        ad-dc01: {}
        dns-server01: {}
        dhcp-server01: {}
        db-master-01: {}
        core-gateway-01: {}
        # ... ~15 hosts

    priority_2_critical:
      hosts:
        erp-server-01: {}
        erp-server-02: {}
        mes-server-01: {}
        scada-server-01: {}
        quality-mgmt-01: {}
        # ... ~35 hosts

    priority_3_important:
      hosts:
        mail-server-01: {}
        file-server-01: {}
        oa-server-01: {}
        monitoring-01: {}
        # ... ~50 hosts

    priority_4_general:
      hosts:
        dev-env-01: {}
        test-env-01: {}
        report-server-01: {}
        # ... ~100+ hosts

    # ── 按 OS 类型分组（用于 IP 修改策略） ──
    linux_vms:
      children:
        # 引用上面的分组或直接列出
    windows_vms:
      children:
        # Windows VM 列表
```

每个 VM 的 host_vars 文件：

```yaml
# inventory/host_vars/erp-server-01.yaml
vm_name: erp-server-01
namespace: erp-production
dc1_ip: "10.10.1.100"
dc2_ip: "10.20.1.100"
os_type: linux
priority: 2
network_interface: "eth0"
connection_name: "System eth0"
datavolumes:
  - name: erp-server-01-rootdisk
    pvc: erp-server-01-rootdisk
    size: 100Gi
  - name: erp-server-01-data
    pvc: erp-server-01-data
    size: 500Gi
health_check:
  type: tcp
  port: 8080
```

### 3.5 Workflow Template 设计

#### 3.5.1 主 Workflow：DR-Failover-Master

```
┌──────────────────────────────────────────────────────────────────────────┐
│              Workflow Template: DR-Failover-Master                        │
│              Survey: confirm_failover (必须输入 "YES-FAILOVER")           │
│                                                                          │
│  ┌─────────────┐     ┌─────────────┐                                     │
│  │ JT: Preflight│────►│JT: Storage  │                                     │
│  │ Validation   │ OK  │Failover     │                                     │
│  └─────────────┘     └──────┬──────┘                                     │
│                             │ OK                                          │
│                      ┌──────▼──────┐                                     │
│                      │JT: Rescan   │                                     │
│                      │FC & Verify  │                                     │
│                      │Storage      │                                     │
│                      └──────┬──────┘                                     │
│                             │ OK                                          │
│                      ┌──────▼──────┐                                     │
│                      │JT: Recover  │                                     │
│                      │VM-P1        │  forks=15 (全量并发，P1 仅 ~15 台)    │
│                      │核心基础设施   │                                     │
│                      └──────┬──────┘                                     │
│                             │ OK                                          │
│              ┌──────────────┼──────────────┐                             │
│              │              │              │                              │
│       ┌──────▼──────┐┌─────▼──────┐┌──────▼──────┐                      │
│       │JT: Recover  ││JT: Recover ││JT: Recover  │  ◄── 并行执行         │
│       │VM-P2        ││VM-P3       ││Container    │                      │
│       │关键业务      ││重要业务     ││Apps         │                      │
│       │forks=35     ││forks=50    ││             │                      │
│       └──────┬──────┘└─────┬──────┘└──────┬──────┘                      │
│              │             │              │                              │
│              └──────────────┼──────────────┘                             │
│                      ┌──────▼──────┐  ALL OK                             │
│                      │JT: Recover  │                                     │
│                      │VM-P4        │  forks=50                           │
│                      │一般业务      │                                     │
│                      └──────┬──────┘                                     │
│                             │ OK                                          │
│                      ┌──────▼──────┐                                     │
│                      │JT: Network  │                                     │
│                      │Cutover      │                                     │
│                      └──────┬──────┘                                     │
│                             │ OK                                          │
│                      ┌──────▼──────┐                                     │
│                      │JT: Validate │                                     │
│                      │& Report     │                                     │
│                      └─────────────┘                                     │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 3.5.2 Workflow 设计要点

**P1 完成后 P2/P3/容器应用并行的原因**：
- P1 包含 AD/DNS/DHCP 等基础设施 — 后续 VM 和容器可能依赖这些服务
- P2/P3 之间无强依赖关系，可以并行恢复以压缩 RTO
- P4（一般业务）在 P2/P3 都完成后再启动，避免与高优先级 VM 争抢集群资源
- 容器应用与 VM P2/P3 并行，因为容器应用通常依赖 P1 基础设施即可

**Survey 配置（防止误触发）**：
```yaml
# Workflow Template Survey
- variable: confirm_failover
  type: text
  label: "请输入 YES-FAILOVER 确认执行灾难恢复"
  required: true
  validate: "^YES-FAILOVER$"

- variable: target_dc
  type: multiplechoice
  label: "目标恢复数据中心"
  choices:
    - dc2
  default: dc2

- variable: skip_p4
  type: multiplechoice
  label: "是否跳过 P4（一般业务）恢复以加速 RTO"
  choices:
    - "no"
    - "yes"
  default: "no"
```

### 3.6 Job Template 定义

| Job Template 名称 | Playbook | Inventory | Credential | forks | 超时 |
|-------------------|----------|-----------|------------|-------|------|
| DR-Preflight-Validation | `playbooks/preflight.yaml` | DC2-Infrastructure | ocp-dc2-token, powerstore-api | 10 | 300s |
| DR-Storage-Failover | `playbooks/storage-failover.yaml` | DC2-Infrastructure | powerstore-api, ocp-dc2-token | 5 | 600s |
| DR-Rescan-FC | `playbooks/rescan-fc.yaml` | DC2-Workers | ocp-dc2-token | 20 | 300s |
| DR-Recover-VM-P1 | `playbooks/recover-vms.yaml` | DC2-VMs-P1 | ocp-dc2-token, vm-ssh-key | 15 | 600s |
| DR-Recover-VM-P2 | `playbooks/recover-vms.yaml` | DC2-VMs-P2 | ocp-dc2-token, vm-ssh-key | 35 | 600s |
| DR-Recover-VM-P3 | `playbooks/recover-vms.yaml` | DC2-VMs-P3 | ocp-dc2-token, vm-ssh-key | 50 | 600s |
| DR-Recover-VM-P4 | `playbooks/recover-vms.yaml` | DC2-VMs-P4 | ocp-dc2-token, vm-ssh-key | 50 | 900s |
| DR-Recover-Container-Apps | `playbooks/recover-containers.yaml` | DC2-Container-Apps | ocp-dc2-token | 20 | 600s |
| DR-Network-Cutover | `playbooks/network-cutover.yaml` | DC2-Infrastructure | gslb-api, ocp-dc2-token | 5 | 300s |
| DR-Validate-Report | `playbooks/validate-report.yaml` | DC2-All | ocp-dc2-token | 50 | 600s |

### 3.7 Notification Template

```yaml
# AAP Notification Template — 用于各阶段完成通知
# Email + Webhook 双通道

# 阶段成功通知
notification_success:
  type: email
  recipients:
    - dr-team@factory.com
    - it-manager@factory.com
  subject: "[DR] {{ workflow_name }} - {{ job_name }} 完成"
  body: |
    DR 恢复阶段已完成:
    - Workflow: {{ workflow_name }}
    - Job: {{ job_name }}
    - 状态: 成功
    - 耗时: {{ elapsed_time }}s
    - 详情: {{ url }}

# 阶段失败通知（立即告警）
notification_failure:
  type: webhook
  url: "https://hooks.slack.com/services/xxx"
  body: |
    {
      "text": "🚨 DR 恢复失败: {{ job_name }}\nWorkflow: {{ workflow_name }}\n错误详情: {{ url }}"
    }
```

### 3.8 Schedule（定时任务）

在 AAP 中配置以下定时 Schedule，替代 cron：

| Schedule 名称 | 关联 Job Template | 频率 | 用途 |
|--------------|------------------|------|------|
| Config-Export-Schedule | DR-Export-Configs | 每 4 小时 | 定期导出 DC1 VM/App 配置 |
| Config-Sync-Schedule | DR-Sync-To-DC2 | 每 4 小时（Export 后 30 分钟） | 同步配置到 DC2 |
| Preflight-Check-Schedule | DR-Preflight-Validation | 每日 06:00 | 每日检查 DR 就绪状态 |

### 3.9 RBAC 设计

| 角色 | 权限范围 | 人员 |
|------|---------|------|
| DR Admin | 执行 Failover/Failback Workflow、管理 Inventory 和 Credentials | IT 运维负责人 |
| DR Operator | 执行 Failover Workflow（需 Survey 确认）、查看 Job 日志 | 运维值班人员 |
| DR Viewer | 只读查看 Workflow/Job 执行状态和报告 | IT 管理层 |
| Config Admin | 执行配置导出/同步 Job Template、编辑 Inventory | 配置管理员 |

## 4. Day-0：灾备准备

### 4.1 配置同步机制

由于 VM 通过 OpenShift Console 手动创建，不具备 GitOps 版本化管理能力，因此需要建立**定时配置导出与同步机制**，通过 AAP Schedule 自动执行。

#### 4.1.1 VM 配置导出（AAP 定时 Job）

```yaml
# playbooks/export-configs.yaml
---
- name: 导出 DC1 VM 和应用配置
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    export_base_dir: "/data/dr-configs"
    ocp_api: "{{ dc1_ocp_api }}"

  tasks:
    - name: 获取所有受保护的 Namespace
      kubernetes.core.k8s_info:
        api_version: v1
        kind: Namespace
        label_selectors:
          - dr-protected=true
      register: dr_namespaces

    - name: 导出每个 Namespace 的 VM 定义
      kubernetes.core.k8s_info:
        api_version: kubevirt.io/v1
        kind: VirtualMachine
        namespace: "{{ item.metadata.name }}"
      loop: "{{ dr_namespaces.resources }}"
      loop_control:
        label: "{{ item.metadata.name }}"
      register: vm_exports

    - name: 保存 VM 定义到文件
      copy:
        content: "{{ item.resources | to_nice_yaml }}"
        dest: "{{ export_base_dir }}/vms/{{ item.item.metadata.name }}/vms.yaml"
      loop: "{{ vm_exports.results }}"
      loop_control:
        label: "{{ item.item.metadata.name }}"

    - name: 导出 DataVolume 定义
      kubernetes.core.k8s_info:
        api_version: cdi.kubevirt.io/v1beta1
        kind: DataVolume
        namespace: "{{ item.metadata.name }}"
      loop: "{{ dr_namespaces.resources }}"
      loop_control:
        label: "{{ item.metadata.name }}"
      register: dv_exports

    - name: 保存 DataVolume 定义
      copy:
        content: "{{ item.resources | to_nice_yaml }}"
        dest: "{{ export_base_dir }}/vms/{{ item.item.metadata.name }}/datavolumes.yaml"
      loop: "{{ dv_exports.results }}"
      loop_control:
        label: "{{ item.item.metadata.name }}"

    - name: 导出 PVC 定义
      kubernetes.core.k8s_info:
        api_version: v1
        kind: PersistentVolumeClaim
        namespace: "{{ item.metadata.name }}"
      loop: "{{ dr_namespaces.resources }}"
      loop_control:
        label: "{{ item.metadata.name }}"
      register: pvc_exports

    - name: 保存 PVC 定义
      copy:
        content: "{{ item.resources | to_nice_yaml }}"
        dest: "{{ export_base_dir }}/vms/{{ item.item.metadata.name }}/pvcs.yaml"
      loop: "{{ pvc_exports.results }}"
      loop_control:
        label: "{{ item.item.metadata.name }}"

    - name: 导出关联资源 (Service, Route, ConfigMap, Secret, NAD)
      kubernetes.core.k8s_info:
        api_version: "{{ resource_item.api }}"
        kind: "{{ resource_item.kind }}"
        namespace: "{{ ns_item.metadata.name }}"
      loop: "{{ dr_namespaces.resources | product(export_resources) | list }}"
      loop_control:
        label: "{{ ns_item.metadata.name }}/{{ resource_item.kind }}"
      vars:
        ns_item: "{{ item.0 }}"
        resource_item: "{{ item.1 }}"
        export_resources:
          - { api: v1, kind: Service }
          - { api: route.openshift.io/v1, kind: Route }
          - { api: v1, kind: ConfigMap }
          - { api: v1, kind: Secret }
          - { api: k8s.cni.cncf.io/v1, kind: NetworkAttachmentDefinition }
      register: resource_exports

    - name: 保存关联资源定义
      copy:
        content: "{{ item.resources | to_nice_yaml }}"
        dest: "{{ export_base_dir }}/apps/{{ item.item.0.metadata.name }}/{{ item.item.1.kind | lower }}s.yaml"
      loop: "{{ resource_exports.results }}"
      when: item.resources | length > 0

    - name: 生成导出摘要
      debug:
        msg: >-
          配置导出完成:
          Namespaces={{ dr_namespaces.resources | length }},
          VMs={{ vm_exports.results | map(attribute='resources') | flatten | length }},
          DVs={{ dv_exports.results | map(attribute='resources') | flatten | length }},
          PVCs={{ pvc_exports.results | map(attribute='resources') | flatten | length }}
```

**AAP Schedule**：每 4 小时执行，关联 Job Template `DR-Export-Configs`

#### 4.1.2 DC2 预置同步

```yaml
# playbooks/sync-to-dc2.yaml
---
- name: 同步配置到 DC2 集群（适配转换）
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    config_dir: "/data/dr-configs"

  tasks:
    - name: 确保 DC2 Namespace 存在
      kubernetes.core.k8s:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ item.metadata.name }}"
            labels: "{{ item.metadata.labels }}"
      loop: "{{ dr_namespaces }}"

    - name: 读取 DC1 VM 定义
      include_vars:
        file: "{{ config_dir }}/vms/{{ item }}/vms.yaml"
        name: "vm_defs_{{ item | replace('-', '_') }}"
      loop: "{{ dr_namespace_names }}"

    - name: 适配 VM 定义 — 清理元数据、设置停止状态
      set_fact:
        adapted_vms: >-
          {{ adapted_vms | default([]) + [item | combine({
            'metadata': item.metadata | dict2items
              | rejectattr('key', 'in', ['resourceVersion', 'uid', 'creationTimestamp', 'generation', 'managedFields'])
              | items2dict,
            'spec': item.spec | combine({'running': false})
          })] }}
      loop: "{{ all_vm_defs }}"

    - name: 应用适配后的 VM 定义到 DC2（stopped 状态）
      kubernetes.core.k8s:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        state: present
        definition: "{{ item }}"
      loop: "{{ adapted_vms }}"
      throttle: 20
```

### 4.2 IP 映射表维护

200+ VM 的 IP 映射通过 AAP Inventory host_vars 管理。建议使用 **AAP 项目**关联的 Git 仓库存放 Inventory，配置变更通过 Git 提交审计追踪。

```
inventory/
├── hosts.yaml              # 分组定义
└── host_vars/
    ├── ad-dc01.yaml         # P1: 15 files
    ├── dns-server01.yaml
    ├── ...
    ├── erp-server-01.yaml   # P2: 35 files
    ├── erp-server-02.yaml
    ├── ...
    ├── mail-server-01.yaml  # P3: 50 files
    ├── ...
    ├── dev-env-01.yaml      # P4: 100+ files
    ├── ...
    └── (共 200+ 文件)
```

### 4.3 日常健康检查

通过 AAP Schedule 每日自动验证 DR 就绪状态：

```yaml
# playbooks/preflight.yaml
---
- name: DR 就绪状态预检
  hosts: localhost
  connection: local
  gather_facts: no

  tasks:
    - name: 检查 DC2 集群可达性
      uri:
        url: "{{ dc2_ocp_api }}/healthz"
        validate_certs: no
        status_code: 200

    - name: 检查 Powerstore Metro 复制状态
      uri:
        url: "https://{{ powerstore_dc2_mgmt }}/api/rest/replication_session"
        method: GET
        headers:
          Authorization: "Bearer {{ powerstore_token }}"
        validate_certs: no
      register: metro_status

    - name: 验证 Metro 复制状态正常
      assert:
        that:
          - item.state == "Synchronized"
        fail_msg: "Metro session {{ item.id }} 状态异常: {{ item.state }}"
      loop: "{{ metro_status.json }}"

    - name: 检查 DC2 CSI Driver 状态
      kubernetes.core.k8s_info:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        api_version: storage.k8s.io/v1
        kind: CSIDriver
        name: csi-powerstore.dellemc.com
      register: csi_status
      failed_when: csi_status.resources | length == 0

    - name: 检查 DC2 VM 定义数量与 DC1 一致
      block:
        - kubernetes.core.k8s_info:
            host: "{{ dc1_ocp_api }}"
            api_key: "{{ dc1_ocp_token }}"
            api_version: kubevirt.io/v1
            kind: VirtualMachine
            namespace: "{{ item }}"
          loop: "{{ dr_namespace_names }}"
          register: dc1_vm_count

        - kubernetes.core.k8s_info:
            host: "{{ dc2_ocp_api }}"
            api_key: "{{ dc2_ocp_token }}"
            api_version: kubevirt.io/v1
            kind: VirtualMachine
            namespace: "{{ item }}"
          loop: "{{ dr_namespace_names }}"
          register: dc2_vm_count

        - assert:
            that:
              - dc1_total == dc2_total
            fail_msg: "VM 数量不匹配: DC1={{ dc1_total }}, DC2={{ dc2_total }}"
          vars:
            dc1_total: "{{ dc1_vm_count.results | map(attribute='resources') | flatten | length }}"
            dc2_total: "{{ dc2_vm_count.results | map(attribute='resources') | flatten | length }}"

    - name: 检查 DC2 节点资源余量
      shell: |
        oc get nodes --selector=node-role.kubernetes.io/worker \
          -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.cpu}{"\t"}{.status.allocatable.memory}{"\n"}{end}'
      register: dc2_capacity

    - name: 生成预检报告
      debug:
        msg: |
          === DR 预检报告 ===
          DC2 集群: ✅ 可达
          Metro 复制: ✅ Synchronized
          CSI Driver: ✅ 已安装
          VM 定义同步: ✅ 数量一致
          DC2 节点资源:
          {{ dc2_capacity.stdout }}
```

## 5. Day-1：灾难恢复执行流程

### 5.1 整体时间线（200+ VM 分批恢复）

```
0:00     0:05       0:10        0:15     0:22        0:30     0:38     0:50    1:00
  │        │          │           │        │           │        │        │       │
  ▼        ▼          ▼           ▼        ▼           ▼        ▼        ▼       ▼
故障确认  AAP       存储切换     FC扫描    P1完成      P2完成   P3完成   P4完成  验收
         Survey     完成        PV验证    (~15台)     (~35台)  (~50台)  (100+)  完成
         确认                             │           │        │        │
                                          │   ┌───────┘        │        │
                                          │   │  ┌─────────────┘        │
                                          ▼   ▼  ▼                      │
                                    ┌─── P2/P3/容器并行恢复 ──┐          │
                                    └─────────────────────────┘          │
                                                                        │
Phase 1   Phase 2     Phase 3       Phase 4                   Phase 5  Phase 6
决策      存储切换     FC刷新        VM + 容器分批恢复          网络切换  验收
```

### 5.2 Phase 1：故障检测与决策（0-5 分钟）

**触发条件**：
- 监控系统报警：DC1 多个核心服务不可达
- GSLB Health Check 连续失败
- 存储团队确认 Powerstore DC1 端不可用

**通过 AAP 执行**：
1. 运维人员登录 AAP Web Console
2. 进入 Workflow Template `DR-Failover-Master`
3. 点击 Launch，填写 Survey：输入 `YES-FAILOVER` 确认
4. Workflow 自动按阶段编排执行

### 5.3 Phase 2：存储 Failover（5-10 分钟）

```yaml
# playbooks/storage-failover.yaml
---
- name: Powerstore Metro Failover
  hosts: localhost
  connection: local
  gather_facts: no

  tasks:
    - name: 获取当前 Metro Session 状态
      uri:
        url: "https://{{ powerstore_host }}/api/rest/replication_session"
        method: GET
        user: "{{ powerstore_user }}"
        password: "{{ powerstore_password }}"
        validate_certs: no
        force_basic_auth: yes
      register: metro_sessions

    - name: 触发每个 Metro Session Failover
      uri:
        url: "https://{{ powerstore_host }}/api/rest/replication_session/{{ item.id }}/failover"
        method: POST
        user: "{{ powerstore_user }}"
        password: "{{ powerstore_password }}"
        body_format: json
        body:
          is_planned: false
        validate_certs: no
        force_basic_auth: yes
        status_code: [200, 204]
      loop: "{{ metro_sessions.json }}"
      loop_control:
        label: "session={{ item.id }} volume={{ item.local_resource_name | default('N/A') }}"
      register: failover_results

    - name: 等待 Failover 完成
      uri:
        url: "https://{{ powerstore_host }}/api/rest/replication_session/{{ item.id }}"
        method: GET
        user: "{{ powerstore_user }}"
        password: "{{ powerstore_password }}"
        validate_certs: no
        force_basic_auth: yes
      loop: "{{ metro_sessions.json }}"
      register: session_status
      until: session_status.json.state in ['Failed_Over', 'OK']
      retries: 30
      delay: 10

    - name: 验证所有 LUN 可写
      uri:
        url: "https://{{ powerstore_host }}/api/rest/volume"
        method: GET
        user: "{{ powerstore_user }}"
        password: "{{ powerstore_password }}"
        validate_certs: no
        force_basic_auth: yes
      register: volumes

    - name: 断言所有卷就绪
      assert:
        that:
          - item.state == "Ready"
        fail_msg: "LUN {{ item.name }} (id={{ item.id }}) 状态: {{ item.state }}"
      loop: "{{ volumes.json }}"
      loop_control:
        label: "{{ item.name }}"
```

### 5.4 Phase 3：FC 扫描与存储验证（10-15 分钟）

```yaml
# playbooks/rescan-fc.yaml
---
- name: 刷新 DC2 Worker 节点的 FC 多路径并验证 PV
  hosts: localhost
  connection: local
  gather_facts: no

  tasks:
    - name: 获取 DC2 Worker 节点列表
      kubernetes.core.k8s_info:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        api_version: v1
        kind: Node
        label_selectors:
          - node-role.kubernetes.io/worker
      register: worker_nodes

    - name: 在每个 Worker 节点执行 FC rescan
      shell: |
        oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
          debug node/{{ item.metadata.name }} -- \
          chroot /host bash -c "rescan-scsi-bus.sh -r && multipath -r"
      loop: "{{ worker_nodes.resources }}"
      loop_control:
        label: "{{ item.metadata.name }}"
      async: 120
      poll: 0
      register: rescan_jobs

    - name: 等待所有 FC rescan 完成
      async_status:
        jid: "{{ item.ansible_job_id }}"
      loop: "{{ rescan_jobs.results }}"
      loop_control:
        label: "{{ item.item.metadata.name }}"
      register: rescan_results
      until: rescan_results.finished
      retries: 12
      delay: 10

    - name: 验证 CSI Driver Pod 全部就绪
      shell: |
        oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
          get pod -n csi-powerstore --no-headers \
          | grep -v Running | wc -l
      register: csi_not_ready
      failed_when: csi_not_ready.stdout | int > 0
```

### 5.5 Phase 4：VM 与容器应用恢复（15-50 分钟）

#### 5.5.1 通用 VM 恢复 Playbook（所有优先级复用）

```yaml
# playbooks/recover-vms.yaml
---
- name: 恢复虚拟机（按 Inventory 分组执行）
  hosts: all
  connection: local
  gather_facts: no
  serial: "100%"

  tasks:
    # ── Step 1: 创建 PV（静态绑定到 Metro 复制的 LUN） ──
    - name: 创建 PV 对象
      kubernetes.core.k8s:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        state: present
        definition:
          apiVersion: v1
          kind: PersistentVolume
          metadata:
            name: "{{ item.name }}-pv"
            annotations:
              pv.kubernetes.io/provisioned-by: csi-powerstore.dellemc.com
          spec:
            capacity:
              storage: "{{ item.size }}"
            accessModes: ["ReadWriteOnce"]
            persistentVolumeReclaimPolicy: Retain
            storageClassName: "{{ dc2_storage_class }}"
            csi:
              driver: csi-powerstore.dellemc.com
              volumeHandle: "{{ item.volume_handle }}"
              fsType: "{{ item.fs_type | default('ext4') }}"
            volumeMode: "{{ item.volume_mode | default('Block') }}"
      loop: "{{ datavolumes }}"
      loop_control:
        label: "{{ item.name }}"

    # ── Step 2: 创建 PVC 并绑定 PV ──
    - name: 创建 PVC 并绑定
      kubernetes.core.k8s:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        state: present
        definition:
          apiVersion: v1
          kind: PersistentVolumeClaim
          metadata:
            name: "{{ item.pvc }}"
            namespace: "{{ namespace }}"
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: "{{ item.size }}"
            volumeName: "{{ item.name }}-pv"
            storageClassName: "{{ dc2_storage_class }}"
            volumeMode: "{{ item.volume_mode | default('Block') }}"
      loop: "{{ datavolumes }}"
      loop_control:
        label: "{{ item.pvc }}"

    # ── Step 3: 注入 CDI 预填充注解 ──
    - name: 注入 DataVolume prePopulated 注解
      kubernetes.core.k8s:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        state: patched
        api_version: cdi.kubevirt.io/v1beta1
        kind: DataVolume
        name: "{{ item.name }}"
        namespace: "{{ namespace }}"
        definition:
          metadata:
            annotations:
              cdi.kubevirt.io/storage.prePopulated: "{{ item.name }}"
      loop: "{{ datavolumes }}"
      loop_control:
        label: "{{ item.name }}"
      ignore_errors: yes

    - name: 注入 PVC populatedFor 注解
      kubernetes.core.k8s:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        state: patched
        api_version: v1
        kind: PersistentVolumeClaim
        name: "{{ item.pvc }}"
        namespace: "{{ namespace }}"
        definition:
          metadata:
            annotations:
              cdi.kubevirt.io/storage.populatedFor: "{{ item.name }}"
      loop: "{{ datavolumes }}"
      loop_control:
        label: "{{ item.pvc }}"
      ignore_errors: yes

    # ── Step 4: 启动 VM ──
    - name: 启动 VirtualMachine
      kubernetes.core.k8s:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        state: patched
        api_version: kubevirt.io/v1
        kind: VirtualMachine
        name: "{{ vm_name }}"
        namespace: "{{ namespace }}"
        definition:
          spec:
            running: true

    # ── Step 5: 等待 VM 就绪 ──
    - name: 等待 VMI 进入 Running 状态
      kubernetes.core.k8s_info:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        api_version: kubevirt.io/v1
        kind: VirtualMachineInstance
        name: "{{ vm_name }}"
        namespace: "{{ namespace }}"
      register: vmi_status
      until:
        - vmi_status.resources | length > 0
        - vmi_status.resources[0].status.phase == "Running"
      retries: 30
      delay: 10

    # ── Step 6: 修改 IP 地址 ──
    - name: 等待 VM SSH 可达
      wait_for:
        host: "{{ dc1_ip }}"
        port: 22
        timeout: 120
      ignore_errors: yes

    - name: 修改 Linux VM IP 地址
      shell: |
        virtctl ssh {{ vm_name }} -n {{ namespace }} \
          --known-hosts=/dev/null \
          -c "sudo nmcli con mod '{{ connection_name }}' \
            ipv4.addresses {{ dc2_ip }}/{{ dc2_prefix }} \
            ipv4.gateway {{ dc2_gateway }} \
            ipv4.dns '{{ dc2_dns | join(",") }}' \
            ipv4.method manual && \
          sudo nmcli con up '{{ connection_name }}'"
      when: os_type == "linux"

    - name: 修改 Windows VM IP 地址
      shell: |
        virtctl ssh {{ vm_name }} -n {{ namespace }} \
          --known-hosts=/dev/null \
          -c "powershell -Command \"
            \$adapter = Get-NetAdapter | Where-Object {\\$_.Status -eq 'Up'} | Select-Object -First 1;
            New-NetIPAddress -InterfaceIndex \\$adapter.InterfaceIndex \
              -IPAddress {{ dc2_ip }} -PrefixLength {{ dc2_prefix }} \
              -DefaultGateway {{ dc2_gateway }};
            Set-DnsClientServerAddress -InterfaceIndex \\$adapter.InterfaceIndex \
              -ServerAddresses {{ dc2_dns | join(',') }}
          \""
      when: os_type == "windows"

    # ── Step 7: 验证 VM 健康 ──
    - name: 验证 VM 网络可达（新 IP）
      wait_for:
        host: "{{ dc2_ip }}"
        port: "{{ health_check.port | default(22) }}"
        timeout: 60
      ignore_errors: yes
      register: vm_health

    - name: 记录恢复结果
      set_fact:
        recovery_result:
          vm_name: "{{ vm_name }}"
          namespace: "{{ namespace }}"
          status: "{{ 'success' if vm_health is not failed else 'failed' }}"
          dc2_ip: "{{ dc2_ip }}"
```

> **关键设计**：此 Playbook 以 `hosts: all` 运行，由 AAP Job Template 的 **Inventory Limit** 和 **forks** 参数控制并发度。
> 不同优先级的 Job Template 使用不同的 Inventory 分组（`priority_1_infra`、`priority_2_critical` 等），
> 同一 Playbook 无需修改即可复用。

#### 5.5.2 并发控制策略（200+ VM 的关键）

| 优先级 | VM 数量 | AAP forks | 并发策略 | 预估耗时 |
|--------|---------|-----------|---------|---------|
| P1 | ~15 | 15 | 全量并发（全部同时启动） | ~7 min |
| P2 | ~35 | 35 | 全量并发 | ~8 min |
| P3 | ~50 | 50 | 全量并发 | ~8 min |
| P4 | ~100+ | 50 | 分两轮（50 + 50+），滚动执行 | ~12 min |

> AAP 2.7 中 forks 控制的是**同时处理的 host 数量**。对于 P4 的 100+ VM，
> forks=50 表示同时处理 50 台 VM，前 50 台中有完成的会立即让出 slot 给下一台（滚动执行，非严格分批）。

#### 5.5.3 容器应用恢复（与 VM P2/P3 并行）

```yaml
# playbooks/recover-containers.yaml
---
- name: 恢复有状态容器应用
  hosts: localhost
  connection: local
  gather_facts: no

  tasks:
    - name: 获取所有需恢复的容器应用 Namespace
      kubernetes.core.k8s_info:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        api_version: v1
        kind: Namespace
        label_selectors:
          - dr-protected=true
          - app-type=container
      register: app_namespaces

    - name: 为每个 Namespace 创建 PV/PVC 绑定
      include_tasks: tasks/rebind-container-storage.yaml
      loop: "{{ app_namespaces.resources }}"
      loop_control:
        loop_var: app_ns
        label: "{{ app_ns.metadata.name }}"

    - name: Scale up Deployments
      shell: |
        oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
          scale deployment --all --replicas=1 -n {{ item.metadata.name }}
      loop: "{{ app_namespaces.resources }}"
      loop_control:
        label: "{{ item.metadata.name }}"

    - name: Scale up StatefulSets
      shell: |
        for ss in $(oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
          get statefulset -n {{ item.metadata.name }} -o jsonpath='{.items[*].metadata.name}'); do
          oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
            scale statefulset "$ss" --replicas=1 -n {{ item.metadata.name }}
        done
      loop: "{{ app_namespaces.resources }}"
      loop_control:
        label: "{{ item.metadata.name }}"

    - name: 等待 Deployment 就绪
      shell: |
        oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
          wait --for=condition=Available deployment --all \
          -n {{ item.metadata.name }} --timeout=300s
      loop: "{{ app_namespaces.resources }}"
      loop_control:
        label: "{{ item.metadata.name }}"
      ignore_errors: yes
      register: deploy_status

    - name: 等待 StatefulSet 就绪
      shell: |
        oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
          rollout status statefulset --all \
          -n {{ item.metadata.name }} --timeout=300s
      loop: "{{ app_namespaces.resources }}"
      loop_control:
        label: "{{ item.metadata.name }}"
      ignore_errors: yes
```

### 5.6 Phase 5：DNS/网络切换

```yaml
# playbooks/network-cutover.yaml
---
- name: GSLB/GTM DNS 切换
  hosts: localhost
  connection: local
  gather_facts: no

  tasks:
    - name: 检查 GSLB DC1 Pool 状态
      uri:
        url: "https://{{ gslb_host }}/mgmt/tm/gtm/pool/a/{{ gslb_pool_dc1 }}/members/stats"
        method: GET
        headers:
          Authorization: "Basic {{ gslb_auth | b64encode }}"
        validate_certs: no
      register: gslb_dc1_status
      ignore_errors: yes

    - name: 禁用 GSLB DC1 Pool（强制切换到 DC2）
      uri:
        url: "https://{{ gslb_host }}/mgmt/tm/gtm/pool/a/{{ gslb_pool_dc1 }}"
        method: PATCH
        headers:
          Authorization: "Basic {{ gslb_auth | b64encode }}"
          Content-Type: application/json
        body_format: json
        body:
          enabled: false
        validate_certs: no

    - name: 启用 GSLB DC2 Pool
      uri:
        url: "https://{{ gslb_host }}/mgmt/tm/gtm/pool/a/{{ gslb_pool_dc2 }}"
        method: PATCH
        headers:
          Authorization: "Basic {{ gslb_auth | b64encode }}"
          Content-Type: application/json
        body_format: json
        body:
          enabled: true
        validate_certs: no

    - name: 验证 DNS 解析（等待 TTL 过期）
      shell: |
        dig +short {{ item.dns_name }} @{{ dc2_dns[0] }}
      loop: "{{ critical_dns_records }}"
      register: dns_verify
      retries: 12
      delay: 10
      until: "item.dc2_vip in dns_verify.stdout"
```

### 5.7 Phase 6：验收确认与报告

```yaml
# playbooks/validate-report.yaml
---
- name: DR 恢复验收与报告生成
  hosts: all
  connection: local
  gather_facts: no

  tasks:
    - name: 检查 VMI 状态
      kubernetes.core.k8s_info:
        host: "{{ dc2_ocp_api }}"
        api_key: "{{ dc2_ocp_token }}"
        api_version: kubevirt.io/v1
        kind: VirtualMachineInstance
        name: "{{ vm_name }}"
        namespace: "{{ namespace }}"
      register: vmi_check

    - name: 记录 VM 状态
      set_stats:
        data:
          "vm_{{ vm_name }}_status": "{{ vmi_check.resources[0].status.phase | default('NotFound') }}"
          "vm_{{ vm_name }}_ip": "{{ dc2_ip }}"
        per_host: no

- name: 汇总生成报告
  hosts: localhost
  connection: local
  gather_facts: yes

  tasks:
    - name: 收集所有 VMI 状态
      shell: |
        oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
          get vmi -A -o json
      register: all_vmi

    - name: 统计恢复结果
      set_fact:
        total_vms: "{{ all_vmi.stdout | from_json | json_query('items[*]') | length }}"
        running_vms: "{{ all_vmi.stdout | from_json | json_query('items[?status.phase==`Running`]') | length }}"
        failed_vms: "{{ all_vmi.stdout | from_json | json_query('items[?status.phase!=`Running`]') | list }}"

    - name: 收集容器应用状态
      shell: |
        oc --server={{ dc2_ocp_api }} --token={{ dc2_ocp_token }} \
          get pod -A -l dr-protected=true --no-headers \
          | awk '{print $4}' | sort | uniq -c
      register: pod_summary

    - name: 生成 DR 恢复报告
      template:
        src: templates/dr-report.html.j2
        dest: "/tmp/dr-report-{{ ansible_date_time.iso8601_basic_short }}.html"
      vars:
        report_data:
          timestamp: "{{ ansible_date_time.iso8601 }}"
          total_vms: "{{ total_vms }}"
          running_vms: "{{ running_vms }}"
          failed_vms: "{{ failed_vms }}"
          pod_summary: "{{ pod_summary.stdout }}"
          success_rate: "{{ (running_vms | int / total_vms | int * 100) | round(1) }}%"

    - name: 输出恢复摘要
      debug:
        msg: |
          ╔══════════════════════════════════════╗
          ║        DR 恢复报告                    ║
          ╠══════════════════════════════════════╣
          ║ 总 VM 数量:    {{ total_vms }}
          ║ 运行中:        {{ running_vms }}
          ║ 失败:          {{ total_vms | int - running_vms | int }}
          ║ 成功率:        {{ (running_vms | int / total_vms | int * 100) | round(1) }}%
          ║ 容器应用状态:
          {{ pod_summary.stdout | indent(10) }}
          ╚══════════════════════════════════════╝
```

> **注意**：验收阶段的 `set_stats` 模块可将每台 VM 的状态数据回传到 AAP Workflow，
> 在 Automation Controller 的 Workflow Visualizer 中直接查看整体恢复结果。

## 6. Day-2：Failback 回切流程

当 DC1 恢复正常后，需要将业务切回 DC1。

### 6.1 Failback 前提条件
- DC1 基础设施完全恢复（网络、计算、存储）
- DC1 OpenShift 集群正常运行
- DC1 Powerstore 已重新上线

### 6.2 Failback Workflow Template

在 AAP 中创建对称的 `DR-Failback-Master` Workflow Template，方向反转：

```
┌──────────────────────────────────────────────────────────────────┐
│              Workflow Template: DR-Failback-Master                │
│              Survey: confirm_failback (输入 "YES-FAILBACK")      │
│                                                                  │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐    │
│  │ JT: Stop   │─►│JT: Resync    │─►│JT: Wait-For-Sync      │    │
│  │ DC2 VMs    │  │Metro DC2→DC1 │  │(等待数据完全同步)       │    │
│  └────────────┘  └──────────────┘  └──────────┬─────────────┘    │
│                                               │ OK                │
│                                        ┌──────▼──────┐           │
│                                        │JT: Storage  │           │
│                                        │Failback     │           │
│                                        │(DC1 Active) │           │
│                                        └──────┬──────┘           │
│                                               │ OK                │
│                           ┌───────────────────┼────────────┐     │
│                    ┌──────▼──────┐ ┌──────────▼──┐ ┌───────▼──┐  │
│                    │JT: Recover  │ │JT: Recover  │ │JT: Recov │  │
│                    │DC1 VMs     │ │DC1 VMs      │ │Container │  │
│                    │(P1→P4 分批) │ │(其余)        │ │Apps DC1  │  │
│                    └──────┬──────┘ └──────┬──────┘ └────┬─────┘  │
│                           └───────────────┼─────────────┘        │
│                                    ┌──────▼──────┐               │
│                                    │JT: Network  │               │
│                                    │Cutover→DC1  │               │
│                                    └──────┬──────┘               │
│                                           │ OK                    │
│                                    ┌──────▼──────┐               │
│                                    │JT: Validate │               │
│                                    │& Restore    │               │
│                                    │Metro DC1→DC2│               │
│                                    └─────────────┘               │
└──────────────────────────────────────────────────────────────────┘
```

### 6.3 Failback 关键步骤

```
1. 有序停止 DC2 工作负载（反序：P4→P3→P2→P1）
2. 重建 Powerstore Metro 复制（DC2 → DC1 方向）
3. 等待初始同步完成（数据从 DC2 完整同步到 DC1）
4. 计划停机窗口 — 停止 DC2 最终写入
5. 执行存储 Failback（DC1 LUN 提升为 Active）
6. 在 DC1 恢复 VM 和容器应用（P1→P4 分批启动）
7. 修改 VM IP 回 DC1 地址
8. GSLB/GTM 切换回 DC1
9. 验收确认
10. 恢复正常 Metro 复制方向（DC1 → DC2）
```

## 7. AAP 项目结构

```
K8SDRwithStorageDR/
├── ansible.cfg
├── execution-environment/
│   └── execution-environment.yml          # 自定义 EE 定义
├── inventory/
│   ├── hosts.yaml                         # 主 Inventory（分优先级组）
│   ├── group_vars/
│   │   ├── all.yaml                       # 全局变量（网关、DNS、API 地址等）
│   │   ├── priority_1_infra.yaml          # P1 组特有变量
│   │   ├── priority_2_critical.yaml       # P2 组特有变量
│   │   ├── priority_3_important.yaml      # P3 组特有变量
│   │   ├── priority_4_general.yaml        # P4 组特有变量
│   │   ├── linux_vms.yaml                 # Linux VM 公共变量
│   │   └── windows_vms.yaml               # Windows VM 公共变量
│   └── host_vars/
│       ├── ad-dc01.yaml                   # 每台 VM 的变量（200+ 文件）
│       ├── erp-server-01.yaml
│       ├── ...
│       └── (200+ files)
├── playbooks/
│   ├── preflight.yaml                     # 预检验证
│   ├── storage-failover.yaml              # 存储 Failover
│   ├── rescan-fc.yaml                     # FC 扫描与验证
│   ├── recover-vms.yaml                   # VM 恢复（通用，所有优先级复用）
│   ├── recover-containers.yaml            # 容器应用恢复
│   ├── network-cutover.yaml               # 网络/DNS 切换
│   ├── validate-report.yaml               # 验收报告
│   ├── export-configs.yaml                # 定期配置导出
│   ├── sync-to-dc2.yaml                   # 配置同步到 DC2
│   ├── stop-workloads.yaml                # 停止工作负载（Failback 用）
│   └── tasks/
│       └── rebind-container-storage.yaml   # 容器存储重绑定子任务
├── roles/
│   └── (可选 — 当前方案以 Playbook 为主,
│         复杂逻辑可重构为 role)
├── templates/
│   └── dr-report.html.j2                  # 恢复报告模板
├── collections/
│   └── requirements.yml                   # Ansible Collection 依赖
├── docs/
│   ├── aap-setup-guide.md                 # AAP 配置指南
│   └── runbook.md                         # 操作手册
├── CLAUDE.md
├── DR-Design.md                           # 本设计文档
└── Requirement.md                         # 需求文档
```

```yaml
# collections/requirements.yml
---
collections:
  - name: kubernetes.core
    version: ">=3.0.0"
  - name: community.general
    version: ">=8.0.0"
  - name: ansible.posix
    version: ">=1.5.0"
  - name: dellemc.powerstore
    version: ">=3.0.0"
```

## 8. 关键风险与缓解措施

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| VM 配置与 DC1 不同步 | 恢复后 VM 配置过时 | AAP Schedule 每 4 小时自动导出同步；关键变更后手动触发 |
| 200+ VM 同时启动资源不足 | 部分 VM 调度失败 | 按优先级分 4 批恢复；P1 先启动后再并行 P2/P3；DC2 预留充足资源 |
| PV/PVC 绑定失败 | VM 或容器无法访问数据 | Day-0 预检验证 LUN 可见性；playbook 含详细错误处理 |
| 批量 IP 修改失败 | VM 启动但网络不通 | 每台 VM 的 IP 映射在 host_vars 中管理；支持单台 VM 重试 |
| GSLB 切换延迟 | 用户在切换期间无法访问 | DNS TTL 设为 60s；playbook 含强制切换步骤 |
| CDI 覆盖恢复数据 | 数据丢失 | 在 PVC 绑定后立即注入 prePopulated 注解 |
| AAP 控制节点不可用 | 无法触发 DR | AAP 部署在独立于 DC1/DC2 的第三方或 DC2 侧；配置 HA |
| forks 过高导致 OCP API 限流 | VM 创建/启动失败 | forks 上限设为 50，结合 throttle 控制 API 请求速率 |
| Failback 数据不一致 | 数据冲突 | Failback 前确保 Metro 完全同步；设置停机窗口 |

## 9. AAP 部署检查清单

### 9.1 AAP 平台配置

- [ ] AAP 2.7 已部署并可访问（建议部署在 DC2 侧或独立基础设施）
- [ ] 自定义 Execution Environment 已构建并推送到镜像仓库
- [ ] EE 已在 AAP 中注册并设为 DR 项目默认 EE
- [ ] Credential Types 已创建（Powerstore、GSLB）
- [ ] Credentials 已配置（OCP DC1/DC2 Token、Powerstore API、GSLB API、VM SSH Key）
- [ ] Project 已关联 Git 仓库，SCM 更新策略已设置
- [ ] Inventory 已创建，200+ VM host_vars 已维护完整
- [ ] 所有 Job Templates 已创建并关联正确的 Playbook/Inventory/Credential/EE
- [ ] Workflow Templates（Failover + Failback）已创建并配置 Survey
- [ ] Notification Templates 已配置（Email + Webhook/Slack）
- [ ] Schedule 已配置（配置导出、每日预检）
- [ ] RBAC 角色已分配

### 9.2 基础设施就绪

- [ ] DC2 OpenShift 集群已部署并正常运行
- [ ] DC2 Dell CSI PowerStore Driver 已安装并配置
- [ ] Powerstore Metro 复制已建立且状态 Synchronized
- [ ] DC2 所有 worker 节点 FC HBA 已连接并识别 LUN
- [ ] DC2 计算资源 ≥ DC1（可支撑 200+ VM 并发运行）
- [ ] DC2 Namespace 和基础配置已创建
- [ ] GSLB/GTM 已配置 DC1/DC2 池和健康检查（TTL=60s）
- [ ] VM 配置定期导出任务已正常运行
- [ ] 日常预检 Job 已连续通过至少 7 天

### 9.3 验证与演练

- [ ] 单台 VM 恢复测试通过
- [ ] 单批次（P1 ~15 台）恢复测试通过
- [ ] 全流程 DR 演练已至少执行一次并验证通过
- [ ] Failback 回切演练已验证
- [ ] RTO 实测 ≤ 1 小时
