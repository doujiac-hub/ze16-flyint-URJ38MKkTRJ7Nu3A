
## Quartz基本概念




```
Quartz是一个任务调度框架，主要用于在特定时间触发任务执行。‌

Quartz的核心概念
‌调度器（Scheduler）‌：负责任务的调度和管理，包括任务的启动、暂停、恢复等操作。
‌任务（Job）‌：需要实现org.quartz.Job接口的execute方法，定义了任务的具体执行逻辑。
‌触发器（Trigger）‌：定义任务执行的触发条件，包括简单触发器（SimpleTrigger）和cron触发器（CronTrigger）。
‌任务详情（JobDetail）‌：用于定义任务的详细信息，如任务名、组名等。
‌任务构建器（JobBuilder）和触发器构建器（TriggerBuilder）‌：用于定义和构建任务和触发器的实例。
‌线程池（ThreadPool）‌：用于并行调度执行每个作业，提高效率。
‌监听器（Listener）‌：包括任务监听器、触发器监听器和调度器监听器，用于监听任务和触发器的状态变化。
Quartz的基本使用步骤
‌创建任务类‌：实现Job接口的execute方法，定义任务的执行逻辑。
‌生成任务详情（JobDetail）‌：通过JobBuilder定义任务的详细信息。
‌生成触发器（Trigger）‌：通过TriggerBuilder定义任务的触发条件，可以选择使用简单触发器或cron触发器。
‌获取调度器（Scheduler）‌：通过SchedulerFactory创建调度器对象，并将任务和触发器绑定在一起，启动调度器。
Quartz的优点和缺点
‌优点‌：支持复杂的调度需求，包括定时、重复执行、并发执行等；提供了丰富的API和工具类，易于使用和维护；支持Spring集成，方便在Spring项目中应用。
‌缺点‌：配置复杂，需要一定的学习成本；对于简单的定时任务，使用Quartz可能会显得过于复杂。
```


## 整合SpringBoot


第一步：添加依赖




```
<dependency>
    <groupId>org.springframework.bootgroupId>
    <artifactId>spring-boot-starter-quartzartifactId>
dependency>
```


第二步：创建scheduler




```
package com.xy.quartz;

import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.SchedulerFactory;
import org.quartz.impl.StdSchedulerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class QuartzConfig {
    @Bean
    public Scheduler scheduler() throws SchedulerException {
        SchedulerFactory schedulerFactoryBean = new StdSchedulerFactory();
        return schedulerFactoryBean.getScheduler();
    }
}
```


第三步：创建Job




```
import com.songwp.utils.DateUtil;
import lombok.extern.slf4j.Slf4j;
import org.quartz.Job;
import org.quartz.JobDataMap;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.stereotype.Component;

import java.util.Date;

@Slf4j
@Component
public class MyJob implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        JobDataMap jobDataMap = jobExecutionContext.getJobDetail().getJobDataMap();
        log.info("入参:{}", jobDataMap.toString());
        log.info("执行定时任务的时间:{}", DateUtil.dateToStr(new Date()));
    }
}
```


第四步：创建任务信息类




```
import lombok.Data;

@Data
public class JobInfo {
    private Long jobId;
    private String cronExpression;
    private String businessId;
}
```


第五步：创建JobDetail和trigger创建包装类




```
import com.songwp.domain.quartz.JobInfo;
import com.songwp.test.MyJob;
import org.quartz.*;
import java.util.Date;

public class QuartzBuilder {
    public static final String RUN_CRON ="定时执行";
    public static final String RUN_ONE ="执行一次";
    private static final String JOB_NAME_PREFIX ="flow";
    public static final String TRIGGER_NAME_PREFIX ="trigger.";

    public static JobDetail createJobDetail(JobInfo jobInfo, String type){
        String jobKey =JOB_NAME_PREFIX + jobInfo.getJobId();
        if (RUN_ONE.equals(type)){
            jobKey = JOB_NAME_PREFIX + new Date().getTime();
        }

        return JobBuilder.newJob(MyJob.class)
                .withIdentity(jobKey,"my_group")
                .usingJobData("businessId",jobInfo.getBusinessId())
                .usingJobData("businessType","其他参数")
                .storeDurably().build();
    }


    public static Trigger createTrigger(JobDetail jobDetail, JobInfo jobInfo){
        return TriggerBuilder.newTrigger()
                .forJob(jobDetail)
                .withIdentity(TRIGGER_NAME_PREFIX + jobInfo.getJobId(),RUN_CRON)
                .withSchedule(CronScheduleBuilder.cronSchedule(jobInfo.getCronExpression()))
                .build();
        }
}
```


第六步：控制层接口实现接口（执行一次、启动定时、暂停任务）




```
import com.songwp.config.quartz.QuartzBuilder;
import com.songwp.domain.quartz.JobInfo;
import lombok.extern.slf4j.Slf4j;
import org.quartz.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import javax.annotation.PostConstruct;

@RestController
@Slf4j
public class QuartzController {

    @Autowired
    private Scheduler scheduler;

    /**
     * 定时任务执行（只执行一次）
     * @return
     */
    @GetMapping("runOne")
    public  String runOne(){
        JobInfo jobInfo = new JobInfo();
        jobInfo.setJobId(1L);
        jobInfo.setBusinessId("123");
        jobInfo.setCronExpression("0/5 * * * * ?");
        JobDetail jobDetail = QuartzBuilder.createJobDetail(jobInfo, QuartzBuilder.RUN_ONE);
        Trigger trigger = TriggerBuilder.newTrigger()
                .forJob(jobDetail)
                .startNow()
                .withSchedule(SimpleScheduleBuilder.simpleSchedule().withRepeatCount(0))
                .build();

        try {
            scheduler.scheduleJob(jobDetail, trigger);
            if (!scheduler.isStarted()) {
                scheduler.start();
            }
        } catch (SchedulerException e) {
            log.error(e.getMessage());
            return "执行失败";
        }
        return "执行成功";
    }

    /**
     * 开始定时执行
     * @return 执行结果
     */
    @GetMapping("start")
    public String start() {
        try {
            JobInfo jobInfo = new JobInfo();
            jobInfo.setJobId(1L);
            jobInfo.setBusinessId("123");
            jobInfo.setCronExpression("0/5 * * * * ?");
            TriggerKey triggerKey = new TriggerKey(QuartzBuilder.TRIGGER_NAME_PREFIX + jobInfo.getJobId(),QuartzBuilder.RUN_CRON);
            if (scheduler.checkExists(triggerKey)) {
                scheduler.resumeTrigger(triggerKey);
            } else {
                JobDetail jobDetail = QuartzBuilder.createJobDetail(jobInfo, QuartzBuilder.RUN_CRON);
                Trigger trigger = QuartzBuilder.createTrigger(jobDetail, jobInfo);
                scheduler.scheduleJob(jobDetail, trigger);
                if (!scheduler.isStarted()) {
                    scheduler.start();
                }
            }
        } catch (SchedulerException e) {
            log.error(e.getMessage());
            return "执行失败";

        }
        return "执行成功";
    }

    /**
     * 停止任务执行
     * @return 执行结果
     */
    @GetMapping("pause")
    public String pause() {
        try {
            JobInfo jobInfo = new JobInfo();
            jobInfo.setJobId(1L);
            jobInfo.setBusinessId("123");
            jobInfo.setCronExpression("0/5 * * * * ?");
            TriggerKey triggerKey = new TriggerKey(QuartzBuilder.TRIGGER_NAME_PREFIX + jobInfo.getJobId(), QuartzBuilder.RUN_CRON);
            if (scheduler.checkExists(triggerKey)) {
                scheduler.pauseTrigger(triggerKey);
            }
        } catch (SchedulerException e) {
            log.error(e.getMessage());
            return "执行失败";
        }
        return "执行成功";
    }

    /**
     * 查询已启动状态的任务，然后重新执行
     */
    @PostConstruct
    public void init(){
       log.info("查询已启动状态的任务，然后重新执行");
        start();
    }

}
```


**最后访问接口：**




```
http://localhost:8080/runOne
http://localhost:8080/start
http://localhost:8080/pause
```


[![](https://img2024.cnblogs.com/blog/2156747/202410/2156747-20241030170101593-626886036.png)](https://img2024.cnblogs.com/blog/2156747/202410/2156747-20241030170101593-626886036.png)


**正常情况下的步骤应该是这样:**


1、创建任务时记录到任务表job\_info，此时初始状态为0


2、启动任务时更新任务表状态，更新为1


3、如果应用关闭了，那么在下次应用启动的时候，需要把状态为1的任务也给启动了，就不需要认为再去调接口启动。


  * [Quartz基本概念](#tid-QNBjsx)
* [整合SpringBoot](#tid-t7TpMN)

   \_\_EOF\_\_

   ![](https://github.com/songweipeng)奋 斗  - **本文链接：** [https://github.com/songweipeng/p/18516181](https://github.com):[楚门加速器p](https://tianchuang88.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
