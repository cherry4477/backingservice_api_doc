
## 列出服务目录
服务目录是一个适配器必须实现的第一个接入点。在初始化时，中信云平台将从所有适配器中读取供应商及其服务信息，并调整用户看到的服务目录。如果服务目录没法初始化或无效，中信云平台将不允许运营管理员注册这一适配器，并将给出具体原因。当适配器更新时，平台将更新响应的服务目录，管理员可以通过命令行中的 update-suppliers-adapter 更新供应商和服务信息。 平台从一个适配器获得服务目录信息后，会用自己存储的供应商和服务的ID unique_id 与适配器中的供应商和服务ID进行比较。如果其中任何一个不存在，则将增加新纪录。对于存在的供应商和服务ID，平台会更新对应服务目录。 如果平台里包含此适配器中没有的服务，并且这个服务没有服务实例，平台将删除此服务，如果服务有服务实例，则此服务会被标识为对所有人不可见。如果供应商不在存在服务，平台将删除此供应商。

### 客户端请求
```
GET /v2/catalog
```

### 服务器回应
```
services:创建对象数组，下面定义的服务对象的架构
   [0].name:服务名称将显示在目录中,全部小写，不能空格，唯一
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
示例：
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
## 创建服务实例
适配器接收到此接口点调用时，应该按照指定参数创建服务实例。 提示：服务实例的 instance_id 由中信云平台给提供。这个ID会用来对此服务实例进行后继操作，所以适配器必须将此ID与具体的服务实例相关联。
### 客户端请求
```
PUT /v2/service_instances/:instance_id
```
解释：
```
organization_guid：云控制器 GUID 的组织下，该服务是创建服务实例。虽然多数代理人将不使用此字段，有助于确定数据放置或应用自定义的业务规则。
plan_id:计划在上述服务的ID(从目录端点),用户愿意提供。因为计划有标识符唯一的代理,这是提供足够的信息来确定
service_id:在上面的目录服务的ID
space_guid:与organization_guid相似,但对于空间而言
parameters:云计算API客户端可以与他们的要求提供一个JSON对象的配置参数,这个值将通过服务代理,代理人负责验证。
accepts_incomplete:真实值表明,云控制器和请求客户机支持异步供应。如果这个参数是不包括在请求,和代理只能提供异步请求的计划的一个实例,代理应该拒绝请求422如下所述
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
```
### 服务端响应
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
示例：
```
{
 "dashboard_url": "http://example-dashboard.example.com/9189kdfsk0vfnku",
 "operation": "task_10"
}
```
   

     




