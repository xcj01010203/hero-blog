---
title: 为spring-quartz动态设置参数
date: 2017-09-27 17:36:24
tags:
- spring
- quartz
description: 本文主要介绍如何动态设置spring quartz执行参数。本文spring版本为4.2.2.RELEASE，quartz版本为2.2.1。先说业务，实现一个定时器，默认是每隔100ms执行一次（注意这里是毫秒），并且定时执行的方法需要支持传递参数，参数默认为100。可以通过页面设置定时器每隔100/200/300/400/500/600/700/800/900/1000毫秒执行一次，参数也需要支持自定义功能。
---
maven中的quartz引用如下：
```xml
<dependency>
  <groupId>org.quartz-scheduler</groupId>
  <artifactId>quartz</artifactId>
  <version>2.2.1</version>
</dependency>
```
### 总体思路
这个业务有三个难点：
1. 间隔执行需要精确到毫秒
2. 定时任务需要支持传参
3. 定时任务需要支持动态设置执行时间和参数

首先梳理一下quartz的使用思路，首先需要定义JobDetail，也就是你想要定时执行的任务。有了任务，需要把任务装到Trigger中，也就是触发器，在触发器中，你可以定义job执行的时间。最后一步，需要定义SchedulerFactoryBean（计划工厂），用于装载触发器。注意，工厂里可以装载多个触发器。

### 触发器
在触发器的选择上有两种方式，一种是CronTriggerFactoryBean，一种是SimpleTriggerFactoryBean，下面说一下各自的特点：
* ### CronTriggerFactoryBean：

#### 说明：
这是最常见的方式，随处可搜索到相关代码。此种方式是通过设置cronExpression执行表达式，定时执行任务，相关代码如下：
```xml
<bean id="myTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
  <property name="jobDetail">
      <!--自定义的执行任务-->
      <ref bean="myJobDetail"/>
  </property>
  <!-- cron表达式 -->
  <property name="cronExpression">
      <value>0/5 * * * * ?</value><!--每5秒执行一次-->
    </property>
</bean>
```
#### 优点
表达式的方式优点就是对时间对指定非常灵活，可以指定“每隔星期三”，“每个月2号”等等类似的时间。
#### 缺点
虽然说也可以通过表达式指定任务“每1秒执行一次”，但是，这个时间也仅仅限定在秒的级别，如果想设置“每100毫秒执行一次”，这种方式就没办法了。

* ### SimpleTriggerFactoryBean

#### 说明
这种方式网上传的少，因为这种方式的功能比较单一，仅仅可以设置任务延迟执行时间和重复执行时间间隔。
相关代码如下：
```xml
<bean id="myTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
  <property name="jobDetail" ref="myJobDetail"/>
  <property name="startDelay" value="0"/>
  <property name="repeatInterval" value="1000"/>
</bean>
```
#### 优点
任务重复执行时间间隔可以设定到毫秒级别。
#### 缺点
时间功能设置较少。

> #### 比较了触发器，再结合业务，就可以猜到我使用的是SimpleTriggerFactoryBean

---
### 任务（JobDetail）
说完了触发器，咱再说jobDetail，在spring-quartz中，也有两种的任务定义：MethodInvokingJobDetailFactoryBean（不可传参）和JobDetailFactoryBean（可传参），下面分别做说明：
* ### MethodInvokingJobDetailFactoryBean

#### 说明
这依然是可以搜索到很多示例代码的使用方式，这种方式是执行bean中的执行方法，代码：
```xml
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <!--false表示等上一个任务执行完后再开启新的任务-->
    <property name="concurrent" value="false"/>
    <property name="targetObject" ref="locationController" /><!--目标bean-->
    <property name="targetMethod" value="test" /><!--目标方法-->
</bean>
```
#### 优点
省事，代码比较少，适用于不需要传参的定时任务
#### 缺点
不能传参

* ### JobDetailFactoryBean

#### 说明
这种方式网上讨论的比较少，但是如果想对执行任务传递参数，也就只能使用这种方式。示例代码：
```xml
<!--定义参数列表-->
<bean id="params" class="org.quartz.JobDataMap">
  <constructor-arg>
    <map>
      <entry key="maxAge" value="100"/>
    </map>
  </constructor-arg>
</bean>
<bean id="jobDetail" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
  <!--定时任务-->
  <property name="jobClass" value="com.cao.controller.JobController"/>
  <property name="jobDataMap" ref="params"/>
</bean>
```
可以看到，上面的配置，只见执行类，并没有指定执行的方法。这是因为可传参任务必须继承QuartzJobBean，重写protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException方法，调度任务最终执行的就是executeInternal方法。所以整个JobController就是一个执行任务。JobController代码如下：
```java
package com.cao.controller;

import com.cao.service.LocationService;
import com.cao.util.ApplicationContextUtil;
import org.quartz.JobDataMap;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.quartz.QuartzJobBean;

public class JobController extends QuartzJobBean {

    Logger logger = LoggerFactory.getLogger(JobController.class);

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        try {
            JobDataMap jobDataMap = context.getJobDetail().getJobDataMap();
            int maxAge = jobDataMap.getInt("maxAge");

            //需要定时执行的业务逻辑
            ...
        } catch (Exception e) {
            logger.error("定时获取坐标失败", e);
        }
    }
}

```
#### 优点
可以传参
#### 缺点
代码比较多

> #### 综合以上，可以看出，我采用的是JobDetailFactoryBean

---
### 动态设置参数
这部分的实现思路是，通过SchedulerFactoryBean，获取到对应的触发器（Trigger），设置触发器时间。通过触发器，获取到对应的任务（JobDetail），设置任务的参数。最后再重新部署工厂中的这个任务。实现代码为：
```java
@Autowired
private SchedulerFactoryBean myScheduler;

/**
* 调整定时器设置
* @param time
* @param maxAge
* @throws SchedulerException
*/
public void adjustQuartzTime(int time, int maxAge) throws Exception {
  Scheduler scheduler = this.myScheduler.getScheduler();

  //获取触发器
  TriggerKey key = TriggerKey.triggerKey("myTrigger");
  SimpleTrigger trigger = (SimpleTrigger) scheduler.getTrigger(key);

  //获取JobDetail，并设置job的参数
  JobKey jobKey = trigger.getJobKey();
  JobDetail jobDetail = scheduler.getJobDetail(jobKey);
  jobDetail.getJobDataMap().put("maxAge", maxAge);

  //触发器构建器
  TriggerBuilder<SimpleTrigger> triggerBuilder = trigger.getTriggerBuilder();

  //设置执行时间间隔
  SimpleScheduleBuilder simpleScheduleBuilder = SimpleScheduleBuilder.simpleSchedule();
  simpleScheduleBuilder.withIntervalInMilliseconds(time);
  simpleScheduleBuilder.repeatForever();

  //利用触发器构建器构建新的触发器
  Trigger newTrigger = triggerBuilder.withSchedule(simpleScheduleBuilder).build();

  //这里本来应该使用scheduler.rescheduleJob方法来重新设置任务，但是这个方法不支持传入新的JobDetail，所以后来采用了先删除任务，再添加任务的形式。
  scheduler.deleteJob(jobKey);
  scheduler.scheduleJob(jobDetail, newTrigger);

  //把定时器参数入库，用于页面展示
  QuartzModel quartzModel = this.quartzService.queryRecord();
  quartzModel.setMaxAge(maxAge);
  quartzModel.setTime(time);
  this.quartzService.updateOne(quartzModel);
}
```
