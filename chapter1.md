# chapter1

Metrics可以为你的代码的运行提供无与伦比的洞察力。作为一款监控指标的度量类库，它提供了很多模块可以为第三方库或者应用提供辅助统计信息， 比如Jetty, Logback, Log4j, Apache HttpClient, Ehcache, JDBI, Jersey, 它还可以将度量数据发送给Ganglia和Graphite以提供图形化的监控。



Metrics提供了Gauge、Counter、Meter、Histogram、Timer等度量工具类以及Health Check功能。







引用Metric库

将metrics-core加入到maven pom.xml中：





复制代码

&lt;dependencies&gt;

    &lt;dependency&gt;

        &lt;groupId&gt;com.codahale.metrics&lt;/groupId&gt;

        &lt;artifactId&gt;metrics-core&lt;/artifactId&gt;

        &lt;version&gt;${metrics.version}&lt;/version&gt;

    &lt;/dependency&gt;

&lt;/dependencies&gt;

复制代码

将metrics.version 设置为metrics最新的版本。

现在你可以在你的程序代码中加入一些度量了。



Registry

Metric的中心部件是MetricRegistry。 它是程序中所有度量metric的容器。让我们接着在代码中加入一行：



final MetricRegistry metrics = new MetricRegistry\(\);

Gauge \(仪表\)

Gauge代表一个度量的即时值。 当你开汽车的时候， 当前速度是Gauge值。 你测体温的时候， 体温计的刻度是一个Gauge值。 当你的程序运行的时候， 内存使用量和CPU占用率都可以通过Gauge值来度量。

比如我们可以查看一个队列当前的size。





复制代码

public class QueueManager {

    private final Queue queue;

    public QueueManager\(MetricRegistry metrics, String name\) {

        this.queue = new Queue\(\);

        metrics.register\(MetricRegistry.name\(QueueManager.class, name, "size"\),

                         new Gauge&lt;Integer&gt;\(\) {

                             @Override

                             public Integer getValue\(\) {

                                 return queue.size\(\);

                             }

                         }\);

    }

}

复制代码

registry 中每一个metric都有唯一的名字。 它可以是以.连接的字符串。 如"things.count" 和 "com.colobu.Thing.latency"。 MetricRegistry 提供了一个静态的辅助方法用来生成这个名字：



MetricRegistry.name\(QueueManager.class, "jobs", "size"\)

生成的name为com.colobu.QueueManager.jobs.size。



实际编程中对于队列或者类似队列的数据结构，你不会简单的度量queue.size\(\)， 因为在java.util和java.util.concurrent包中大部分的queue的\#size是O\(n\)，这意味的调用此方法会有性能的问题， 更深一步，可能会有lock的问题。

RatioGauge可以计算两个Gauge的比值。 Meter和Timer可以参考下面的代码创建。下面的代码用来计算计算命中率 \(hit/call\)。





复制代码

public class CacheHitRatio extends RatioGauge {

    private final Meter hits;

    private final Timer calls;

    public CacheHitRatio\(Meter hits, Timer calls\) {

        this.hits = hits;

        this.calls = calls;

    }

    @Override

    public Ratio getValue\(\) {

        return Ratio.of\(hits.oneMinuteRate\(\),

                        calls.oneMinuteRate\(\)\);

    }

}

复制代码

CachedGauge可以缓存耗时的测量。DerivativeGauge可以引用另外一个Gauage。



Counter \(计数器\)

Counter是一个AtomicLong实例， 可以增加或者减少值。 例如，可以用它来计数队列中加入的Job的总数。 





复制代码

private final Counter pendingJobs = metrics.counter\(name\(QueueManager.class, "pending-jobs"\)\);

public void addJob\(Job job\) {

    pendingJobs.inc\(\);

    queue.offer\(job\);

}

public Job takeJob\(\) {

    pendingJobs.dec\(\);

    return queue.take\(\);

}

复制代码

和上面Gauage不同，这里我们使用的是metrics.counter方法而不是metrics.register方法。 使用metrics.counter更简单。



Meter \(\)

Meter用来计算事件的速率。 例如 request per second。 还可以提供1分钟，5分钟，15分钟不断更新的平均速率。





private final Meter requests = metrics.meter\(name\(RequestHandler.class, "requests"\)\);

public void handleRequest\(Request request, Response response\) {

    requests.mark\(\);

    // etc

}

Histogram \(直方图\)

Histogram可以为数据流提供统计数据。 除了最大值，最小值，平均值外，它还可以测量 中值\(median\)，百分比比如XX%这样的Quantile数据 。





private final Histogram responseSizes = metrics.histogram\(name\(RequestHandler.class, "response-sizes"\);

public void handleRequest\(Request request, Response response\) {

    // etc

    responseSizes.update\(response.getContent\(\).length\);

}

这个例子用来统计response的字节数。

Metrics提供了一批的Reservoir实现，非常有用。例如SlidingTimeWindowReservoir 用来统计最新N个秒\(或其它时间单元）的数据。



Timer \(计时器\)

Timer用来测量一段代码被调用的速率和用时。





复制代码

private final Timer responses = metrics.timer\(name\(RequestHandler.class, "responses"\)\);

public String handleRequest\(Request request, Response response\) {

    final Timer.Context context = responses.time\(\);

    try {

        // etc;

        return "OK";

    } finally {

        context.stop\(\);

    }

}

复制代码

这段代码用来计算中间的代码用时以及request的速率。



Health Check \(健康检查\)

Metric还提供了服务健康检查能力， 由metrics-healthchecks模块提供。

先创建一个HealthCheckRegistry实例。



final HealthCheckRegistry healthChecks = new HealthCheckRegistry\(\);

再实现一个HealthCheck子类， 用来检查数据库的状态。





复制代码

public class DatabaseHealthCheck extends HealthCheck {

    private final Database database;

    public DatabaseHealthCheck\(Database database\) {

        this.database = database;

    }

    @Override

    public HealthCheck.Result check\(\) throws Exception {

        if \(database.isConnected\(\)\) {

            return HealthCheck.Result.healthy\(\);

        } else {

            return HealthCheck.Result.unhealthy\("Cannot connect to " + database.getUrl\(\)\);

        }

    }

}

复制代码

注册一下。



healthChecks.register\("mysql", new DatabaseHealthCheck\(database\)\);

最后运行健康检查并查看检查结果。





复制代码

final Map&lt;String, HealthCheck.Result&gt; results = healthChecks.runHealthChecks\(\);

for \(Entry&lt;String, HealthCheck.Result&gt; entry : results.entrySet\(\)\) {

    if \(entry.getValue\(\).isHealthy\(\)\) {

        System.out.println\(entry.getKey\(\) + " is healthy"\);

    } else {

        System.err.println\(entry.getKey\(\) + " is UNHEALTHY: " + entry.getValue\(\).getMessage\(\)\);

        final Throwable e = entry.getValue\(\).getError\(\);

        if \(e != null\) {

            e.printStackTrace\(\);

        }

    }

}

复制代码

Metric内置一个ThreadDeadlockHealthCheck， 它使用java内置的线程死锁检查方法来检查程序中是否有死锁。



JMX报表

通过JMX报告Metric。



final JmxReporter reporter = JmxReporter.forRegistry\(registry\).build\(\);

reporter.start\(\);

一旦启动， 所有registry中注册的metric都可以通过JConsole或者VisualVM查看 \(通过MBean插件\)。



HTTP报表

Metric也提供了一个servlet \(AdminServlet\)提供JSON风格的报表。它还提供了单一功能的servlet \(MetricsServlet, HealthCheckServlet, ThreadDumpServlet, PingServlet\)。

你需要在pom.xml加入metrics-servlets。



&lt;dependency&gt;

    &lt;groupId&gt;com.codahale.metrics&lt;/groupId&gt;

    &lt;artifactId&gt;metrics-servlets&lt;/artifactId&gt;

    &lt;version&gt;${metrics.version}&lt;/version&gt;

&lt;/dependency&gt;

其它报表

除了JMX和HTTP, metric还提供其它报表。



STDOUT, using ConsoleReporter from metrics-core

final ConsoleReporter reporter = ConsoleReporter.forRegistry\(registry\)

                                                .convertRatesTo\(TimeUnit.SECONDS\)

                                                .convertDurationsTo\(TimeUnit.MILLISECONDS\)

                                                .build\(\);

reporter.start\(1, TimeUnit.MINUTES\);

CSV files, using CsvReporter from metrics-core

final CsvReporter reporter = CsvReporter.forRegistry\(registry\)

                                        .formatFor\(Locale.US\)

                                        .convertRatesTo\(TimeUnit.SECONDS\)

                                        .convertDurationsTo\(TimeUnit.MILLISECONDS\)

                                        .build\(new File\("~/projects/data/"\)\);

reporter.start\(1, TimeUnit.SECONDS\);

SLF4J loggers, using Slf4jReporter from metrics-core

final Slf4jReporter reporter = Slf4jReporter.forRegistry\(registry\)

                                            .outputTo\(LoggerFactory.getLogger\("com.example.metrics"\)\)

                                            .convertRatesTo\(TimeUnit.SECONDS\)

                                            .convertDurationsTo\(TimeUnit.MILLISECONDS\)

                                            .build\(\);

reporter.start\(1, TimeUnit.MINUTES\);

Ganglia, using GangliaReporter from metrics-ganglia

final GMetric ganglia = new GMetric\("ganglia.example.com", 8649, UDPAddressingMode.MULTICAST, 1\);

final GangliaReporter reporter = GangliaReporter.forRegistry\(registry\)

                                                .convertRatesTo\(TimeUnit.SECONDS\)

                                                .convertDurationsTo\(TimeUnit.MILLISECONDS\)

                                                .build\(ganglia\);

reporter.start\(1, TimeUnit.MINUTES\);

Graphite, using GraphiteReporter from metrics-graphite

MetricSet

可以将一组Metric组织成一组便于重用。



复制代码

final Graphite graphite = new Graphite\(new InetSocketAddress\("graphite.example.com", 2003\)\);

final GraphiteReporter reporter = GraphiteReporter.forRegistry\(registry\)

                                                  .prefixedWith\("web1.example.com"\)

                                                  .convertRatesTo\(TimeUnit.SECONDS\)

                                                  .convertDurationsTo\(TimeUnit.MILLISECONDS\)

                                                  .filter\(MetricFilter.ALL\)

                                                  .build\(graphite\);

reporter.start\(1, TimeUnit.MINUTES\);

复制代码

一些模块

metrics-json提供了json格式的序列化。

以及为其它库提供度量的能力

metrics-ehcache

metrics-httpclient

metrics-jdbi

metrics-jersey

metrics-jetty

metrics-log4j

metrics-logback

metrics-jvm

metrics-servlet 注意不是metrics-servlets

第三方库

metrics-librato 提供Librato Metrics报表

Metrics Spring Integration 提供了Spring的集成

sematext-metrics-reporter 提供了SPM报表.

wicket-metrics提供Wicket应用.

metrics-guice 提供Guice集成.

metrics-scala 提供了为Scala优化的API.

这里重点介绍一下Metrics for Spring



Metrics for Spring

这个库为Spring增加了Metric库， 提供基于XML或者注解方式。



可以使用注解创建metric和代理类。 @Timed, @Metered, @ExceptionMetered, @Counted

为注解了 @Gauge 和 @CachedGauge的bean注册Gauge

为@Metric注解的字段自动装配

注册HealthCheck

通过XML配置产生报表

通过XML注册metric和metric组

你需要在pom.xml加入



&lt;dependency&gt;

    &lt;groupId&gt;com.ryantenney.metrics&lt;/groupId&gt;

    &lt;artifactId&gt;metrics-spring&lt;/artifactId&gt;

    &lt;version&gt;3.0.1&lt;/version&gt;

&lt;/dependency&gt;

基本用法

XML风格的配置

复制代码

&lt;beans xmlns="http://www.springframework.org/schema/beans"

       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"

       xmlns:metrics="http://www.ryantenney.com/schema/metrics"

       xsi:schemaLocation="

           http://www.springframework.org/schema/beans

           http://www.springframework.org/schema/beans/spring-beans-3.2.xsd

           http://www.ryantenney.com/schema/metrics

           http://www.ryantenney.com/schema/metrics/metrics-3.0.xsd"&gt;

    &lt;!-- Registry should be defined in only one context XML file --&gt;

    &lt;metrics:metric-registry id="metrics" /&gt;

    &lt;!-- annotation-driven must be included in all context files --&gt;

    &lt;metrics:annotation-driven metric-registry="metrics" /&gt;

    &lt;!-- \(Optional\) Registry should be defined in only one context XML file --&gt;

    &lt;metrics:reporter type="console" metric-registry="metrics" period="1m" /&gt;

    &lt;!-- \(Optional\) The metrics in this example require the metrics-jvm jar--&gt;

    &lt;metrics:register metric-registry="metrics"&gt;

        &lt;bean metrics:name="jvm.gc" class="com.codahale.metrics.jvm.GarbageCollectorMetricSet" /&gt;

        &lt;bean metrics:name="jvm.memory" class="com.codahale.metrics.jvm.MemoryUsageGaugeSet" /&gt;

        &lt;bean metrics:name="jvm.thread-states" class="com.codahale.metrics.jvm.ThreadStatesGaugeSet" /&gt;

        &lt;bean metrics:name="jvm.fd.usage" class="com.codahale.metrics.jvm.FileDescriptorRatioGauge" /&gt;

    &lt;/metrics:register&gt;

    &lt;!-- Beans and other Spring config --&gt;

&lt;/beans&gt;



java注解的方式



import java.util.concurrent.TimeUnit;

import org.springframework.context.annotation.Configuration;

import com.codahale.metrics.ConsoleReporter;

import com.codahale.metrics.MetricRegistry;

import com.codahale.metrics.SharedMetricRegistries;

import com.ryantenney.metrics.spring.config.annotation.EnableMetrics;

import com.ryantenney.metrics.spring.config.annotation.MetricsConfigurerAdapter;

@Configuration

@EnableMetrics

public class SpringConfiguringClass extends MetricsConfigurerAdapter {

    @Override

    public void configureReporters\(MetricRegistry metricRegistry\) {

        ConsoleReporter

            .forRegistry\(metricRegistry\)

            .build\(\)

            .start\(1, TimeUnit.MINUTES\);

    }

}



