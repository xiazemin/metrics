# section3

Metrics —— JVM上的实时监控类库

96  whthomas 

2016.04.25 22:35\* 字数 1268 阅读 9122评论 10喜欢 20

系统开发到一定的阶段，线上的机器越来越多，就需要一些监控了，除了服务器的监控，业务方面也需要一些监控服务。Metrics作为一款监控指标的度量类库，提供了许多工具帮助开发者来完成自定义的监控工作。



使用Metrics

通过构建一个Spring Boot的基本应用来演示Metrics的工作方式。



在Maven的pom.xml中引入Metrics：



&lt;dependency&gt;

    &lt;groupId&gt;io.dropwizard.metrics&lt;/groupId&gt;

    &lt;artifactId&gt;metrics-core&lt;/artifactId&gt;

    &lt;version&gt;${metrics.version}&lt;/version&gt;

&lt;/dependency&gt;

目前Metrics的最新版本是3.1.2。



Metrics的基本工具

Metrics提供了五个基本的度量类型：



Gauges（度量）

Counters（计数器）

Histograms（直方图数据）

Meters（TPS计算器）

Timers（计时器）

Metrics中MetricRegistry是中心容器，它是程序中所有度量的容器，所有新的度量工具都要注册到一个MetricRegistry实例中才可以使用，尽量在一个应用中保持让这个MetricRegistry实例保持单例。



MetricRegistry 容器

在代码中配置好这个MetricRegistry容器：



@Bean

public MetricRegistry metrics\(\) {

    return new MetricRegistry\(\);

}

Meters TPS计算器

TPS计算器这个名称并不准确，Meters工具会帮助我们统计系统中某一个事件的速率。比如每秒请求数（TPS），每秒查询数（QPS）等等。这个指标能反应系统当前的处理能力，帮助我们判断资源是否已经不足。Meters本身是一个自增计数器。



通过MetricRegistry可以获得一个Meter：



@Bean

public Meter requestMeter\(MetricRegistry metrics\) {

    return metrics.meter\("request"\);

}

在请求中调用mark\(\)方法，来增加计数，我们可以在不同的请求中添加不同的Meter，针对自己的系统完成定制的监控需求。



@RequestMapping\("/hello"\)

@ResponseBody

public String helloWorld\(\) {

    requestMeter.mark\(\);

    return "Hello World";

}

应用运行的过程中，在console中反馈的信息：



-- Meters ----------------------------------------------------------------------

request

             count = 21055

         mean rate = 133.35 events/second

     1-minute rate = 121.66 events/second

     5-minute rate = 36.99 events/second

    15-minute rate = 13.33 events/second

从以上信息中可以看出Meter可以为我们提供平均速率，以及采样后的1分钟，5分钟，15分钟的速率。



Histogram 直方图数据

直方图是一种非常常见的统计图表，Metrics通过这个Histogram这个度量类型提供了一些方便实时绘制直方图的数据。



和之前的Meter相同，我们可以通过MetricRegistry来获得一个Histogram。



@Bean

public Histogram responseSizes\(MetricRegistry metrics\) {

    return metrics.histogram\("response-sizes"\);

}

在应用中，需要统计的位置调用Histogram的update\(\)方法。



responseSizes.update\(new Random\(\).nextInt\(10\)\);

比如我们需要统计某个方法的网络流量，通过Histogram就非常的方便。



在console中Histogram反馈的信息：



-- Histograms ------------------------------------------------------------------

response-sizes

             count = 21051

               min = 0

               max = 9

              mean = 4.55

            stddev = 2.88

            median = 4.00

              75% &lt;= 7.00

              95% &lt;= 9.00

              98% &lt;= 9.00

              99% &lt;= 9.00

            99.9% &lt;= 9.00

Histogram为我们提供了最大值，最小值和平均值等数据，利用这些数据，我们就可以开始绘制自定义的直方图了。



Counter 计数器

Counter的本质就是一个AtomicLong实例，可以增加或者减少值，可以用它来统计队列中Job的总数。



通过MetricRegistry也可以获得一个Counter实例。



@Bean

public Counter pendingJobs\(MetricRegistry metrics\) {

    return metrics.counter\("requestCount"\);

}

在需要统计数据的位置调用inc\(\)和dec\(\)方法。



// 增加计数

pendingJobs.inc\(\);

// 减去计数

pendingJobs.dec\(\);

console的输出非常简单：



-- Counters --------------------------------------------------------------------

requestCount

             count = 21051

只是输出了当前度量的值。



Timer 计时器

Timer是一个Meter和Histogram的组合。这个度量单位可以比较方便地统计请求的速率和处理时间。对于接口中调用的延迟等信息的统计就比较方便了。如果发现一个方法的RPS（请求速率）很低，而且平均的处理时间很长，那么这个方法八成出问题了。



同样，通过MetricRegistry获取一个Timer的实例：



@Bean

public Timer responses\(MetricRegistry metrics\) {

    return metrics.timer\("executeTime"\);

}

在需要统计信息的位置使用这样的代码：



final Timer.Context context = responses.time\(\);

try {

    // handle request

} finally {

    context.stop\(\);

}

console中就会实时返回这个Timer的信息：



-- Timers ----------------------------------------------------------------------

executeTime

             count = 21061

         mean rate = 133.39 calls/second

     1-minute rate = 122.22 calls/second

     5-minute rate = 37.11 calls/second

    15-minute rate = 13.37 calls/second

               min = 0.00 milliseconds

               max = 0.01 milliseconds

              mean = 0.00 milliseconds

            stddev = 0.00 milliseconds

            median = 0.00 milliseconds

              75% &lt;= 0.00 milliseconds

              95% &lt;= 0.00 milliseconds

              98% &lt;= 0.00 milliseconds

              99% &lt;= 0.00 milliseconds

            99.9% &lt;= 0.01 milliseconds

Gauges 度量

除了Metrics提供的几个度量类型，我们可以通过Gauges完成自定义的度量类型。比方说很简单的，我们想看我们缓存里面的数据大小，就可以自己定义一个Gauges。



metrics.register\(

                MetricRegistry.name\(ListManager.class, "cache", "size"\),

                \(Gauge&lt;Integer&gt;\) \(\) -&gt; cache.size\(\)

        \);

这样Metrics就会一直监控Cache的大小。



除此之外有时候，我们需要计算自己定义的一直单位，比如消息队列里面消费者\(consumers\)消费的速率和生产者\(producers\)的生产速率的比例，这也是一个度量。



public class CompareRatio extends RatioGauge {



    private final Meter consumers;

    private final Meter producers;



    public CacheHitRatio\(Meter consumers, Meter producers\) {

        this.consumers = consumers;

        this.producers = producers;

    }



    @Override

    protected Ratio getRatio\(\) {

        return Ratio.of\(consumers.getOneMinuteRate\(\),

                producers.getOneMinuteRate\(\)\);

    }

}

把这个类也注册到Metrics容器里面：



@Bean

public CompareRatio cacheHitRatio\(MetricRegistry metrics, Meter requestMeter, Meter producers\) {



    CompareRatio compareRatio = new CompareRatio\(consumers, producers\);



    metrics.register\("生产者消费者比率", compareRatio\);



    return cacheHitRatio;

}

Reporter 报表

Metrics通过报表，将采集的数据展现到不同的位置,这里比如我们注册一个ConsoleReporter到MetricRegistry中，那么console中就会打印出对应的信息。



@Bean

public ConsoleReporter consoleReporter\(MetricRegistry metrics\) {

    return ConsoleReporter.forRegistry\(metrics\)

            .convertRatesTo\(TimeUnit.SECONDS\)

            .convertDurationsTo\(TimeUnit.MILLISECONDS\)

            .build\(\);

}

除此之外Metrics还支持JMX、HTTP、Slf4j等等，可以访问 http://metrics.dropwizard.io/3.1.0/manual/core/\#reporters 来查看Metrics提供的报表，如果还是不能满足自己的业务，也可以自己继承Metrics提供的ScheduledReporter类完成自定义的报表类。



完整的代码

这个demo是在一个很简单的spring boot下运行的，关键的几个类完整代码如下。



配置类MetricConfig.java



package demo.metrics.config;



import com.codahale.metrics.\*;

import org.slf4j.LoggerFactory;

import org.springframework.context.annotation.Bean;

import org.springframework.context.annotation.Configuration;



import java.util.concurrent.TimeUnit;



@Configuration

public class MetricConfig {



    @Bean

    public MetricRegistry metrics\(\) {

        return new MetricRegistry\(\);

    }



    /\*\*

     \* Reporter 数据的展现位置

     \*

     \* @param metrics

     \* @return

     \*/

    @Bean

    public ConsoleReporter consoleReporter\(MetricRegistry metrics\) {

        return ConsoleReporter.forRegistry\(metrics\)

                .convertRatesTo\(TimeUnit.SECONDS\)

                .convertDurationsTo\(TimeUnit.MILLISECONDS\)

                .build\(\);

    }



    @Bean

    public Slf4jReporter slf4jReporter\(MetricRegistry metrics\) {

        return Slf4jReporter.forRegistry\(metrics\)

                .outputTo\(LoggerFactory.getLogger\("demo.metrics"\)\)

                .convertRatesTo\(TimeUnit.SECONDS\)

                .convertDurationsTo\(TimeUnit.MILLISECONDS\)

                .build\(\);

    }



    @Bean

    public JmxReporter jmxReporter\(MetricRegistry metrics\) {

        return JmxReporter.forRegistry\(metrics\).build\(\);

    }



    /\*\*

     \* 自定义单位

     \*

     \* @param metrics

     \* @return

     \*/

    @Bean

    public ListManager listManager\(MetricRegistry metrics\) {

        return new ListManager\(metrics\);

    }



    /\*\*

     \* TPS 计算器

     \*

     \* @param metrics

     \* @return

     \*/

    @Bean

    public Meter requestMeter\(MetricRegistry metrics\) {

        return metrics.meter\("request"\);

    }



    /\*\*

     \* 直方图

     \*

     \* @param metrics

     \* @return

     \*/

    @Bean

    public Histogram responseSizes\(MetricRegistry metrics\) {

        return metrics.histogram\("response-sizes"\);

    }



    /\*\*

     \* 计数器

     \*

     \* @param metrics

     \* @return

     \*/

    @Bean

    public Counter pendingJobs\(MetricRegistry metrics\) {

        return metrics.counter\("requestCount"\);

    }



    /\*\*

     \* 计时器

     \*

     \* @param metrics

     \* @return

     \*/

    @Bean

    public Timer responses\(MetricRegistry metrics\) {

        return metrics.timer\("executeTime"\);

    }



}

接收请求的类MainController.java



package demo.metrics.action;



import com.codahale.metrics.Counter;

import com.codahale.metrics.Histogram;

import com.codahale.metrics.Meter;

import com.codahale.metrics.Timer;

import demo.metrics.config.ListManager;

import org.springframework.beans.factory.annotation.Autowired;

import org.springframework.stereotype.Controller;

import org.springframework.web.bind.annotation.RequestMapping;

import org.springframework.web.bind.annotation.ResponseBody;



import java.util.Random;



@Controller

@RequestMapping\("/"\)

public class MainController {



    @Autowired

    private Meter requestMeter;



    @Autowired

    private Histogram responseSizes;



    @Autowired

    private Counter pendingJobs;



    @Autowired

    private Timer responses;



    @Autowired

    private ListManager listManager;



    @RequestMapping\("/hello"\)

    @ResponseBody

    public String helloWorld\(\) {



        requestMeter.mark\(\);



        pendingJobs.inc\(\);



        responseSizes.update\(new Random\(\).nextInt\(10\)\);



        listManager.getList\(\).add\(1\);



        final Timer.Context context = responses.time\(\);

        try {

            return "Hello World";

        } finally {

            context.stop\(\);

        }

    }

}

项目启动类DemoApplication.java：



package demo.metrics;



import com.codahale.metrics.ConsoleReporter;

import org.springframework.boot.SpringApplication;

import org.springframework.boot.autoconfigure.SpringBootApplication;

import org.springframework.context.ApplicationContext;



import java.util.concurrent.TimeUnit;



@SpringBootApplication

public class DemoApplication {



    public static void main\(String\[\] args\) {

        ApplicationContext ctx = SpringApplication.run\(DemoApplication.class, args\);



        // 启动Reporter

        ConsoleReporter reporter = ctx.getBean\(ConsoleReporter.class\);

        reporter.start\(1, TimeUnit.SECONDS\);

        

    }

}







