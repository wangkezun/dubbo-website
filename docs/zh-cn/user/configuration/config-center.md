# 动态配置中心

配置中心（v2.7.0）在Dubbo中承担两个职责：

1. 外部化配置。启动配置的集中式存储 （简单理解为dubbo.properties的外部化存储）。
2. 服务治理。服务治理规则的存储与通知。



启用动态配置（以Zookeeper为例，可查看[动态配置配置项详解](../references/xml/dubbo-config-center.md)）：

```xml
<dubbo:config-center address="zookeeper://127.0.0.1:2181"/>
```

或者

```properties
dubbo.config-center.address=zookeeper://127.0.0.1:2181
```

或者

```java
ConfigCenterConfig configCenter = new ConfigCenterConfig();
configCenter.setAddress("zookeeper://127.0.0.1:2181");
```

> 为了兼容2.6.x版本配置，在使用Zookeeper作为注册中心，且没有显示配置配置中心的情况下，Dubbo框架会默认将此Zookeeper用作配置中心，但将只作服务治理用途。


## 外部化配置

外部化配置目的之一是实现配置的集中式管理，这部分业界已经有很多成熟的专业配置系统如Apollo, Nacos等，Dubbo所做的主要是保证能配合这些系统正常工作。

外部化配置和其他本地配置在内容和格式上并无区别，可以简单理解为`dubbo.properties`的外部化存储，配置中心更适合将一些公共配置如注册中心、元数据中心配置等抽取以便做集中管理。

```properties
# 将注册中心地址、元数据中心地址等配置集中管理，可以做到统一环境、减少开发侧感知。
dubbo.registry.address=zookeeper://127.0.0.1:2181
dubbo.registry.simplified=true

dubbo.metadata-report.address=zookeeper://127.0.0.1:2181

dubbo.protocol.name=dubbo
dubbo.protocol.port=20880

dubbo.application.qos.port=33333
```


- 优先级

外部化配置默认较本地配置有更高的优先级，因此这里配置的内容会覆盖本地配置值，关于[各配置形式间的覆盖关系](./configuration-load-process.md)有单独一章说明，你也可通过以下选项调整配置中心的优先级：

  ```properties
  -Ddubbo.config-center.highest-priority=false
  ```

- 作用域

外部化配置有全局和应用两个级别，全局配置是所有应用共享的，应用级配置是由每个应用自己维护且只对自身可见的。


当前已支持的扩展实现有Zookeeper、Apollo。


#### Zookeeper

```xml
<dubbo:config-center address="zookeeper://127.0.0.1:2181"/>
```



默认所有的配置都存储在`/dubbo/config`节点，具体节点结构图如下：

![image-20190127225608553](/img/zk-configcenter.jpg)

- namespace，用于不同配置的环境隔离。
- config，Dubbo约定的固定节点，不可更改，所有配置和服务治理规则都存储在此节点下。
- dubbo/application，分别用来隔离全局配置、应用级别配置：dubbo是默认group值，application对应应用名
- dubbo.properties，此节点的node value存储具体配置内容



#### Apollo

```xml
<dubbo:config-center protocol="apollo" address="127.0.0.1:2181"/>
```

Apollo中的一个核心概念是命名空间 - namespace（和上面zookeeper的namespace概念不同），在这里全局和应用级别配置就是通过命名空间来区分的。

默认情况下，Dubbo会从名叫`dubbo-properties`（由于 Apollo 不支持特殊后缀 `.properties` ）的命名空间中读取全局配置（`<dubbo:config-center namespace="your namespace">`）

![image-20190128095444169](/img/apollo-configcenter-dubbo.png)


> 这里相当于是把 `dubbo.properties` 配置文件的内容存储在了 Apollo 中，应用通过关联共享的 `dubbo-properties` namespace 继承公共配置,
> 应用也可以按照 Apollo 的做法来覆盖个别配置项。


#### 自己加载外部化配置

所谓Dubbo对配置中心的支持，本质上就是把`.properties`从远程拉取到本地，然后和本地的配置做一次融合。理论上只要Dubbo框架能拿到需要的配置就可以正常的启动，它并不关心这些配置是自己加载到的还是应用直接塞给它的，所以Dubbo还提供了以下API，让用户将自己组织好的配置塞给Dubbo框架（配置加载的过程是用户要完成的），这样Dubbo框架就不再直接和Apollo或Zookeeper做读取配置交互。

```java
// 应用自行加载配置
Map<String, String> dubboConfigurations = new HashMap<>();
dubboConfigurations.put("dubbo.registry.address", "zookeeper://127.0.0.1:2181");
dubboConfigurations.put("dubbo.registry.simplified", "true");

//将组织好的配置塞给Dubbo框架
ConfigCenterConfig configCenter = new ConfigCenterConfig();
configCenter.setExternalConfig(dubboConfigurations);
```



## 服务治理

#### Zookeeper

默认节点结构：

![image-20190128101129591](/img/zk-configcenter-governance.jpg)

- namespace，用于不同配置的环境隔离。
- config，Dubbo约定的固定节点，不可更改，所有配置和服务治理规则都存储在此节点下。
- dubbo，所有服务治理规则都是全局性的，dubbo为默认节点
- configurators/tag-router/condition-router，不同的服务治理规则类型，node value存储具体规则内容



#### Apollo

所有的服务治理规则都是全局性的，默认从公共命名空间`dubbo`读取和订阅：

![image-20190128100600055](/img/apollo-configcenter-governance.jpg)

不同的规则以不同的key后缀区分：

- configurators，[覆盖规则](../demos/config-rule.md)
- tag-router，[标签路由](../demos/routing-rule.md)
- condition-router，[条件路由](../demos/routing-rule.md)