## EXP Introduction

Extension Plugin 扩展点插件系统


名词定义:

1. 主应用
    - exp 需要运行在一个 jvm 之上, 通常, 这是一个 springboot, 这个 springboot 就是主应用;
2. 扩展点
    - 主应用定义的接口, 可被插件实现;
    - 注意：插件是扩展点的具体实现集合，扩展点，仅仅是接口定义。一个插件里，可以有多个扩展点的实现，一个扩展点，可以有多个插件的实现。
3. 插件
    - 扩展功能使用插件的方式支持，你可以理解为 idea、eclipse 里的插件。
    - 插件里的代码写法和 spring 一样（如果你的程序是在 spring 里运行）
4. 热插拔
    - 插件支持从 jvm 和 spring 容器里摘除.
    - 支持运行时动态安装 jar 和 zip;

## 举例

- 贵州茅台和五粮液都购买了你司的标准产品, 但是. 由于客户有定制需求. 需要开发新功能.

- 贵州茅台客户定制了 2 个插件;
- 五粮液客户定制了 3 个插件;
- 程序运行时, 会根据客户的租户 id 进行逻辑切换.



![](desc.png)


场景:

1. B 端大客户对业务进行定制, 需要对主代码扩展.
    - 传统做法是 git 拉取分支.
    - 现在基于扩展点的方式进行定制, 可热插拔
2. 多个程序可分可合, 支持将多个 springboot 应用合并部署, 或拆开部署.
3. 支持扩展点类似 swagger 文档 doc, 用于类插件系统管理平台进行展示.

## Feature

1. 支持 热插拔 or 启动时加载
2. 基于 classloader 双亲委派的类隔离机制
3. 支持多租户场景下的单个扩展点有多实现, 业务支持租户过滤, 租户多个实现可自定义排序
4. 支持 springboot2.x/1.x 依赖
5. 支持插件内对外暴露 Spring Controller Rest, 可热插拔;
6. 支持插件获取独有的配置, 支持自定义设计插件配置热更新逻辑;
7. 支持插件和主应用绑定事务.

## USE

环境准备:

1. JDK 1.8
2. Maven

```shell
git clone git@github.com:stateIs0/exp.git
cd all-package
mvn clean package
```

主程序依赖(springboot starter)

```xml

<dependency>
    <groupId>cn.think.in.java</groupId>
    <!-- 这里是 springboot 2 例子, 如果是普通应用或者 springboot 1 应用, 请进行 artifactId 更换  -->
    <artifactId>open-exp-adapter-springboot2-starter</artifactId>
</dependency>
```

插件依赖

```xml
<dependency>
   <groupId>cn.think.in.java</groupId>
   <artifactId>open-exp-plugin-depend</artifactId>
</dependency>
```

## 编程界面 API 使用

```java
@RequestMapping("/run")
public String run(String tenantId) {
  // 上下文设置租户 id
  context.set(tenantId);
  try {
      List<UserService> userServices = expAppContext.get(UserService.class);
      // first 第一个就是这个租户优先级最高的.
      Optional<UserService> optional = userServices.stream().findFirst();
      if (optional.isPresent()) {
          optional.get().createUserExt();
      } else {
          return "not found";
      }
      return "success";
  } finally {
      // 上下文删除租户 id
      context.remove();
  }
}


@RequestMapping("/install")
public String install(String path, String tenantId) throws Throwable {
  Plugin plugin = expAppContext.load(new File(path));

  sortMap.put(plugin.getPluginId(), Math.abs(new Random().nextInt(100)));
  pluginIdTenantIdMap.put(plugin.getPluginId(), tenantId);

  return plugin.getPluginId();
}

@RequestMapping("/unInstall")
public String unInstall(String pluginId) throws Exception {
  log.info("plugin id {}", pluginId);
  expAppContext.unload(pluginId);
  pluginIdTenantIdMap.remove(pluginId);
  sortMap.remove(pluginId);
  return "ok";
}
```

## 模块

1. [all-package](all-package) 打包模块
2. [bom-manager](bom-manager) pom 管理, 自身管理和三方依赖管理
    - [exp-one-bom](bom-manager%2Fexp-one-bom) 自身包管理
    - [exp-third-bom](bom-manager%2Fexp-third-bom) 三方包管理
3. [open-exp-code](open-exp-code) exp 核心代码
    - [open-exp-classloader-container](open-exp-code%2Fopen-exp-classloader-container) classloader 隔离 API
    - [open-exp-classloader-container-impl](open-exp-code%2Fopen-exp-classloader-container-impl) classloader 隔离 API 具体实现
    - [open-exp-client-api](open-exp-code%2Fopen-exp-client-api) 核心 api 模块
    - [open-exp-core-impl](open-exp-code%2Fopen-exp-core-impl) 核心 api 实现; 内部 shade cglib 动态代理, 可不以来 spring 实现;
    - [open-exp-document-api](open-exp-code%2Fopen-exp-document-api) 扩展点文档 api
    - [open-exp-document-core-impl](open-exp-code%2Fopen-exp-document-core-impl) 扩展点文档导出实现
    - [open-exp-plugin-depend](open-exp-code%2Fopen-exp-plugin-depend) exp 插件依赖
4. [example](example) exp 使用示例代码
    - [example-extension-define](example%2Fexample-extension-define) 示例扩展点定义
    - [example-plugin1](example%2Fexample-plugin1) 示例插件实现 1
    - [example-plugin2](example%2Fexample-plugin2) 示例插件实现 2
    - [example-springboot1](example%2Fexample-springboot1) 示例 springboot 1.x 例子
    - [example-springboot2](example%2Fexample-springboot2) 示例 springboot 2.x 例子; 使用 spring cglib 动态代理
5. [spring-adapter](spring-adapter) springboot starter, exp 适配 spring boot
    - [open-exp-adapter-springboot2](spring-adapter%2Fopen-exp-adapter-springboot2-starter)  springboot2 依赖
    - [open-exp-adapter-springboot1-starter](spring-adapter%2Fopen-exp-adapter-springboot1-starter) springboot1 依赖

## 模块依赖

![](ar.png)

## 核心 API

```java
public interface ExpAppContext {

    /**
     * 加载插件
     */
    Plugin load(File file) throws Throwable;

    /**
     * 卸载插件
     */
    void unload(String id) throws Exception;

    /**
     * 获取多个扩展点的插件实例
     */
    <P> List<P> get(String extCode);

    /**
     * 简化操作, code 就是全路径类名
     */
    <P> List<P> get(Class<P> pClass);

    /**
     * 获取单个插件实例.
     */
    <P> P get(String extCode, String pluginId);

    /**
     * 获取 TenantCallback 扩展逻辑;
     */
    default TenantCallback getTenantCallback() {
        return TenantCallback.TenantCallbackMock.instance;
    }

    /**
     * 设置 callback;
     */
    default void setTenantCallback(TenantCallback callback) {
    }
}
```

## 扩展

cn.think.in.java.open.exp.client.TenantCallback

```java
public interface TenantCallback {

   /**
    * 返回这个插件的序号, 默认 0; 
    * {@link  cn.think.in.java.open.exp.client.ExpAppContext#get(java.lang.Class)} 函数返回的List 的第一位就是 sort 最高的.
    */
   Integer getSort(String pluginId);

   /**
    * 这个插件是否属于当前租户, 默认是;
    * 这个返回值, 会影响 {@link  cn.think.in.java.open.exp.client.ExpAppContext#get(java.lang.Class)} 的结果
    * 即进行过滤, 返回为 true 的 plugin 实现, 才会被返回.
    */
   Boolean isOwnCurrentTenant(String pluginId);
}
```

租户过滤示例代码:

````java
expAppContext.setTenantCallback(new TenantCallback() {
   @Override
   public Integer getSort(String pluginId) {
       // 获取这个插件的排序
       return sortMap.get(pluginId);
   }

   @Override
   public Boolean isOwnCurrentTenant(String pluginId) {
       // 判断当前租户是不是这个匹配这个插件
       return context.get().equals(pluginIdTenantIdMap.get(pluginId));
   }
});
````

插件获取配置示例代码:
```java
public class Boot extends AbstractBoot {
    private static String selfPluginId;

    @Override
    protected String getScanPath() {
        return Boot.class.getPackage().getName();
    }

    @Override
    public void setPluginId(String pluginId) {
        // 系统自动注入自身的插件 id;
        selfPluginId = pluginId;
    }

    public static String get(String key, String value) {
        //  简化操作, 读取配置
        return PluginConfig.getSpi().getProperty(selfPluginId, key, value);
    }

}
```


