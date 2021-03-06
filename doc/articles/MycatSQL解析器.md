# Mycat-0GC-SQLParser

陈俊文 qq:294712221

Mycat-0GC-SQLParser是Mycat分布式数据库中间件的一个SQL解析组件,0GC是指不会产生不必要的(临时)对象,力求减少内存分配回收造成的性能损耗,在大多数情况下,使用这个解析器不应该产生垃圾回收.



## 背景

传统的SQL解析器需要对query packet的SQL进行编码,分词,语法分析,建立parse tree,而对于Mycat的单节点路由,并不需要一系列的转换产生大量的临时对象,仅仅进行提取表名,就可以进行路由了.这个过程是可以做到不产生临时对象的.

## 定位

在Mycat对query packet的SQL进行解析分为两个阶段

第一阶段使用Mycat自研的SQL解析器

在第一阶段中需要完成以下功能

0GC-SQL解析器的功能受到0GC技术以及单节点路由透传的技术限制,并不完整支持所有SQL语法

第二阶段可以使用第三方SQL解析器进行解析



## 如何评估性能

使用多种复杂的SQL对0GC-SQLParser在JVM的微基准测试工具(JMH)上进行多轮测试,微基准测试可以提供性能测试报告,其中有时间,垃圾回收吞吐量的维度数据.

可能存在的评估维度SQL长度,token数量,列表(..,...,...,),表达式嵌套深度



## 技术与限制



### 词法分析

Mycat-0GC-SQLParser的词法分析中,把token的字符从ASCII可能的字符选择41个字符尽可能映射到一个64位二进制组合里,可以达到一个long可以无冲突存储11个字符,但是mycat的词法处理模式一般而言是固定的,请求中的SQL可以动态变化的,当token长度大于11时,就有可能冲突.

处理这个冲突有以下方法

- 可以使用Hash冲突的方法处理.
- 转至二阶段解析处理


### 语法分析

在解析过程里,排除临时转换对象,分配临时对象的主要原因是存储解析的结果,以便后续的解析能根据上下文判断语义以及方便输出异常提示.如果解析的序列是无限长而且解析结果都需要保存,那么必然要一直动态分配对象以保存解析结果.为了减少产生对象的开销,需要两种降级的手段处理SQL解析

对SQL语法结合单节点路由透传的特点,进行语法裁剪,形成单节点路由专用的语法规则,当SQL满足此规则,则此SQL被识别为单节点路由SQL,如果不满足此语法规则则转至二阶段解析.

在不破坏上述的单节点路由语法规则下,可以使用过滤式或者跳跃式(**Boyer-Moore**)的思想进行解析,仅仅抽取对单节点路由透传必要信息.



### 静态注解

静态注解是在SQL注释里提供路由的注释,与Mycat 1.6一样,注解可以提供部分的或者是完整的路由信息.

注解的信息会覆盖SQL本身提取出来的路由信息.

注解对应的效果是mycat本身的提供的执行类,也可以是用户编写的执行类

一个SQL可以使用多个注解,但是注解之间可能存在冲突,以及注解会被误以为代码顺序关系,故提出注解领域的概念解决此问题.



#### 注解领域

注解在SQL上标注的顺序与实际执行顺序没有关系的,它的执行取决于它的执行在mycat中的处理流程

常用的注解有以下几类

- 路由领域

- 缓存领域

- 监控审计领域

- 拦截阻断领域


一个领域在一个SQL执行流程中只被执行一次

标注了多个相同领域的注解,则执行最后一个



### 动态注解

静态注解的注解领域会覆盖动态注解对应的注解领域,如果不是相同的领域则不会覆盖

动态注解是在mycat里配置SQL中一些字符串模式(简化版正则表达式),table,schema信息,对SQLParser解析结果进行匹配,如果匹配则执行对应注解的执行类

其余规则同静态注解



#### 

### 一阶段解析路由结果

一阶段是为透传设计,不会涉及数据处理,多节点数据汇聚,仅仅是计算出真实的分片节点即可

#### 基础功能

##### 获取以下属性

获取SQL语句中所有的table

获取SQL语句中所有的schema(包含上下文推断)

获取SQL语句中所有的schema.table

获取SQL语句中最外层的limit范围

获取SQL语句的类型

支持使用索引获取一条SQL中的多条SQL语句

支持MyCAT 一条SQL一个或者多个注解解析

获取所有的select 字段 条件字段以及对应的值

##### 规则

- 根据多个分片字段条件以及多个table结合分片算法若能计算出在同一个dataNode,则能透传

- 分片字段获取的语法区域

- select item 中的值

- where 条件以及对应的值

- 一个属性使用唯一一个list存储,例如一个所有table都用一个list存储

- 当所有属性在路由里能计算出唯一一个的dataNode,则执行透传

##### 禁止条件

- 进行两数据类型的关系运算(or (1 =1))

- select *

- 路由中配置的字段在SQL中对应的值必须是数据类型的,而不能是未求值的函数

  如此看来,透传与子查询等时没有关系的,但是在解析上存在一定的难度,子查询提取表名,分片字段条件解析允许实现不完备
