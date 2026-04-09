# 阿里云中间件 OpenAPI 能力调研报告

> 调研日期：2026-04-09
> 调研目标：全面了解阿里云中间件产品的 OpenAPI 覆盖度，为构建 MCP Server 提供技术参考

---

## 目录

1. [执行摘要](#一执行摘要)
2. [各产品 OpenAPI 能力矩阵](#二各产品-openapi-能力矩阵)
3. [核心产品 API 详细分析](#三核心产品-api-详细分析)
4. [认证和权限模型](#四认证和权限模型)
5. [SDK 使用示例（Go 语言）](#五sdk-使用示例go-语言)
6. [API 限制和注意事项](#六api-限制和注意事项)
7. [推荐用于 MCP Server 封装的 API 清单](#七推荐用于-mcp-server-封装的-api-清单)

---

## 一、执行摘要

### 1.1 整体评估

阿里云中间件产品线的 OpenAPI 覆盖度整体较高，主要特点：

| 维度 | 评估 | 说明 |
|------|------|------|
| **API 完整性** | 良好 | 核心功能（CRUD、监控、配置）基本覆盖 |
| **文档质量** | 优秀 | 官方 OpenAPI 门户文档详尽，支持在线调试 |
| **SDK 支持** | 优秀 | Java、Go、Python、Node.js 等多语言支持 |
| **认证方式** | 标准 | 支持 AK/SK、STS、RAM Role 等多种方式 |
| **调用限制** | 中等 | 默认 QPS 1-1000 次/秒不等，需关注流控 |

### 1.2 产品覆盖度总览

| 产品 | API 数量 | 覆盖度 | 推荐优先级 |
|------|---------|--------|-----------|
| 云数据库 RDS | 150+ | 高 | P0 |
| 云数据库 Redis | 100+ | 高 | P0 |
| 消息队列 RocketMQ | 50+ | 高 | P0 |
| 日志服务 SLS | 80+ | 高 | P0 |
| 微服务引擎 MSE | 60+ | 中 | P1 |
| 应用监控 ARMS | 70+ | 中 | P1 |
| 消息队列 Kafka | 40+ | 中 | P1 |
| 任务调度 SchedulerX | 30+ | 中 | P2 |
| 消息服务 MNS | 20+ | 低 | P2 |

---

## 二、各产品 OpenAPI 能力矩阵

### 2.1 云数据库 RDS

**API 版本**: 2014-08-15（经典版）/ 2025-05-07（AI 服务版）

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **实例管理** | DescribeDBInstances, CreateDBInstance, DeleteDBInstance, RestartDBInstance | 完整支持 |
| **数据库管理** | CreateDatabase, DeleteDatabase, DescribeDatabases | 完整支持 |
| **账号管理** | CreateAccount, DeleteAccount, DescribeAccounts, GrantAccountPrivilege | 完整支持 |
| **备份恢复** | CreateBackup, DescribeBackups, RestoreDBInstance | 完整支持 |
| **监控查询** | DescribeDBInstancePerformance, DescribeResourceUsage | 完整支持 |
| **网络配置** | DescribeDBInstanceNetInfo, SwitchNetwork | 完整支持 |
| **参数管理** | DescribeParameters, ModifyParameter | 完整支持 |
| **只读实例** | CreateReadOnlyDBInstance, DescribeReadDBInstanceDelay | 完整支持 |
| **代理配置** | DescribeDBProxy, ModifyDBProxy | 完整支持 |
| **AI 服务** | CreateAppInstance, ChatMessages, CreateInspectionTask | 2025 新增 |

**支持的引擎**: MySQL、PostgreSQL、SQL Server、MariaDB

**官方文档**: https://api.aliyun.com/product/Rds

### 2.2 云数据库 Redis（Tair）

**API 版本**: 2015-01-01

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **实例管理** | DescribeInstances, CreateInstance, DeleteInstance, RestartInstance | 完整支持 |
| **规格变更** | ModifyInstanceSpec | 完整支持 |
| **连接管理** | ModifyDBInstanceConnectionString, AllocateInstancePublicConnection | 完整支持 |
| **参数配置** | DescribeParameters, ModifyInstanceConfig | 完整支持 |
| **备份恢复** | CreateBackup, DescribeBackups, RestoreInstance | 完整支持 |
| **监控查询** | DescribeInstanceAttribute, DescribeHistoryMonitorValues | 完整支持 |
| **高可用** | SwitchInstanceHA, LockDBInstanceWrite | 完整支持 |
| **数据清空** | FlushInstance, FlushExpireKeys | 完整支持 |

**官方文档**: https://next.api.aliyun.com/document/R-kvstore

### 2.3 消息队列 RocketMQ

**API 版本**: 2019-02-14（4.x）/ 2022-08-01（5.x）

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **实例管理** | OnsInstanceCreate, OnsInstanceDelete, OnsInstanceInServiceList | 完整支持 |
| **Topic 管理** | OnsTopicCreate, OnsTopicDelete, OnsTopicList | 完整支持 |
| **Group 管理** | OnsGroupCreate, OnsGroupDelete, OnsGroupList, OnsConsumerStatus | 完整支持 |
| **消息查询** | OnsMessageGetByMsgId, OnsMessageGetByKey, OnsMessagePageQueryByTopic | 完整支持 |
| **消息轨迹** | OnsMessageTrace | 完整支持 |
| **消费查询** | OnsConsumerStatus, OnsTopicSubDetail | 完整支持 |
| **5.x 新 API** | ListMessages, GetConsumeTimespan | 5.x 版本 |

**重要提示**: OpenAPI 仅用于管控链路，消息收发请使用 SDK

**官方文档**: https://api.aliyun.com/api/Ons/2019-02-14

### 2.4 日志服务 SLS

**API 版本**: 2020-12-30

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **Project 管理** | CreateProject, GetProject, DeleteProject, ListProject | 完整支持 |
| **Logstore 管理** | CreateLogStore, GetLogStore, UpdateLogStore, DeleteLogStore, ListLogStores | 完整支持 |
| **日志查询** | GetLogsV2（推荐）, GetLogs（旧版）, GetHistograms | 完整支持 |
| **上下文查询** | GetContextLogs | 完整支持 |
| **消费组管理** | CreateConsumerGroup, ListConsumerGroup, UpdateConsumerGroup, DeleteConsumerGroup | 完整支持 |
| **数据消费** | GetCursor, PullLogs | 完整支持 |
| **告警管理** | CreateAlert, GetAlert, DeleteAlert | 完整支持 |
| **投递配置** | 支持投递到 OSS、MaxCompute、Kafka | 完整支持 |

**查询语法**: 支持 SPL 查询和 SQL 分析

**官方文档**: https://api.aliyun.com/document/Sls/2020-12-30/overview

### 2.5 微服务引擎 MSE

**API 版本**: 2019-05-31

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **Nacos 服务管理** | CreateNacosService, ListAnsServices, DeleteNacosService | 完整支持 |
| **Nacos 配置管理** | ListNacosConfigs, UpdateNacosConfig, ImportNacosConfig | 完整支持 |
| **Nacos 实例管理** | CreateNacosInstance, ListAnsInstances, UpdateNacosCluster | 完整支持 |
| **命名空间管理** | CreateEngineNamespace, ListNamespaces, DeleteNamespace | 完整支持 |
| **配置轨迹** | ListConfigTrack, ListNamingTrack | 完整支持 |
| **云原生网关** | AddServiceSource, GetMseSource, ListServiceSource | 完整支持 |
| **ZooKeeper 管理** | 通过 MSE 控制台 API 支持 | 部分支持 |

**官方文档**: https://api.aliyun.com/document/mse

### 2.6 应用实时监控 ARMS

**API 版本**: 2019-08-08

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **应用监控** | GetTraceApp, SearchTraceAppByPage, ListTraceApps, GetTrace | 完整支持 |
| **调用链查询** | SearchTracesByPage, GetMultipleTrace | 完整支持 |
| **拓扑查询** | QueryAppTopology | 完整支持 |
| **前端监控** | CreateRetcodeApp, ListRetcodeApps, GetRetcodeDataByQuery | 完整支持 |
| **告警规则** | CreateAlertRule, UpdateAlertRule, GetAlertRules, DeleteAlertRule | 完整支持 |
| **Prometheus 告警** | CreatePrometheusAlertRule, ListPrometheusAlertRules | 完整支持 |
| **告警联系人** | CreateAlertContact, SearchAlertContact, CreateAlertContactGroup | 完整支持 |
| **指标查询** | QueryMetricByPage, GetAppApiByPage | 完整支持 |

**官方文档**: https://help.aliyun.com/zh/arms/developer-reference/api-reference

### 2.7 消息队列 Kafka

**API 版本**: 2019-09-16

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **实例管理** | GetInstanceList, CreatePostPayOrder, CreatePrePayOrder | 完整支持 |
| **实例操作** | StartInstance, ModifyInstanceName | 完整支持 |
| **Topic 管理** | CreateTopic, DeleteTopic, GetTopicList | 完整支持 |
| **Group 管理** | CreateConsumerGroup, DeleteConsumerGroup | 完整支持 |
| **白名单管理** | GetAllowedIpList, UpdateAllowedIp | 完整支持 |
| **自动创建** | EnableAutoTopicCreation, EnableAutoGroupCreation | 完整支持 |
| **弹性策略** | CreateScheduledScalingRule, GetAutoScalingConfiguration | Serverless 版 |

**官方文档**: https://next.api.aliyun.com/document/alikafka

### 2.8 任务调度 SchedulerX

**API 版本**: 2019-04-30（SchedulerX2.0）

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **任务管理** | CreateJob, UpdateJob, ListJobs, ExecuteJob | 完整支持 |
| **任务实例** | GetJobInstance, GetJobInstanceList, StopInstance, RetryJobInstance | 完整支持 |
| **工作流管理** | CreateWorkflow, UpdateWorkflow, GetWorkflow, GetWorkflowDAG | 完整支持 |
| **命名空间** | CreateNamespace, ListNamespaces | 完整支持 |
| **应用分组** | CreateAppGroup, ListGroups | 完整支持 |

**官方文档**: https://next.api.aliyun.com/document/schedulerx2

### 2.9 消息服务 MNS

**API 版本**: 2015-06-06

| 功能类别 | 核心 API | 支持状态 |
|---------|---------|---------|
| **队列管理** | CreateQueue, DeleteQueue, ListQueue, GetQueueAttributes | 完整支持 |
| **主题管理** | CreateTopic, DeleteTopic, ListTopic, GetTopicAttributes | 完整支持 |
| **消息操作** | SendMessage, ReceiveMessage, DeleteMessage, PeekMessage | 完整支持 |
| **订阅管理** | Subscribe, Unsubscribe, ListSubscription | 完整支持 |

**协议**: HTTP/HTTPS，请求格式 XML

**官方文档**: https://www.alibabacloud.com/help/zh/mns/developer-reference/request-protocol-description

---

## 三、核心产品 API 详细分析

### 3.1 RDS 核心 API 列表

#### 实例管理类

| API 名称 | 功能描述 | 关键参数 |
|---------|---------|---------|
| DescribeDBInstances | 查询 RDS 实例列表 | RegionId, DBInstanceId, Engine |
| CreateDBInstance | 创建 RDS 实例 | Engine, DBInstanceClass, DBInstanceStorage |
| DeleteDBInstance | 删除 RDS 实例 | DBInstanceId |
| RestartDBInstance | 重启 RDS 实例 | DBInstanceId |
| DescribeDBInstanceAttribute | 查询实例详情 | DBInstanceId |

#### 数据库和账号类

| API 名称 | 功能描述 | 关键参数 |
|---------|---------|---------|
| CreateDatabase | 创建数据库 | DBInstanceId, DBName, CharacterSetName |
| DeleteDatabase | 删除数据库 | DBInstanceId, DBName |
| DescribeDatabases | 查询数据库列表 | DBInstanceId |
| CreateAccount | 创建账号 | DBInstanceId, AccountName, AccountPassword |
| GrantAccountPrivilege | 授权账号 | DBInstanceId, AccountName, DBName, AccountPrivilege |

#### 备份恢复类

| API 名称 | 功能描述 | 关键参数 |
|---------|---------|---------|
| CreateBackup | 创建备份 | DBInstanceId, BackupMethod |
| DescribeBackups | 查询备份列表 | DBInstanceId, StartTime, EndTime |
| RestoreDBInstance | 恢复实例 | DBInstanceId, BackupId |
| DescribeBackupPolicy | 查询备份策略 | DBInstanceId |
| ModifyBackupPolicy | 修改备份策略 | DBInstanceId, BackupRetentionPeriod |

### 3.2 Redis 核心 API 列表

| API 名称 | 功能描述 | 关键参数 |
|---------|---------|---------|
| DescribeInstances | 查询实例列表 | RegionId, InstanceId |
| CreateInstance | 创建实例 | RegionId, InstanceClass, EngineVersion |
| DeleteInstance | 删除实例 | InstanceId |
| ModifyInstanceSpec | 变更规格 | InstanceId, InstanceClass |
| RestartInstance | 重启实例 | InstanceId |
| DescribeInstanceAttribute | 查询实例属性 | InstanceId |
| DescribeParameters | 查询参数 | InstanceId |
| ModifyInstanceConfig | 修改参数 | InstanceId, Config |
| CreateBackup | 创建备份 | InstanceId |
| DescribeBackups | 查询备份 | InstanceId |
| FlushInstance | 清空数据 | InstanceId |

### 3.3 RocketMQ 核心 API 列表

| API 名称 | 功能描述 | 关键参数 |
|---------|---------|---------|
| OnsInstanceInServiceList | 查询实例列表 | RegionId |
| OnsTopicList | 查询 Topic 列表 | InstanceId |
| OnsTopicCreate | 创建 Topic | InstanceId, Topic |
| OnsTopicDelete | 删除 Topic | InstanceId, Topic |
| OnsGroupList | 查询 Group 列表 | InstanceId |
| OnsGroupCreate | 创建 Group | InstanceId, GroupId |
| OnsConsumerStatus | 查询消费状态 | InstanceId, GroupId |
| OnsMessageGetByMsgId | 按 ID 查询消息 | InstanceId, Topic, MsgId |
| OnsMessageGetByKey | 按 Key 查询消息 | InstanceId, Topic, Key |
| OnsMessagePageQueryByTopic | 分页查询消息 | InstanceId, Topic, BeginTime, EndTime |
| OnsMessageTrace | 查询消息轨迹 | InstanceId, Topic, MsgId |

### 3.4 SLS 核心 API 列表

| API 名称 | 功能描述 | 关键参数 |
|---------|---------|---------|
| ListProject | 列出 Project | - |
| CreateProject | 创建 Project | projectName, description |
| GetProject | 获取 Project | projectName |
| DeleteProject | 删除 Project | projectName |
| ListLogStores | 列出 Logstore | projectName |
| CreateLogStore | 创建 Logstore | projectName, logstoreName, ttl, shardCount |
| GetLogStore | 获取 Logstore | projectName, logstoreName |
| DeleteLogStore | 删除 Logstore | projectName, logstoreName |
| GetLogsV2 | 查询日志（推荐） | projectName, logstoreName, from, to, query |
| GetHistograms | 查询日志分布 | projectName, logstoreName, from, to, query |
| GetContextLogs | 查询上下文 | projectName, logstoreName, pack_id, pack_meta |
| CreateConsumerGroup | 创建消费组 | projectName, logstoreName, consumerGroup |
| ListConsumerGroup | 列出消费组 | projectName, logstoreName |

---

## 四、认证和权限模型

### 4.1 认证方式对比

| 认证方式 | 适用场景 | 安全性 | 有效期 |
|---------|---------|--------|--------|
| **AccessKey** | 长期访问，服务器端应用 | 中 | 永久 |
| **RAM 用户 AK** | 权限管控，日常使用 | 中 | 永久 |
| **STS Token** | 临时授权，前端/移动端 | 高 | 1-12 小时 |
| **RamRoleArn** | 角色扮演，跨账号访问 | 高 | 自动刷新 |

### 4.2 推荐认证方式

**场景 1：MCP Server 服务端部署**
- 推荐：使用 RamRoleArn 方式
- 理由：自动刷新 STS Token，无需手动管理过期时间

**场景 2：本地开发测试**
- 推荐：使用环境变量存储 RAM 用户 AK
- 理由：简单直接，便于调试

**场景 3：CI/CD 流水线**
- 推荐：使用 OIDC 角色扮演
- 理由：无需管理长期凭证，安全性高

### 4.3 RAM 权限策略示例

#### RDS 只读权限
```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBInstanceAttribute",
        "rds:DescribeDatabases",
        "rds:DescribeAccounts",
        "rds:DescribeBackups",
        "rds:DescribeDBInstancePerformance"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Redis 完全权限
```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "r-kvstore:*",
      "Resource": "*"
    }
  ]
}
```

#### SLS 读写权限
```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "log:ListProject",
        "log:GetProject",
        "log:ListLogStores",
        "log:GetLogStore",
        "log:GetLogs",
        "log:GetHistograms"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 五、SDK 使用示例（Go 语言）

### 5.1 安装依赖

```bash
# 核心 SDK
go get github.com/alibabacloud-go/darabonba-openapi/v2

# 凭证管理
go get -u github.com/aliyun/credentials-go

# 各产品 SDK
go get github.com/alibabacloud-go/rds-20140815/v6
go get github.com/alibabacloud-go/r-kvstore-20150101/v4
go get github.com/alibabacloud-go/sls-20201230/v3
go get github.com/alibabacloud-go/tea-utils/v2
```

### 5.2 RDS 调用示例

```go
package main

import (
	"fmt"
	"os"

	openapi "github.com/alibabacloud-go/darabonba-openapi/v2/client"
	rds20140815 "github.com/alibabacloud-go/rds-20140815/v6/client"
	"github.com/alibabacloud-go/tea/tea"
)

func main() {
	// 初始化配置
	config := &openapi.Config{
		AccessKeyId:     tea.String(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_ID")),
		AccessKeySecret: tea.String(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_SECRET")),
	}
	config.Endpoint = tea.String("rds.aliyuncs.com")

	// 创建客户端
	client, err := rds20140815.NewClient(config)
	if err != nil {
		panic(err)
	}

	// 查询 RDS 实例列表
	request := &rds20140815.DescribeDBInstancesRequest{
		RegionId: tea.String("cn-hangzhou"),
	}

	response, err := client.DescribeDBInstances(request)
	if err != nil {
		panic(err)
	}

	// 处理响应
	for _, instance := range response.Body.Items.DBInstance {
		fmt.Printf("Instance ID: %s, Engine: %s, Status: %s\n",
			*instance.DBInstanceId,
			*instance.Engine,
			*instance.DBInstanceStatus)
	}
}
```

### 5.3 Redis 调用示例

```go
package main

import (
	"fmt"
	"os"

	openapi "github.com/alibabacloud-go/darabonba-openapi/v2/client"
	rkvstore20150101 "github.com/alibabacloud-go/r-kvstore-20150101/v4/client"
	"github.com/alibabacloud-go/tea/tea"
)

func main() {
	config := &openapi.Config{
		AccessKeyId:     tea.String(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_ID")),
		AccessKeySecret: tea.String(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_SECRET")),
	}
	config.Endpoint = tea.String("r-kvstore.aliyuncs.com")

	client, err := rkvstore20150101.NewClient(config)
	if err != nil {
		panic(err)
	}

	// 查询 Redis 实例列表
	request := &rkvstore20150101.DescribeInstancesRequest{
		RegionId: tea.String("cn-hangzhou"),
	}

	response, err := client.DescribeInstances(request)
	if err != nil {
		panic(err)
	}

	for _, instance := range response.Body.Instances.KVStoreInstance {
		fmt.Printf("Instance ID: %s, Instance Name: %s, Instance Status: %s\n",
			*instance.InstanceId,
			*instance.InstanceName,
			*instance.InstanceStatus)
	}
}
```

### 5.4 SLS 调用示例

```go
package main

import (
	"fmt"
	"os"

	openapi "github.com/alibabacloud-go/darabonba-openapi/v2/client"
	sls20201230 "github.com/alibabacloud-go/sls-20201230/v3/client"
	"github.com/alibabacloud-go/tea/tea"
)

func main() {
	config := &openapi.Config{
		AccessKeyId:     tea.String(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_ID")),
		AccessKeySecret: tea.String(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_SECRET")),
	}
	config.Endpoint = tea.String("cn-hangzhou.log.aliyuncs.com")

	client, err := sls20201230.NewClient(config)
	if err != nil {
		panic(err)
	}

	// 列出 Project
	listProjectRequest := &sls20201230.ListProjectRequest{}
	response, err := client.ListProject(listProjectRequest)
	if err != nil {
		panic(err)
	}

	for _, project := range response.Body.Projects {
		fmt.Printf("Project Name: %s, Description: %s\n",
			*project.ProjectName,
			*project.Description)
	}
}
```

### 5.5 泛化调用示例（通用方式）

```go
package main

import (
	"fmt"
	"os"

	openapi "github.com/alibabacloud-go/darabonba-openapi/v2/client"
	openapiutil "github.com/alibabacloud-go/openapi-util/service"
	util "github.com/alibabacloud-go/tea-utils/v2/service"
	"github.com/alibabacloud-go/tea/tea"
)

func main() {
	config := &openapi.Config{
		AccessKeyId:     tea.String(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_ID")),
		AccessKeySecret: tea.String(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_SECRET")),
	}
	config.Endpoint = tea.String("rds.aliyuncs.com")

	client, err := openapi.NewClient(config)
	if err != nil {
		panic(err)
	}

	// 设置 API 参数
	params := &openapi.Params{
		Action:      tea.String("DescribeDBInstances"),
		Version:     tea.String("2014-08-15"),
		Protocol:    tea.String("HTTPS"),
		Method:      tea.String("POST"),
		AuthType:    tea.String("AK"),
		Style:       tea.String("RPC"),
		Pathname:    tea.String("/"),
		ReqBodyType: tea.String("json"),
		BodyType:    tea.String("json"),
	}

	query := map[string]interface{}{
		"RegionId": tea.String("cn-hangzhou"),
	}

	request := &openapi.OpenApiRequest{
		Query: openapiutil.Query(query),
	}

	runtime := &util.RuntimeOptions{}
	response, err := client.CallApi(params, request, runtime)
	if err != nil {
		panic(err)
	}

	fmt.Println(response["body"])
}
```

### 5.6 使用 STS Token 认证

```go
package main

import (
	"os"

	openapi "github.com/alibabacloud-go/darabonba-openapi/v2/client"
	"github.com/aliyun/credentials-go/credentials"
	"github.com/alibabacloud-go/tea/tea"
)

func main() {
	// 使用 RamRoleArn 方式，自动管理 STS Token
	credentialsConfig := new(credentials.Config).
		SetType("ram_role_arn").
		SetAccessKeyId(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_ID")).
		SetAccessKeySecret(os.Getenv("ALIBABA_CLOUD_ACCESS_KEY_SECRET")).
		SetRoleArn("acs:ram::1234567890123456:role/MCPRole").
		SetRoleSessionName("mcp-session").
		SetRoleSessionExpiration(3600)

	credentialClient, err := credentials.NewCredential(credentialsConfig)
	if err != nil {
		panic(err)
	}

	config := &openapi.Config{
		Credential: credentialClient,
	}
	config.Endpoint = tea.String("rds.aliyuncs.com")

	// 创建客户端并调用 API
	// ...
}
```

---

## 六、API 限制和注意事项

### 6.1 QPS 限制汇总

| 产品 | 默认 QPS 限制 | 说明 |
|------|--------------|------|
| **RDS** | 100-500 次/秒 | 根据接口不同 |
| **Redis** | 100-200 次/秒 | 查询类接口较高 |
| **RocketMQ** | 100 次/秒 | 消息查询类接口 |
| **SLS** | 100 次/秒 | GetLogs 等查询接口 |
| **MSE** | 100 次/秒 | 配置管理类接口 |
| **ARMS** | 100 次/秒 | 监控查询类接口 |
| **Kafka** | 100 次/秒 | 实例管理类接口 |

> 注意：主账号和所有 RAM 子账号共享同一个 QPS 配额

### 6.2 通用限制

| 限制类型 | 默认值 | 说明 |
|---------|--------|------|
| **单账号 QPS** | 100 次/秒 | 大部分接口默认值 |
| **单次查询返回条数** | 100 条 | 可调整，最大 500 条 |
| **单次查询时间范围** | 24 小时 | SLS 等日志服务 |
| **分页大小** | 100 | 大部分 List 接口 |

### 6.3 计费说明

| 项目 | 是否收费 | 说明 |
|------|---------|------|
| **SDK 使用** | 免费 | SDK 本身免费 |
| **OpenAPI 调用** | 视产品而定 | 大部分管控 API 免费 |
| **资源创建** | 收费 | 按各云产品标准计费 |
| **DataWorks API** | 收费 | 超出免费额度后收费 |
| **API 网关** | 收费 | 按调用次数收费 |

### 6.4 错误处理建议

```go
// 限流错误处理示例
import (
	"strings"
	"time"
)

func callWithRetry(client *rds20140815.Client, request *rds20140815.DescribeDBInstancesRequest) (*rds20140815.DescribeDBInstancesResponse, error) {
	maxRetries := 3
	var response *rds20140815.DescribeDBInstancesResponse
	var err error

	for i := 0; i < maxRetries; i++ {
		response, err = client.DescribeDBInstances(request)
		if err == nil {
			return response, nil
		}

		// 检查是否为限流错误
		if strings.Contains(err.Error(), "Throttling") || strings.Contains(err.Error(), "RateLimit") {
			// 指数退避
			time.Sleep(time.Duration(i+1) * time.Second)
			continue
		}

		return nil, err
	}

	return nil, err
}
```

### 6.5 重要注意事项

1. **RocketMQ 消息收发**: OpenAPI 仅用于管控，消息收发请使用 SDK
2. **SLS 查询限制**: 单次查询最多扫描 1 亿行数据
3. **STS Token 过期**: 使用 RamRoleArn 方式自动刷新
4. **权限最小化**: 为 MCP Server 分配最小必要权限
5. **日志记录**: 建议记录所有 API 调用日志，便于审计

---

## 七、推荐用于 MCP Server 封装的 API 清单

### 7.1 P0 优先级（核心功能）

#### RDS 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| DescribeDBInstances | 查询实例列表 | `rds_list_instances` |
| DescribeDBInstanceAttribute | 查询实例详情 | `rds_get_instance` |
| DescribeDatabases | 查询数据库列表 | `rds_list_databases` |
| DescribeAccounts | 查询账号列表 | `rds_list_accounts` |
| DescribeBackups | 查询备份列表 | `rds_list_backups` |
| DescribeDBInstancePerformance | 查询性能指标 | `rds_get_metrics` |
| DescribeDBInstanceNetInfo | 查询连接信息 | `rds_get_connection` |
| CreateBackup | 创建备份 | `rds_create_backup` |

#### Redis 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| DescribeInstances | 查询实例列表 | `redis_list_instances` |
| DescribeInstanceAttribute | 查询实例详情 | `redis_get_instance` |
| DescribeParameters | 查询参数配置 | `redis_get_parameters` |
| DescribeBackups | 查询备份列表 | `redis_list_backups` |
| CreateBackup | 创建备份 | `redis_create_backup` |

#### RocketMQ 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| OnsInstanceInServiceList | 查询实例列表 | `rocketmq_list_instances` |
| OnsTopicList | 查询 Topic 列表 | `rocketmq_list_topics` |
| OnsGroupList | 查询 Group 列表 | `rocketmq_list_groups` |
| OnsConsumerStatus | 查询消费状态 | `rocketmq_get_consumer_status` |
| OnsMessageGetByMsgId | 按 ID 查询消息 | `rocketmq_get_message_by_id` |
| OnsMessageGetByKey | 按 Key 查询消息 | `rocketmq_get_message_by_key` |
| OnsMessageTrace | 查询消息轨迹 | `rocketmq_get_message_trace` |

#### SLS 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| ListProject | 列出 Project | `sls_list_projects` |
| GetProject | 获取 Project | `sls_get_project` |
| ListLogStores | 列出 Logstore | `sls_list_logstores` |
| GetLogStore | 获取 Logstore | `sls_get_logstore` |
| GetLogsV2 | 查询日志 | `sls_query_logs` |
| GetHistograms | 查询日志分布 | `sls_get_histograms` |
| ListConsumerGroup | 列出消费组 | `sls_list_consumer_groups` |

### 7.2 P1 优先级（扩展功能）

#### MSE 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| ListNamespaces | 查询命名空间 | `mse_list_namespaces` |
| ListNacosConfigs | 查询 Nacos 配置 | `mse_list_nacos_configs` |
| ListAnsServices | 查询 Nacos 服务 | `mse_list_nacos_services` |
| ListAnsInstances | 查询 Nacos 实例 | `mse_list_nacos_instances` |

#### ARMS 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| ListTraceApps | 列出应用监控 | `arms_list_applications` |
| SearchTracesByPage | 查询调用链 | `arms_search_traces` |
| QueryAppTopology | 查询应用拓扑 | `arms_get_topology` |
| GetAlertRules | 查询告警规则 | `arms_list_alert_rules` |
| ListPrometheusAlertRules | 查询 Prometheus 告警 | `arms_list_prometheus_alerts` |

#### Kafka 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| GetInstanceList | 查询实例列表 | `kafka_list_instances` |
| GetTopicList | 查询 Topic 列表 | `kafka_list_topics` |
| GetConsumerList | 查询消费组 | `kafka_list_consumers` |

### 7.3 P2 优先级（可选功能）

#### SchedulerX 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| ListJobs | 查询任务列表 | `schedulerx_list_jobs` |
| GetJobInstanceList | 查询任务执行历史 | `schedulerx_list_job_instances` |
| ListWorkflows | 查询工作流列表 | `schedulerx_list_workflows` |

#### MNS 推荐 API

| API 名称 | 功能 | MCP Tool 名称 |
|---------|------|--------------|
| ListQueue | 查询队列列表 | `mns_list_queues` |
| ListTopic | 查询主题列表 | `mns_list_topics` |

### 7.4 MCP Server 架构建议

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP Server (Go)                          │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ RDS Tool │ │Redis Tool│ │RocketMQ  │ │ SLS Tool │       │
│  │          │ │          │ │  Tool    │ │          │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘       │
│       │            │            │            │              │
│  ┌────┴────────────┴────────────┴────────────┴─────┐       │
│  │           Alibaba Cloud SDK (Go)                │       │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐           │       │
│  │  │ RDS SDK │ │Redis SDK│ │ SLS SDK │  ...      │       │
│  │  └─────────┘ └─────────┘ └─────────┘           │       │
│  └─────────────────────────────────────────────────┘       │
│                           │                                 │
│  ┌────────────────────────┴─────────────────────────┐       │
│  │              Authentication Layer                 │       │
│  │  (AK/SK, STS Token, RamRoleArn)                  │       │
│  └───────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### 7.5 实现建议

1. **统一认证管理**: 使用 RamRoleArn 方式，自动刷新 STS Token
2. **错误处理**: 实现指数退避重试机制，处理限流错误
3. **缓存策略**: 对查询类 API 结果进行适当缓存
4. **权限控制**: 为每个 Tool 配置最小必要权限
5. **日志记录**: 记录所有 API 调用，便于审计和排查
6. **并发控制**: 实现客户端级别的 QPS 限制，避免触发服务端流控

---

## 附录：参考链接

### 官方文档

| 产品 | OpenAPI 门户 | 帮助文档 |
|------|-------------|---------|
| RDS | https://api.aliyun.com/product/Rds | https://help.aliyun.com/zh/rds |
| Redis | https://next.api.aliyun.com/document/R-kvstore | https://help.aliyun.com/zh/redis |
| RocketMQ | https://api.aliyun.com/api/Ons/2019-02-14 | https://help.aliyun.com/zh/apsaramq-for-rocketmq |
| SLS | https://api.aliyun.com/document/Sls/2020-12-30 | https://help.aliyun.com/zh/sls |
| MSE | https://api.aliyun.com/document/mse | https://help.aliyun.com/zh/mse |
| ARMS | https://help.aliyun.com/zh/arms/developer-reference/api-reference | https://help.aliyun.com/zh/arms |
| Kafka | https://next.api.aliyun.com/document/alikafka | https://help.aliyun.com/zh/alikafka |
| SchedulerX | https://next.api.aliyun.com/document/schedulerx2 | https://help.aliyun.com/zh/schedulerx |

### SDK 文档

- Go SDK 集成指南：https://www.alibabacloud.com/help/zh/sdk/developer-reference/v2-go-integrated-sdk
- 泛化调用文档：https://www.alibabacloud.com/help/zh/sdk/developer-reference/generalized-call-go
- 凭证管理文档：https://help.aliyun.com/zh/sdk/developer-reference/v2-manage-go-access-credentials

---

*报告生成时间：2026-04-09*
