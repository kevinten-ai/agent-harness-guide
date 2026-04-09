# 阿里云 CLI 工具生态全面调研报告

> 调研目标：全面了解阿里云 CLI 工具现状，为构建 Agent 工具层提供参考
> 调研日期：2026-04-09

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [CLI 工具分类矩阵](#2-cli-工具分类矩阵)
3. [阿里云官方 CLI（aliyun-cli）](#3-阿里云官方-cli)
4. [各产品独立 CLI 工具](#4-各产品独立-cli-工具)
5. [基础设施即代码工具](#5-基础设施即代码工具)
6. [Serverless 开发工具](#6-serverless-开发工具)
7. [第三方工具和社区生态](#7-第三方工具和社区生态)
8. [CLI 输出解析和数据提取](#8-cli-输出解析和数据提取)
9. [Agent 集成推荐方案](#9-agent-集成推荐方案)
10. [附录：命令速查表](#10-附录命令速查表)

---

## 1. 执行摘要

阿里云 CLI 工具生态呈现**"1+N"**格局：

- **1 个核心**：aliyun-cli 作为统一入口，覆盖 200+ 云产品
- **N 个专用工具**：针对高频场景（OSS、日志服务、MSE 等）提供专用 CLI
- **IaC 集成**：Terraform、Pulumi、ROS 提供基础设施即代码能力
- **Serverless 工具链**：Serverless Devs 提供完整的函数计算开发体验

**核心发现**：
1. aliyun-cli 基于 OpenAPI 自动生成，保持与产品能力同步
2. 专用 CLI 工具（ossutil、aliyun-log-cli、mseutil）在特定场景下效率更高
3. JSON + JMESPath 输出格式适合 Agent 自动化解析
4. Terraform/Pulumi 适合声明式资源管理，Ansible 适合配置管理

---

## 2. CLI 工具分类矩阵

### 2.1 产品 x 工具类型矩阵

| 产品/服务 | aliyun-cli | 专用 CLI | Terraform | Pulumi | ROS CLI | Serverless Devs |
|-----------|:----------:|:--------:|:---------:|:------:|:-------:|:---------------:|
| **ECS** | ✅ | - | ✅ | ✅ | ✅ | - |
| **RDS** | ✅ | - | ✅ | ✅ | ✅ | - |
| **Redis** | ✅ | - | ✅ | ✅ | ✅ | - |
| **OSS** | ✅ | ossutil | ✅ | ✅ | ✅ | - |
| **RocketMQ** | ✅ | - | ✅ | ✅ | ✅ | - |
| **Kafka** | ✅ | - | ✅ | ✅ | ✅ | - |
| **MSE/Nacos** | ✅ | mseutil | ✅ | ✅ | ✅ | - |
| **SLS 日志服务** | ✅ | aliyun-log-cli | ✅ | ✅ | ✅ | - |
| **CDN** | ✅ | - | ✅ | ✅ | ✅ | - |
| **VPC/SLB** | ✅ | - | ✅ | ✅ | ✅ | - |
| **函数计算 FC** | ✅ | - | ✅ | ✅ | ✅ | ✅ |
| **API 网关** | ✅ | - | ✅ | ✅ | ✅ | - |
| **ACK 容器** | ✅ | kubectl | ✅ | ✅ | ✅ | - |

### 2.2 工具特性对比

| 特性 | aliyun-cli | 专用 CLI | Terraform | Pulumi | Serverless Devs |
|------|:----------:|:--------:|:---------:|:------:|:---------------:|
| **安装难度** | 低 | 低 | 中 | 中 | 低 |
| **学习曲线** | 中 | 低 | 高 | 中 | 低 |
| **自动化友好** | 高 | 高 | 高 | 高 | 中 |
| **输出格式** | JSON/Text/Table | JSON/Text | State 文件 | State 文件 | YAML/JSON |
| **幂等性** | 否 | 否 | 是 | 是 | 是 |
| **回滚能力** | 否 | 否 | 是 | 是 | 部分 |
| **多云支持** | 否 | 否 | 是 | 是 | 是 |
| **Agent 集成** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐ |

---

## 3. 阿里云官方 CLI（aliyun-cli）

### 3.1 安装指南

```bash
# Linux/macOS 一键安装
/bin/bash -c "$(curl -fsSL https://aliyuncli.alicdn.com/install.sh)"

# macOS Homebrew
brew install aliyun-cli

# Windows PowerShell
powershell.exe -ExecutionPolicy Bypass -File Install-CLI-Windows.ps1

# 验证安装
aliyun version
```

### 3.2 配置管理

```bash
# 基础配置
aliyun configure
# 输入：Access Key ID、Access Key Secret、Region、Output Format

# 多账号配置（推荐）
aliyun configure --profile prod-account
aliyun configure --profile dev-account

# 查看配置列表
aliyun configure list

# 使用指定 profile
aliyun <command> --profile prod-account

# 启用自动补全
aliyun auto-completion
```

### 3.3 中间件产品命令示例

#### RDS（关系型数据库）

```bash
# 列出所有实例
aliyun rds DescribeDBInstances

# 创建 MySQL 实例
aliyun rds CreateDBInstance \
    --RegionId cn-hangzhou \
    --Engine MySQL \
    --EngineVersion "8.0" \
    --DBInstanceClass mysql.n2.medium.1 \
    --DBInstanceStorage 100 \
    --PayType Postpaid

# 创建数据库
aliyun rds CreateDatabase \
    --DBInstanceId rm-xxxxxxxx \
    --DBName mydatabase \
    --CharacterSetName utf8mb4

# 创建账号
aliyun rds CreateAccount \
    --DBInstanceId rm-xxxxxxxx \
    --AccountName admin \
    --AccountPassword "YourPassword123!" \
    --AccountType Super

# 查询备份列表
aliyun rds DescribeBackups --DBInstanceId rm-xxxxxxxx
```

#### Redis（云数据库 Redis 版）

```bash
# 列出所有实例
aliyun r-kvstore DescribeInstances

# 创建 Redis 实例
aliyun r-kvstore CreateInstance \
    --RegionId cn-hangzhou \
    --InstanceType Redis \
    --EngineVersion "7.0" \
    --InstanceClass redis.master.small.default \
    --Password "YourRedisPassword123!"

# 修改白名单
aliyun r-kvstore ModifySecurityIps \
    --InstanceId r-xxxxxxxx \
    --SecurityIps "192.168.0.0/16,10.0.0.0/8"

# 查询慢日志
aliyun r-kvstore DescribeSlowLogRecords \
    --InstanceId r-xxxxxxxx \
    --StartTime "2025-01-01T00:00:00Z" \
    --EndTime "2025-01-31T23:59:59Z"
```

#### RocketMQ（消息队列）

```bash
# 列出所有实例
aliyun ons DescribeInstances

# 创建实例
aliyun ons CreateInstance \
    --RegionId cn-hangzhou \
    --InstanceName "prod-rocketmq" \
    --ProductType rmq

# 创建 Topic
aliyun ons CreateTopic \
    --InstanceId mq-xxxxxxxx \
    --Topic "order-topic" \
    --MessageType NORMAL

# 创建 Group
aliyun ons CreateGroup \
    --InstanceId mq-xxxxxxxx \
    --GroupId "GID_ORDER_CONSUMER" \
    --GroupType tcp

# 查询消费堆积
aliyun ons DescribeConsumerAccumulate \
    --InstanceId mq-xxxxxxxx \
    --GroupId "GID_ORDER_CONSUMER"
```

#### Kafka（消息队列 Kafka 版）

```bash
# 获取实例列表
aliyun alikafka get-instance-list --region cn-hangzhou

# 创建 Topic
aliyun alikafka create-topic \
    --InstanceId kafka-xxxxxxxx \
    --Topic test-topic \
    --Remark "测试 Topic"

# 查询消费进度
aliyun alikafka get-consumer-progress \
    --InstanceId kafka-xxxxxxxx \
    --ConsumerId test-consumer
```

### 3.4 输出格式支持

```bash
# JSON 格式（默认，适合程序解析）
aliyun ecs DescribeInstances --output json

# 表格格式（适合人工阅读）
aliyun ecs DescribeInstances --output table

# 文本格式（适合 Unix 工具处理）
aliyun ecs DescribeInstances --output text

# 自定义表格输出
aliyun ecs DescribeInstances \
  --output cols="InstanceId,Status,PrivateIp" \
  rows="Instances.Instance[]"
```

---

## 4. 各产品独立 CLI 工具

### 4.1 OSS：ossutil

**安装**：
```bash
# 下载对应版本
wget https://gosspublic.alicdn.com/ossutil/1.7.19/ossutil64
chmod 755 ossutil64
mv ossutil64 /usr/local/bin/ossutil

# 配置
ossutil config
```

**常用命令**：
```bash
# 上传文件
ossutil cp localfile.txt oss://mybucket/

# 下载文件
ossutil cp oss://mybucket/remotefile.txt ./

# 同步目录
ossutil sync localdir/ oss://mybucket/dest/

# 列举文件
ossutil ls oss://mybucket

# 通过阿里云 CLI 调用 ossutil
aliyun oss ossutil cp localfile.txt oss://mybucket/
```

### 4.2 日志服务：aliyun-log-cli

**安装**：
```bash
pip install aliyun-log-cli
```

**配置**：
```bash
aliyunlog configure
# 输入：Access ID、Access Key、Endpoint
```

**常用命令**：
```bash
# 查询日志
aliyunlog log get_logs \
  --project="my-project" \
  --logstore="my-logstore" \
  --query="*" \
  --from_time="2025-01-01 00:00:00" \
  --to_time="2025-01-01 23:59:59"

# 批量获取日志（自动分页）
aliyunlog log get_log_all \
  --project="my-project" \
  --logstore="my-logstore" \
  --query="host:test.com and response_time>5000"

# JMESPath 过滤输出
aliyunlog log get_logs ... --jmes-filter="join('\n', map(&to_string(@), @))"
```

### 4.3 MSE：mseutil

**安装**：
```bash
# 根据架构选择下载
wget https://msetools.oss-cn-hangzhou.aliyuncs.com/mseutil/linux/x86_64/mseutil
chmod +x mseutil
```

**Nacos 操作**：
```bash
# 查询配置
mseutil nacos get --serverAddr=mse-xxx.mse.aliyuncs.com:8848 --dataId=test --group=DEFAULT_GROUP

# 发布配置
mseutil nacos set --serverAddr=mse-xxx.mse.aliyuncs.com:8848 --dataId=test --group=DEFAULT_GROUP --content="test=value"

# 删除配置
mseutil nacos delete --serverAddr=mse-xxx.mse.aliyuncs.com:8848 --dataId=test --group=DEFAULT_GROUP
```

**ZooKeeper 操作**：
```bash
# 列出节点
mseutil zookeeper ls --serverAddr=mse-xxx.mse.aliyuncs.com:2181 --path=/

# 获取节点数据
mseutil zookeeper get --serverAddr=mse-xxx.mse.aliyuncs.com:2181 --path=/config

# 创建节点
mseutil zookeeper create --serverAddr=mse-xxx.mse.aliyuncs.com:2181 --path=/test --data="test-data"
```

---

## 5. 基础设施即代码工具

### 5.1 Terraform Alicloud Provider

**安装**：
```bash
# macOS
brew install terraform

# 其他系统参考 https://developer.hashicorp.com/terraform/downloads
```

**Provider 配置**：
```hcl
terraform {
  required_providers {
    alicloud = {
      source  = "aliyun/alicloud"
      version = ">= 1.220.0"
    }
  }
}

provider "alicloud" {
  region  = "cn-hangzhou"
  profile = "Your-Profile-Name"
}
```

**RDS 资源示例**：
```hcl
# 创建 VPC 和交换机
resource "alicloud_vpc" "vpc" {
  vpc_name   = "tf-vpc"
  cidr_block = "172.16.0.0/12"
}

resource "alicloud_vswitch" "vswitch" {
  vpc_id       = alicloud_vpc.vpc.id
  cidr_block   = "172.16.0.0/21"
  zone_id      = "cn-hangzhou-h"
}

# 创建 RDS MySQL 实例
resource "alicloud_db_instance" "rds" {
  engine               = "MySQL"
  engine_version       = "8.0"
  instance_type        = "rds.mysql.s2.large"
  instance_storage     = 100
  instance_charge_type = "Postpaid"
  vswitch_id           = alicloud_vswitch.vswitch.id
  security_ips         = ["10.0.0.0/8"]
}

# 创建数据库
resource "alicloud_db_database" "db" {
  instance_id   = alicloud_db_instance.rds.id
  name          = "mydatabase"
  character_set = "utf8mb4"
}

# 创建账号
resource "alicloud_db_account" "account" {
  instance_id = alicloud_db_instance.rds.id
  name        = "dbuser"
  password    = var.db_password
  type        = "Normal"
}
```

**Redis 资源示例**：
```hcl
resource "alicloud_kvstore_instance" "redis" {
  instance_name  = "tf-redis"
  instance_type  = "Redis"
  engine_version = "7.0"
  instance_class = "redis.master.small.default"
  vswitch_id     = alicloud_vswitch.vswitch.id
  password       = var.redis_password
  security_ips   = ["10.0.0.0/8", "172.16.0.0/12"]

  # 备份配置
  backup_period = ["Monday", "Wednesday", "Friday"]
  backup_time   = "02:00Z-03:00Z"
}
```

**使用模块（推荐）**：
```hcl
module "rds" {
  source  = "terraform-alicloud-modules/rds/alicloud"

  engine           = "MySQL"
  engine_version   = "8.0"
  instance_type    = "rds.mysql.s2.large"
  instance_storage = 100
  vswitch_id       = alicloud_vswitch.vswitch.id

  create_database = true
  databases = [
    {
      name          = "mydb"
      character_set = "utf8mb4"
      description   = "Main database"
    }
  ]

  create_account   = true
  account_name     = "admin"
  account_password = var.password
}
```

### 5.2 Pulumi Alibaba Cloud Provider

**安装**：
```bash
# 安装 Pulumi CLI
curl -fsSL https://get.pulumi.com | sh

# Python SDK
pip install pulumi-alicloud
```

**Python 示例**：
```python
import pulumi
import pulumi_alicloud as alicloud

# 创建 VPC
vpc = alicloud.vpc.Network("my-vpc", cidr_block="172.16.0.0/12")

# 创建交换机
vswitch = alicloud.vpc.Switch("my-vswitch",
    availability_zone="cn-hangzhou-i",
    cidr_block="172.16.0.0/21",
    vpc_id=vpc.id)

# 创建 ECS 实例
instance = alicloud.ecs.Instance("my-instance",
    instance_type="ecs.t6-c1m1.large",
    image_id="ubuntu_18_04_64_20G_alibase_20190624.vhd",
    vswitch_id=vswitch.id)

# 输出
pulumi.export("instance_id", instance.id)
pulumi.export("public_ip", instance.public_ip)
```

### 5.3 ROS CLI

**基础命令**：
```bash
# 创建资源栈
aliyun ros CreateStack \
  --RegionId cn-hangzhou \
  --StackName MyStack \
  --TimeoutInMinutes 10 \
  --TemplateURL oss://ros-template/demo

# 查询资源栈
aliyun ros DescribeStack \
  --StackId c18d62d8-51ce-4e8e-b8f6-e00be431****

# 更新资源栈
aliyun ros UpdateStack \
  --StackId c18d62d8-51ce-4e8e-b8f6-e00be431**** \
  --TemplateURL oss://ros-template/demo-v2

# 删除资源栈
aliyun ros DeleteStack \
  --StackId c18d62d8-51ce-4e8e-b8f6-e00be431****
```

**ROS CDK CLI**：
```bash
# 初始化项目
ros-cdk init --language=typescript --generate-only

# 合成模板
ros-cdk synth

# 部署
ros-cdk deploy

# 删除
ros-cdk destroy
```

---

## 6. Serverless 开发工具

### 6.1 Serverless Devs（s 工具）

**安装**：
```bash
npm install -g @serverless-devs/s

# 验证
s -v
```

**配置**：
```bash
# 添加阿里云凭证
s config add

# 查看配置
s config get
```

**常用命令**：
```bash
# 初始化项目
s init devsapp/start-fc-http-python3

# 部署
s deploy -y

# 本地调试
s local start

# 查看日志
s logs

# 删除资源
s remove
```

**s.yaml 配置示例**：
```yaml
edition: 3.0.0
name: hello-world-app
access: default

resources:
  hello_world_function:
    component: fc3
    props:
      region: cn-hangzhou
      functionName: hello-world
      runtime: python3.9
      code: ./code
      handler: index.handler
      memorySize: 128
      timeout: 30
      triggers:
        - triggerName: httpTrigger
          triggerType: http
          triggerConfig:
            authType: anonymous
            methods:
              - GET
              - POST
```

### 6.2 中间件集成能力

Serverless Devs 支持在函数计算中集成中间件：

| 中间件 | 集成方式 |
|--------|---------|
| **RDS** | VPC 内网访问 + 连接池 |
| **Redis** | VPC 内网访问 + Redis 客户端 |
| **OSS** | 触发器 + SDK |
| **RocketMQ** | 触发器 + SDK |
| **MNS** | 触发器 + SDK |

---

## 7. 第三方工具和社区生态

### 7.1 GitHub 开源项目

| 项目 | 地址 | 说明 |
|------|------|------|
| aliyun-cli | https://github.com/aliyun/aliyun-cli | 官方 CLI |
| ossutil | https://github.com/aliyun/ossutil | OSS 工具 |
| aliyun-log-cli | https://github.com/aliyun/aliyun-log-cli | 日志服务 CLI |
| fcli | https://github.com/aliyun/fcli | 函数计算 CLI（已合并到 Serverless Devs） |
| ChaosBlade | https://github.com/chaosblade-io/chaosblade | 混沌工程工具 |
| configure-aliyun-credentials-action | https://github.com/aliyun/configure-aliyun-credentials-action | GitHub Actions |

### 7.2 Ansible 集成

```yaml
# playbook.yml
- name: Manage Aliyun ECS
  hosts: localhost
  tasks:
    - name: Create ECS instance
      ali_instance:
        image_id: ubuntu_18_04_64_20G_alibase_20190624.vhd
        type: ecs.t6-c1m1.large
        vswitch_id: vsw-xxxxxxxx
        security_groups: [sg-xxxxxxxx]
        state: present
      register: instance

    - name: Display instance info
      debug:
        var: instance
```

### 7.3 Cloud Shell 预装工具

阿里云 Cloud Shell（https://shell.aliyun.com/）预装：
- `aliyun` CLI
- `terraform`
- `ansible`
- `kubectl`
- `helm`
- `docker`

---

## 8. CLI 输出解析和数据提取

### 8.1 JMESPath 查询语法

阿里云 CLI 使用 JMESPath 作为查询语言：

| 表达式 | 说明 | 示例 |
|--------|------|------|
| `key` | 提取对象属性 | `InstanceId` |
| `key.subkey` | 嵌套属性 | `NetworkInterfaces.NetworkInterface` |
| `[]` | 展开数组 | `Instances.Instance[]` |
| `[*]` | 遍历数组 | `Instances.Instance[*]` |
| `[0]` | 取第一个元素 | `SecurityGroupIds[0]` |
| `[?condition]` | 条件过滤 | `[?Status=='Running']` |
| `\| pipe` | 管道操作 | `[] \| length(@)` |

### 8.2 数据提取实战

```bash
# 提取所有 RDS 实例 ID
aliyun rds DescribeDBInstances | jq -r '.Items.DBInstance[].DBInstanceId'

# 提取 Redis 连接地址
aliyun r-kvstore DescribeInstances | jq -r '.Instances.KVStoreInstance[].ConnectionDomain'

# 过滤运行中的实例
aliyun rds DescribeDBInstances | jq '.Items.DBInstance[] | select(.DBInstanceStatus == "Running")'

# 获取数组长度
aliyun ecs DescribeInstances --query "length(Instances.Instance)"

# 复杂对象重构
aliyun ecs DescribeInstances \
  --output cols="InstanceId,Status,PrivateIp" \
  rows='Instances.Instance[].{InstanceId:InstanceId,Status:Status,PrivateIp:NetworkInterfaces.NetworkInterface[0].PrimaryIpAddress}'
```

### 8.3 配合 jq 工具

```bash
# 安装 jq
# macOS: brew install jq
# Linux: apt-get install jq / yum install jq

# 格式化 JSON
aliyun ecs DescribeRegions | jq

# 提取特定字段
aliyun ecs DescribeInstances | jq '.Instances.Instance[].InstanceId'

# 过滤并映射
aliyun rds DescribeDBInstances | jq '.Items.DBInstance[] | {id: .DBInstanceId, status: .DBInstanceStatus}'

# 数组操作
aliyun r-kvstore DescribeInstances | jq '[.Instances.KVStoreInstance[] | select(.InstanceStatus == "Normal")]'
```

---

## 9. Agent 集成推荐方案

### 9.1 方案选型矩阵

| 场景 | 推荐工具 | 理由 |
|------|---------|------|
| **临时查询/探索** | aliyun-cli | 快速、直接 |
| **批量操作/脚本** | aliyun-cli + jq | 自动化友好 |
| **文件传输** | ossutil | 高效、支持断点续传 |
| **日志分析** | aliyun-log-cli | 专业、支持 JMESPath |
| **配置管理** | mseutil | 针对 Nacos/ZK 优化 |
| **资源编排** | Terraform | 声明式、幂等、状态管理 |
| **函数计算** | Serverless Devs | 完整生命周期管理 |
| **CI/CD 集成** | aliyun-cli / Terraform | 标准化、可重复 |

### 9.2 Agent 工具层设计建议

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent 工具层架构                          │
├─────────────────────────────────────────────────────────────┤
│  应用层：CLI Wrapper / SDK Wrapper / API Gateway             │
├─────────────────────────────────────────────────────────────┤
│  抽象层：Resource Manager / Operation Executor               │
├─────────────────────────────────────────────────────────────┤
│  适配层：                                                     │
│    ├─ aliyun-cli Adapter (通用 OpenAPI)                     │
│    ├─ ossutil Adapter (文件操作)                            │
│    ├─ aliyun-log-cli Adapter (日志查询)                     │
│    ├─ mseutil Adapter (配置中心)                            │
│    ├─ Terraform Adapter (IaC)                              │
│    └─ Serverless Devs Adapter (FC)                         │
├─────────────────────────────────────────────────────────────┤
│  输出解析层：JSON Parser / JMESPath Engine / Table Parser   │
├─────────────────────────────────────────────────────────────┤
│  凭证管理层：Profile Manager / STS Token / RAM Role         │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 推荐集成模式

#### 模式一：CLI 命令执行器（适合快速集成）

```python
import subprocess
import json

class AliyunCLIExecutor:
    def __init__(self, profile=None):
        self.profile = profile

    def execute(self, product, action, params=None):
        cmd = ['aliyun', product, action]
        if self.profile:
            cmd.extend(['--profile', self.profile])
        if params:
            for k, v in params.items():
                cmd.extend([f'--{k}', str(v)])
        cmd.extend(['--output', 'json'])

        result = subprocess.run(cmd, capture_output=True, text=True)
        return json.loads(result.stdout) if result.returncode == 0 else None

    def query_rds_instances(self):
        return self.execute('rds', 'DescribeDBInstances')

    def query_redis_instances(self):
        return self.execute('r-kvstore', 'DescribeInstances')
```

#### 模式二：结构化资源操作（适合复杂场景）

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class RDSInstance:
    instance_id: str
    status: str
    engine: str
    connection_string: str

class RDSManager:
    def __init__(self, cli_executor):
        self.cli = cli_executor

    def list_instances(self) -> List[RDSInstance]:
        result = self.cli.execute('rds', 'DescribeDBInstances')
        return [
            RDSInstance(
                instance_id=item['DBInstanceId'],
                status=item['DBInstanceStatus'],
                engine=item['Engine'],
                connection_string=item['ConnectionString']
            )
            for item in result.get('Items', {}).get('DBInstance', [])
        ]

    def get_instance_by_id(self, instance_id: str) -> Optional[RDSInstance]:
        result = self.cli.execute('rds', 'DescribeDBInstanceAttribute', {
            'DBInstanceId': instance_id
        })
        # 解析并返回...
```

### 9.4 安全最佳实践

1. **凭证管理**
   - 使用 RAM 用户而非主账号
   - 启用 MFA 多因素认证
   - 定期轮换 AccessKey
   - 优先使用 STS 临时凭证

2. **权限最小化**
   - 为 Agent 创建专用 RAM 角色
   - 仅授予必要的权限策略
   - 使用资源组隔离环境

3. **审计日志**
   - 启用 ActionTrail 操作审计
   - 记录所有 CLI 调用
   - 定期审查异常操作

---

## 10. 附录：命令速查表

### 10.1 RDS 常用命令

| 操作 | 命令 |
|------|------|
| 列出实例 | `aliyun rds DescribeDBInstances` |
| 创建实例 | `aliyun rds CreateDBInstance` |
| 删除实例 | `aliyun rds DeleteDBInstance` |
| 查询详情 | `aliyun rds DescribeDBInstanceAttribute` |
| 重启实例 | `aliyun rds RestartDBInstance` |
| 创建数据库 | `aliyun rds CreateDatabase` |
| 创建账号 | `aliyun rds CreateAccount` |
| 创建备份 | `aliyun rds CreateBackup` |
| 查询备份 | `aliyun rds DescribeBackups` |
| 修改白名单 | `aliyun rds ModifySecurityIps` |

### 10.2 Redis 常用命令

| 操作 | 命令 |
|------|------|
| 列出实例 | `aliyun r-kvstore DescribeInstances` |
| 创建实例 | `aliyun r-kvstore CreateInstance` |
| 删除实例 | `aliyun r-kvstore DeleteInstance` |
| 查询详情 | `aliyun r-kvstore DescribeInstanceAttribute` |
| 重启实例 | `aliyun r-kvstore RestartInstance` |
| 创建备份 | `aliyun r-kvstore CreateBackup` |
| 修改白名单 | `aliyun r-kvstore ModifySecurityIps` |
| 修改密码 | `aliyun r-kvstore ModifyPassword` |
| 查询慢日志 | `aliyun r-kvstore DescribeSlowLogRecords` |

### 10.3 RocketMQ 常用命令

| 操作 | 命令 |
|------|------|
| 列出实例 | `aliyun ons DescribeInstances` |
| 创建实例 | `aliyun ons CreateInstance` |
| 创建 Topic | `aliyun ons CreateTopic` |
| 删除 Topic | `aliyun ons DeleteTopic` |
| 创建 Group | `aliyun ons CreateGroup` |
| 删除 Group | `aliyun ons DeleteGroup` |
| 查询消费进度 | `aliyun ons DescribeConsumerAccumulate` |
| 重置消费位点 | `aliyun ons ResetConsumerOffset` |

### 10.4 Kafka 常用命令

| 操作 | 命令 |
|------|------|
| 列出实例 | `aliyun alikafka get-instance-list` |
| 创建 Topic | `aliyun alikafka create-topic` |
| 删除 Topic | `aliyun alikafka delete-topic` |
| 查询 Topic | `aliyun alikafka get-topic-list` |
| 创建消费组 | `aliyun alikafka create-consumer-group` |
| 查询消费进度 | `aliyun alikafka get-consumer-progress` |

---

## 参考资源

1. [阿里云 CLI 官方文档](https://help.aliyun.com/zh/cli/)
2. [阿里云 OpenAPI 开发者门户](https://next.api.aliyun.com/)
3. [Terraform Alicloud Provider](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs)
4. [Pulumi Alicloud Provider](https://www.pulumi.com/registry/packages/alicloud/)
5. [Serverless Devs 文档](https://www.serverless-devs.com/)
6. [ROS 资源编排文档](https://help.aliyun.com/zh/ros/)
7. [JMESPath 教程](https://jmespath.org/tutorial.html)

---

*报告生成时间：2026-04-09*
*调研人员：Claude Code Agent*
