# Flowable 框架使用

> Flowable是一个使用Java编写的轻量级业务流程引擎。Flowable流程引擎可用于部署BPMN 2.0流程定义（用于定义流程的行业XML标准）， 创建这些流程定义的流程实例，进行查询，访问运行中或历史的流程实例与相关数据，等等。

[toc]

## 使用笔记(spring boot)

### 添加依赖

**pom.xml**

```xml
<project>
    ...
    <properties>
        ...
        <flowable.version>6.3.0</flowable.version>
    </properties>
    <dependencies>
            ...
        <!--- spring boot web --->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--- flowable --->
		<dependency>
            <groupId>org.flowable</groupId>
            <artifactId>flowable-spring-boot-starter</artifactId>
            <version>${flowable.version}</version>
        </dependency>
        <!--- mysql --->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
</project>
```

### 添加配置

**application.yml**

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/flowable?characterEncoding=UTF-8&serverTimezone=UTC
    driverClassName: com.mysql.cj.jdbc.Driver
    username: root
    password: Gkmysql1@3
```

### 定义流程（BPMN）

使用**Flowable Modeler**设计流程图

#### 应用界面

![image-20220824190320075](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824190321.png)

#### **简单流程图实例**

![image-20220824192303941](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824192305.png)

#### 常用节点

![image-20220824192751951](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824192753.png)

##### 事件

> 事件（event）通常用于为流程生命周期中发生的事情建模。事件总是图形化为圆圈。

- 启动事件
- 结束事件

流程当流程启动时，会自动启动事件开始，至到结束事件结束

##### 顺序流

> 顺序流（sequence flow）是流程中两个元素间的连接器。在流程执行过程中，一个元素被访问后，会沿着其所有出口顺序流继续执行。

![bpmn.sequence.flow](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824193246.png)

用从源元素指向目标元素的箭头表示。箭头总是指向目标元素。

```xml
<sequenceFlow id="flow1" sourceRef="theStart" targetRef="theTask" />
```

可以附加条件。

```xml
<sequenceFlow id="flow" sourceRef="theStart" targetRef="theTask">
  <conditionExpression xsi:type="tFormalExpression">
    <![CDATA[${order.price > 100 && order.price < 250}]]>
  </conditionExpression>
</sequenceFlow>
```

##### 活动

- ###### 用户任务

  >  “用户任务（user task）”用于对需要人工执行的任务进行建模。当流程执行到达用户任务时，会为指派至该任务的用户或组的任务列表创建一个新任务。

  ```xml
  <userTask id="theTask" name="Important task" />
  ```

- ###### 服务任务

  > Java服务任务（Java service task）用于调用Java类。

  ```xml
  <serviceTask id="javaService"
               name="My Java Service Task"
               flowable:class="org.flowable.MyJavaDelegate" />
  ```

  四种方法声明调用逻辑

  - 指定实现了JavaDelegate或ActivityBehavior的类
  - 调用解析为委托对象（delegation object）的表达式
  - 调用方法表达式（method expression）
  - 对值表达式（value expression）求值

  ```java
  public class ToUppercase implements JavaDelegate {
    public void execute(DelegateExecution execution) {
      String var = (String) execution.getVariable("input");
      var = var.toUpperCase();
      execution.setVariable("input", var);
    }
  }
  ```

  **注意**

  - **Flowable只会为serviceTask上定义的Java类创建一个实例**。所有流程实例共享同一个类实例，用于调用*execute(DelegateExecution)*。这意味着该类不能有任何成员变量，并需要是线程安全的，因为它可能会在不同线程中同时执行。
  - **不会在部署时实例化**，只有当流程执行第一次到达该类使用的地方时，才会创建该类的实例。如果找不到这个类，会抛出`FlowableException`。

- ###### 脚本任务

  > 脚本任务（script task）是自动执行的活动。当流程执行到达脚本任务时，会执行相应的脚本。

  ```xml
  <scriptTask id="theScriptTask" name="Execute script" scriptFormat="groovy">
    <script>
      sum = 0
      for ( i in inputArray ) {
        sum += i
      }
    </script>
  </scriptTask>
  ```

  

- ###### 接收任务

  > 接收任务（receive task），是等待特定消息到达的简单任务。目前，我们只为这个任务实现了Java语义。当流程执行到达接收任务时，流程状态将提交至持久化存储。这意味着流程将保持等待状态，直到引擎接收到特定的消息，触发流程穿过接收任务继续执行。

  ```xml
  <receiveTask id="waitState" name="wait" />
  ```

  **继续执行**

  ```java
  ProcessInstance pi = runtimeService.startProcessInstanceByKey("receiveTask");
  Execution execution = runtimeService.createExecutionQuery()
    .processInstanceId(pi.getId())
    .activityId("waitState")
    .singleResult();
  assertNotNull(execution);
  
  runtimeService.trigger(execution.getId());
  ```

- ###### Shell任务

  > Shell任务（Shell task）可以运行Shell脚本与命令。

  ```xml
  <serviceTask id="shellEcho" flowable:type="shell">
  ```

##### 网关

> 网关（gateway）用于控制执行的流向（或者按BPMN 2.0的用词：执行的*“标志（token）”*）。网关可以*消费（consuming）*与*生成（generating）*标志。网关用其中带有图标的菱形表示。该图标显示了网关的类型。

![bpmn.gateway](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824195344.png)

- ###### 排他网关

  > 对流程中的**决策**建模。当执行到达这个网关时，会按照所有出口顺序流定义的顺序对它们进行计算。选择第一个条件计算为true的顺序流（当没有设置条件时，认为顺序流为*true*）继续流程。

  ![bpmn.exclusive.gateway](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824195750.png)

- ###### 并行网关

  > 将执行*分支（fork）*为多条路径，也可以*合并（join）*多条入口路径的执行。
  >
  > 并行网关的功能取决于其入口与出口顺序流：
  >
  > - **分支：**所有的出口顺序流都并行执行，为每一条顺序流创建一个并行执行。
  > - **合并：**所有到达并行网关的并行执行都会在网关处等待，直到每一条入口顺序流都到达了有个执行。然后流程经过该合并网关继续。
  >
  > 请注意，如果并行网关同时具有多条入口与出口顺序流，可以**同时具有分支与合并的行为**。在这种情况下，网关首先合并所有入口顺序流，然后分裂为多条并行执行路径。
  >
  > **与其他网关类型有一个重要区别：并行网关不计算条件。如果连接到并行网关的顺序流上定义了条件，会直接忽略该条件。**

  ![bpmn.parallel.gateway](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824195806.png)

- ###### 包容网关

  > 可以把**包容网关（inclusive gateway）**看做排他网关与并行网关的组合。与排他网关一样，可以在包容网关的出口顺序流上定义条件，包容网关会计算条件。然而主要的区别是，包容网关与并行网关一样，可以同时选择多于一条出口顺序流。
  >
  > 包容网关的功能取决于其入口与出口顺序流：
  >
  > - **分支：**流程会计算所有出口顺序流的条件。对于每一条计算为true的顺序流，流程都会创建一个并行执行。
  > - **合并：**所有到达包容网关的并行执行，都会在网关处等待。直到每一条具有流程标志（process token）的入口顺序流，都有一个执行到达。这是与并行网关的重要区别。换句话说，包容网关只会等待可以被执行的入口顺序流。在合并后，流程穿过合并并行网关继续。
  >
  > 请注意，如果包容网关同时具有多条入口与出口顺序流，可以**同时具有分支与合并的行为**。在这种情况下，网关首先合并所有具有流程标志的入口顺序流，然后为每一个条件计算为true的出口顺序流分裂出并行执行路径。

  ![bpmn.unbalanced.parallel.gateway](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824195814.png)

  ### 下载BPMN文件

  <img src="https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824200516.png" alt="image-20220824200515532"/>

  将`bpmn20.xml`文件放入`src/main/resources/processes`文件夹下可以自动被读取并部署到系统中

  ### 流程控制

  流程控制代码可以写到`Controller`里面

  **实例**

  ```java
  @RestController
  @RequestMapping("flowable")
  public class TestController {
  
      @Autowired
      private RuntimeService runtimeService;
      @Autowired
      private TaskService taskService;
      @Autowired
      private HistoryService historyService;
      @Autowired
      private RepositoryService repositoryService;
      @Autowired
      private ProcessEngine processEngine;
  
      /**
       * 创建流程
       *
       * @param userId
       * @param days
       * @param reason
       * @return
       */
      @GetMapping("add")
      public String addExpense(String userId, String days, String reason) {
          Map<String, Object> map = new HashMap<>();
          map.put("employee", userId);
          map.put("nrOfHolidays", days);
          map.put("description", reason);
  
          ProcessInstance processInstance = runtimeService.startProcessInstanceByKey("holidayRequest", map);
          return "提交成功,流程ID为：" + processInstance.getId();
      }
  
      /**
       * 获取指定用户组流程任务列表
       *
       * @param group
       * @return
       */
      @GetMapping("list")
      public Object list(String group) {
          List<Task> tasks = taskService.createTaskQuery().taskCandidateGroup(group).list();
          return tasks.toString();
      }
  
      /**
       * 通过/拒绝任务
       *
       * @param taskId
       * @param approved 1 ：true  2：false
       * @return
       */
      @GetMapping("apply")
      public String apply(String taskId, String approved) {
          Task task = taskService.createTaskQuery().taskId(taskId).singleResult();
          if (task == null) {
              return "流程不存在";
          }
          Map<String, Object> variables = new HashMap<>();
          Boolean apply = approved.equals("1") ? true : false;
          variables.put("approved", apply);
          taskService.complete(taskId, variables);
          return "审批是否通过：" + approved;
  
      }
  ```

  

  ### 项目结构

  ![image-20220824200938866](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824200940.png)

### 测试

#### 申请

![image-20220824201316057](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824201317.png)

#### 查询进度

![image-20220824201349175](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824201350.png)

#### 查询任务

![image-20220824201423431](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824201424.png)

#### 审批

![image-20220824201449945](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824201451.png)

![image-20220824201512573](https://gitee.com/shiqiguo/figurebed/raw/master/img/20220824201513.png)

## 资料

[Flowable官网]: https://www.flowable.com/
[官方文档]:https://www.flowable.com/open-source/docs/oss-introduction
[中文文档]:https://tkjohn.github.io/flowable-userguide/



## 相关工具

### Flowable网页工具

[官网]:https://github.com/flowable/flowable-engine

#### 部署

```shell
docker run -p 8080:8080 flowable/all-in-one
```

#### 访问网页

- [Flowable IDM]:(http://localhost:8080/flowable-idm
- [Flowable Modeler]:http://localhost:8080/flowable-modeler
- [Flowable Task]:http://localhost:8080/flowable-task
- [Flowable Admin]:http://localhost:8080/flowable-admin

#### 默认用户

用户名：admin
密码：test