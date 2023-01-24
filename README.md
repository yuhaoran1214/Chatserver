# C++实现集群聊天服务器

## 一、技术栈

- Json序列化和反序列化
- muduo网络库开发
- nginx源码编译安装和环境部署
- nginx的tcp负载均衡器配置
- redis缓存服务器编程实践
- 基于发布-订阅的服务器中间件redis消息队列编程实践
- MySQL数据库编程CMake构建编译环境

## 二、项目功能

1.  客户端新用户注册
2.  客户端用户登录
3.  添加好友和添加群组
4.  好友聊天
5. 群组聊天
6. 离线消息
7. nginx配置tcp负载均衡
8. 集群聊天系统支持客户端跨服务器通信

## 三、Json数据序列化

就是把我们想要打包的数据，或者对象，直接处理成Json字符串。

#### 1.普通数据序列化

```C++
json js;
// 添加数组
js["id"] = {1,2,3,4,5};
// 添加key-value
js["name"] = "zhang san";
// 添加对象
js["msg"]["zhang san"] = "hello world";
js["msg"]["liu shuo"] = "hello china";
// 上面等同于下面这句一次性添加数组对象
js["msg"] = {{"zhang san", "hello world"}, {"liu shuo", "hello china"}};
cout << js << endl;
```

序列化结果是：

```
{"id":[1,2,3,4,5],msg":{"liu shuo":"hello china","zhang san":"hello world"},"name":"zhang san"}
```

#### 2.容器序列化

```C++
json js;
// 直接序列化一个vector容器
vector<int> vec;
vec.push_back(1);
vec.push_back(2);
vec.push_back(5);
js["list"] = vec;
// 直接序列化一个map容器
map<int, string> m;
m.insert({1, "黄山"});
m.insert({2, "华山"});
m.insert({3, "泰山"});
js["path"] = m;
cout<<js<<endl;
```

把C++ STL中的容器内容可以直接序列化成Json字符串，上面代码打印如下：

```
{"list":[1,2,5],"path":[[1,"黄山"],[2,"华山"],[3,"泰山"]]}
```

## 四、Json数据反序列化

当从网络接收到字符串为Json格式，可以用JSON for Modern C++ 直接反序列化取得数据或者直接反序列化出对象，甚至容器。

```C++
string jsonstr = js.dump();
cout<<"jsonstr:"<<jsonstr<<endl;
// 模拟从网络接收到json字符串，通过json::parse函数把json字符串专程json对象
json js2 = json::parse(jsonstr);
// 直接取key-value
string name = js2["name"];
cout << "name:" << name << endl;
// 直接反序列化vector容器
vector<int> v = js2["list"];
for(int val : v)
{
cout << val << " ";
}
cout << endl;
// 直接反序列化map容器
map<int, string> m2 = js2["path"];
for(auto p : m2)
{
cout << p.first << " " << p.second << endl;
}
cout << endl;
```

# 服务器集群

## 五、负载均衡-一致性哈希算法

单台服务器受限于硬件资源，其性能是有上限的，当单台服务器不能满足应用场景的并发需求量时，就需要考虑部署多个服务器共同处理客户端的并发请求，但是客户端怎么知道去连接具体哪台服务器呢？

此时就需要一台负载均衡器，通过预设的负载算法，指导客户端连接服务器。负载均衡器有基于客户端的负载均衡和服务器的负载均衡。普通的基于哈希的负载算法，并不能满足负载均衡所要求的单调性和平衡性，但一致性哈希算法非常好的保持了这两种特性，所以经常用在需要设计负载算法的应用场景当中。

## 六、集群服务器之间的通信设计

集群部署的服务器之间进行通信，最好的方式就是引入中间件消息队列，解耦各个服务器，使整个系统松耦合，提高服务器的响应能力，节省服务器的带宽资源。

在集群分布式环境中，经常使用的中间件消息队列有ActiveMQ、RabbitMQ、Kafka等，都是应用场景广泛并且性能很好的消息队列，供集群服务器之间，分布式服务之间进行消息通信。限于我们的项目业务类型并不是非常复杂，对并发请求量也没有太高的要求，因此中间件消息队列选型的是-基于发布-订阅模式的redis。

## 七、数据库设计

### 表设计

**User表**

| 字段名称 | 字段类型                   | 字段说明 | 约束                        |
| -------- | -------------------------- | -------- | --------------------------- |
| id       | INT                        | 用户id   | PRIMARY KEY、AUTO_INCREMENT |
| name     | VARCHAR(50)                | 用户名   | NOT NULL, UNIQUE            |
| password | VARCHAR(50)                | 用户密码 | NOT NULL                    |
| state    | ENUM('online','offerline') | 登录状态 | DEFAULT 'offline'           |

**Friend表**

| 字段名称  | 字段类型 | 字段说明 | 约束               |
| --------- | -------- | -------- | ------------------ |
| user id   | INT      | 用户 id  | NOT NULL、联合主键 |
| friend id | INT      | 好友 id  | NOT NULL、联合主键 |

**AllGroup表**

| 字段名称  | 字段类型     | 字段说明   | 约束                        |
| --------- | ------------ | ---------- | --------------------------- |
| id        | INT          | 组id       | PRIMARY KEY、AUTO_INCREMENT |
| groupname | VARCHAR(50)  | 组名称     | NOT NULL,UNIQUE             |
| groupdesc | VARCHAR(200) | 组功能描述 | DEFAULT ''                  |

**GroupUser表**

| 字段名称  | 字段类型                  | 字段说明 | 约束               |
| --------- | ------------------------- | -------- | ------------------ |
| group id  | INT                       | 组id     | NOT NULL、联合主键 |
| user id   | INT                       | 组员id   | NOT NULL、联合主键 |
| grouprole | ENUM('creator', 'normal') | 组内角色 | DEFAULT ‘normal’   |

**OfflineMessage表**

| 字段名称 | 字段类型     | 字段说明                   | 约束     |
| -------- | ------------ | -------------------------- | -------- |
| user id  | INT          | 用户id                     | NOT NULL |
| message  | VARCHAR(500) | 离线消息（存储Json字符串） | NOT NULL |


