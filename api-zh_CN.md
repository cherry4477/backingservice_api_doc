
(英文文档: http://docs.cloudfoundry.org/services/api.html)

# 服务提供中间商（Service Broker）API 中文文档

以下"服务提供中间商"简称为“服务提供方“或者“提供方“。

## 服务列表（Catalog）API

服务提供方必须提供一个服务列表API，以供云平台能够读取和解析该提供方所提供的所有服务（Service）和套餐（Plan）信息，并汇总整理一份云平台用户可以看到的总的服务目录。每个服务应包含若干套餐。服务列表中的服务和套餐信息由它们的`id`字段标识。

云平台将定期更新调用此API以各个提供方的服务信息。云平台从一个提供方获得其服务列表后，将会和保存在云平台服务器上的旧的服务和套餐列表信息进行对比。
* 如果新的服务列表中的某个服务或者套餐不存在于云平台维护的总的服务目录中，此服务或者套餐将被加入。
* 如果云平台维护的总的服务目录中包含新的服务列表中不存在套餐，
  * 如果不存在套餐没有关联的服务实例，则此套餐将被从云平台维护的总的服务目录中删除。
  * 如果不存在套餐有关联的服务实例，则此套餐将被标识为`inactive`并将对平台用户不可见。
* 云平台维护的总的服务目录中不包含套餐的服务将被从总的服务目录中删除。

### 请求URI

```
GET /v2/catalog
```

### 提供方服务器回应

```
200 OK
```

返回示例：
```
{
  "services": [{
    "name": "fake-service",
    "id": "acb56d7c-XXXX-XXXX-XXXX-feb140a59a66",
    "description": "fake service",
    "tags": ["no-sql", "relational"],
    "requires": ["route_forwarding"],
    "max_db_per_node": 5,
    "bindable": true,
    "metadata": {
      "provider": {
        "name": "The name"
      },
      "listing": {
        "imageUrl": "http://example.com/cat.gif",
        "blurb": "Add a blurb here",
        "longDescription": "A long time ago, in a galaxy far far away..."
      },
      "displayName": "The Fake Broker"
    },
    "dashboard_client": {
      "id": "398e2f8e-XXXX-XXXX-XXXX-19a71ecbcf64",
      "secret": "277cabb0-XXXX-XXXX-XXXX-7822c0a90e5d",
      "redirect_uri": "http://localhost:1234"
    },
    "plan_updateable": true,
    "plans": [{
      "name": "fake-plan",
      "id": "d3031751-XXXX-XXXX-XXXX-a42377d3320e",
      "description": "Shared fake Server, 5tb persistent disk, 40 max concurrent connections",
      "max_storage_tb": 5,
      "metadata": {
        "cost": 0,
        "bullets": [{
          "content": "Shared fake server"
        }, {
          "content": "5 TB storage"
        }, {
          "content": "40 concurrent connections"
        }]
      }
    }, {
      "name": "fake-async-plan",
      "id": "0f4008b5-XXXX-XXXX-XXXX-dace631cd648",
      "description": "Shared fake Server, 5tb persistent disk, 40 max concurrent connections. 100 async",
      "max_storage_tb": 5,
      "metadata": {
        "cost": 0,
        "bullets": [{
          "content": "40 concurrent connections"
        }]
      }
    }, {
      "name": "fake-async-only-plan",
      "id": "8d415f6a-XXXX-XXXX-XXXX-e61f3baa1c77",
      "description": "Shared fake Server, 5tb persistent disk, 40 max concurrent connections. 100 async",
      "max_storage_tb": 5,
      "metadata": {
        "cost": 0,
        "bullets": [{
          "content": "40 concurrent connections"
        }]
      }
    }]
  }]
} 
```
返回解释：
```
services:创建对象数组，下面定义的服务对象的架构
   [0].name:服务名称将显示在目录中,全部小写，不能空格，唯一
   [0].id:用于关联这一服务在以后对该目录的请求标识符,在云计算中必须是唯一，推荐使用 GUID
   [0].description:显示在目录的服务简短描述
   [0].tags:标记提供了灵活的机制来公开分类
   [0].requires:用户必须提供的权限列表,目前支持的唯一权限是 syslog_drain、 route_forwarding 和 volume_mount,更多的信息请参阅应用程序日志流和路由服务卷服务。
   [0].bindable:判断服务是否可以绑定应用
   [0].metadata:服务元数据列表，更多的信息，请参阅服务元数据
   [0].dashboard_client:包含必要的数据来激活此服务的仪表板 SSO 功能
      .id:服务打算使用 Oauth2 客户端的 id。名称可能采取的在这种情况下的解调将返回一个错误给操作员
      .secret:秘密的仪表板客户端
      .redirect_uri:服务指示板，将白名单由 UAA 以启用 SSO 域
   [0].plan_updateable:是否该服务支持一些计划升级/降级。请注意，属性 plan_updatable 到 plan_updateable 的拼写错误做错了。我们选择继续，而不是修复它，从而打破了向后兼容性。
   [0].plans:服务计划列表，架构定义如下
      [0].id:用于关联这一计划在以后对该目录的请求标识符,在云计算中必须是唯一，推荐使用 GUID
      [0].name:计划的名称将显示在目录中,全部小写的形式,不能有空格
      [0].description:显示在目录中对服务的简短描述
      [0].metadata:元数据服务计划的列表。更多的信息，请参阅服务元数据
      [0].free:此字段允许由云铸造配额中的 non_basic_services_allowed 字段限制，请参阅配额计划的计划。默认值︰ true
```

## 创建服务实例API

服务提供方服务器接收到云平台对此接口点的调用时，应该按照指定参数创建一个服务实例。 

### 客户端请求
```
PUT /v2/service_instances/:instance_id
```
解释：
```
organization_guid：云平台提供的服务消费者的GUID。一般来说，服务提供方并不需要关心此字段。
plan_id:套餐的标识符唯一ID,用户愿意提供。因为计划有标识符唯一的代理,这是提供足够的信息来确定
service_id:服务的标识符唯一ID
space_guid:与organization_guid相似。一般来说，服务提供方并不需要关心此字段。
parameters:云平台客户端提供一个JSON格式的配置参数。
accepts_incomplete:如果此值为true，表明云平台支持服务提供方异步创建服务实例。此参数为可选参数，默认值为false。如果此值为false并且服务提供方不支持异步生成一个实例，则此API应该返回一个422拒绝请求。
```
示例：
```
{
  "organization_guid": "org-guid-here",
  "plan_id":           "plan-guid-here",
  "service_id":        "service-guid-here",
  "space_guid":        "space-guid-here",
  "parameters":        {
    "parameter1": 1,
    "parameter2": "value"
  }
}
```
### 提供方服务端响应
```
201 Created:创建服务实例,预期响应实体在下面
   dashboard_url:URL的一个基于web的用户界面服务实例管理;我们称之为服务指示板。仪表板的URL应该包含足够的信息来识别被访问的资源(“9189 kdfsk0vfnku”下面的例子),为信息用户可以通过SSO身份验证服务指示板,看到仪表盘的单点登录
   operation:对于异步响应,服务代理可以作为字符串返回操作状态。这个领域将提供给服务代理last_operation请求作为URL编码查询参数
200 OK:可能返回如果服务实例已经存在和请求的参数是相同的现有服务实例,预期响应实体在下面
   dashboard_url:URL的一个基于web的用户界面服务实例管理;我们称之为服务指示板。仪表板的URL应该包含足够的信息来识别被访问的资源(“9189 kdfsk0vfnku”下面的例子),为信息用户可以通过SSO身份验证服务指示板,看到仪表盘的单点登录
   operation:对于异步响应,服务代理可以作为字符串返回操作状态。这个领域将提供给服务代理last_operation请求作为URL编码查询参数
202 Accepted:服务实例创建过程中,这触发云控制器轮询服务实例的最后操作终端操作状态
409 Conflict:应该返回所请求的服务实例是否已经存在,预期的响应体{ }
422 Unprocessable Entity:应该返回如果代理只支持异步请求的计划,供应和请求不包括? accepts_incomplete = true。预期响应体是:{“错误”:“AsyncRequired”、“描述”:“这服务计划需要客户端支持异步服务操作。”},如下所述
```

返回示例：
```
{
 "dashboard_url": "http://example-dashboard.example.com/9189kdfsk0vfnku",
 "operation": "task_10"
}
```
返回解释：
```
dashboard_url: 服务实例仪表盘url。某些服务实例可能没有仪表盘。
```
   

     





