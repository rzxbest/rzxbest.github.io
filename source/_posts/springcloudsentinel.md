---
title: sentinel
date: 2021-11-25 19:20:47
tags: [Springcloud]
categories: [Springcloud]

---
# sentinel
## 添加依赖
- 在pom文件中dependencyManagement里添加依赖
```
 <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2021.1</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```
- 在pom文件中dependencies里添加依赖
```
 <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```
##  抛出异常的方式定义资源
- 定义流控规则
```
public static void initRule(){
    FlowRule flowRule = new FlowRule();
    flowRule.setResource("helloworld");
    flowRule.setCount(2);
    flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    ArrayList arrayList = new ArrayList();
    arrayList.add(flowRule);
    FlowRuleManager.loadRules(arrayList);
}
```
- 判断是否限流
```
 public  static  void helloworld(){
    try{
        Entry helloworld = SphU.entry("helloworld");
        System.out.println("通过");
    }catch (Exception e){
        System.out.println("限流");
    }finally {
        // SphU.entry(xxx) 需要与 entry.exit() 成对出现,否则会导致调用链记录异常
        if (entry != null) {
            entry.exit();
        }
    }

}
```
## 限流实现方式二: 注解方式定义资源
- @SentinelResource(value = "资源名称",blockHandler ="handleFlowQpsException" ,fallback ="queryOrderInfo2Fallback")
- blockHandler = "handleFlowQpsException"用来处理Sentinel 限流/熔断等错误；
- fallback = "queryOrderInfo2Fallback"用来处理接口中业务代码所有异常(如业务代码异常、sentinel限流熔断异常等)
  
在接口OrderQueryService中，使用注解实现订单查询接口的限流：
```
    @SentinelResource(value = "getOrderInfo", blockHandler = "handleFlowQpsException",
            fallback = "queryOrderInfo2Fallback")
    public String queryOrderInfo2(String orderId) {

        // 模拟接口运行时抛出代码异常
        if ("000".equals(orderId)) {
            throw new RuntimeException();
        }

        System.out.println("获取订单信息:" + orderId);
        return "return OrderInfo :" + orderId;
    }

    /**
     * 订单查询接口抛出限流或降级时的处理逻辑
     *
     * 注意: 方法参数、返回值要与原函数保持一致
     * @return
     */
    public String handleFlowQpsException(String orderId, BlockException e) {
        e.printStackTrace();
        return "handleFlowQpsException for queryOrderInfo2: " + orderId;
    }

    /**
     * 订单查询接口运行时抛出的异常提供fallback处理
     *
     * 注意: 方法参数、返回值要与原函数保持一致
     * @return
     */
    public String queryOrderInfo2Fallback(String orderId, Throwable e) {
        return "fallback queryOrderInfo2: " + orderId;
    }
```
## 控制台的使用

### 启动Sentinel控制台
- 下载地址：https://github.com/alibaba/Sentinel/releases
- 启动命令：java -Dserver.port=8080 \
-Dcsp.sentinel.dashboard.server=localhost:8080 \
-jar target/sentinel-dashboard.jar
  ![img.png](img.png)
### 客户端接入（Spring Boot项目接入控制台
- 添加依赖
```
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>${sentinel.version}</version>
</dependency>
```

- 配置 JVM 启动参数：
```
-Dproject.name=sentinel-demo -Dcsp.sentinel.dashboard.server=127.0.0.1:8080 -Dcsp.sentinel.api.port=8719
```
![img_1.png](img_1.png)
- 控制台页面
![img_2.png](img_2.png)