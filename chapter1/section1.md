# section1

当我们需要为某个系统某个服务做监控、做统计，就需要用到Metrics。



举个栗子，一个图片压缩服务：



每秒钟的请求数是多少（TPS）？

平均每个请求处理的时间？

请求处理的最长耗时？

等待处理的请求队列长度？

又或者一个缓存服务：



缓存的命中率？

平均查询缓存的时间？

基本上每一个服务、应用都需要做一个监控系统，这需要尽量以少量的代码，实现统计某类数据的功能。



以 Java 为例，目前最为流行的 metrics 库是来自 Coda Hale 的 dropwizard/metrics，该库被广泛地应用于各个知名的开源项目中。例如 Hadoop，Kafka，Spark，JStorm 中。



本文就结合范例来主要介绍下 dropwizard/metrics 的概念和用法。



Maven 配置

我们需要在pom.xml中依赖 metrics-core 包：



&lt;dependencies&gt;

&lt;dependency&gt;

&lt;groupId&gt;io.dropwizard.metrics&lt;/groupId&gt;

&lt;artifactId&gt;metrics-core&lt;/artifactId&gt;

&lt;version&gt;${metrics.version}&lt;/version&gt;

&lt;/dependency&gt;

&lt;/dependencies&gt;

注：在POM文件中需要声明 ${metrics.version} 的具体版本号，如 3.1.0



Metric Registries

MetricRegistry类是Metrics的核心，它是存放应用中所有metrics的容器。也是我们使用 Metrics 库的起点。



MetricRegistry registry = new MetricRegistry\(\);

每一个 metric 都有它独一无二的名字，Metrics 中使用句点名字，如 com.example.Queue.size。当你在 com.example.Queue 下有两个 metric 实例，可以指定地更具体：com.example.Queue.requests.size 和 com.example.Queue.response.size 。使用MetricRegistry类，可以非常方便地生成名字。



MetricRegistry.name\(Queue.class, "requests", "size"\)

MetricRegistry.name\(Queue.class, "responses", "size"\)

Metrics 数据展示

Metircs 提供了 Report 接口，用于展示 metrics 获取到的统计数据。metrics-core中主要实现了四种 reporter：JMX, console, SLF4J, 和 CSV。 在本文的例子中，我们使用 ConsoleReporter 。



五种 Metrics 类型

Gauges

最简单的度量指标，只有一个简单的返回值，例如，我们想衡量一个待处理队列中任务的个数，代码如下：



public class GaugeTest {

public static Queue&lt;String&gt; q = new LinkedList&lt;String&gt;\(\);

public static void main\(String\[\] args\) throws InterruptedException {

MetricRegistry registry = new MetricRegistry\(\);

ConsoleReporter reporter = ConsoleReporter.forRegistry\(registry\).build\(\);

reporter.start\(1, TimeUnit.SECONDS\);

registry.register\(MetricRegistry.name\(GaugeTest.class, "queue", "size"\),

new Gauge&lt;Integer&gt;\(\) {

public Integer getValue\(\) {

return q.size\(\);

}

}\);

while\(true\){

Thread.sleep\(1000\);

q.add\("Job-xxx"\);

}

}

}

运行之后的结果如下：



-- Gauges ------------------------------------------------

com.alibaba.wuchong.metrics.GaugeTest.queue.size

value = 6

其中第7行和第8行添加了ConsoleReporter，可以每秒钟将度量指标打印在屏幕上，理解起来会更清楚。



但是对于大多数队列数据结构，我们并不想简单地返回queue.size\(\)，因为java.util和java.util.concurrent中实现的\#size\(\)方法很多都是 O\(n\) 的复杂度，这会影响 Gauge 的性能。



Counters

Counter 就是计数器，Counter 只是用 Gauge 封装了 AtomicLong 。我们可以使用如下的方法，使得获得队列大小更加高效。



public class CounterTest {

public static Queue&lt;String&gt; q = new LinkedBlockingQueue&lt;String&gt;\(\);

public static Counter pendingJobs;

public static Random random = new Random\(\);

public static void addJob\(String job\) {

pendingJobs.inc\(\);

q.offer\(job\);

}

public static String takeJob\(\) {

pendingJobs.dec\(\);

return q.poll\(\);

}

public static void main\(String\[\] args\) throws InterruptedException {

MetricRegistry registry = new MetricRegistry\(\);

ConsoleReporter reporter = ConsoleReporter.forRegistry\(registry\).build\(\);

reporter.start\(1, TimeUnit.SECONDS\);

pendingJobs = registry.counter\(MetricRegistry.name\(Queue.class,"pending-jobs","size"\)\);

int num = 1;

while\(true\){

Thread.sleep\(200\);

if \(random.nextDouble\(\) &gt; 0.7\){

String job = takeJob\(\);

System.out.println\("take job : "+job\);

}else{

String job = "Job-"+num;

addJob\(job\);

System.out.println\("add job : "+job\);

}

num++;

}

}

}

运行之后的结果大致如下：



add job : Job-15

add job : Job-16

take job : Job-8

take job : Job-10

add job : Job-19

15-8-1 16:11:31 ============================================

-- Counters ----------------------------------------------

java.util.Queue.pending-jobs.size

count = 5

Meters

Meter度量一系列事件发生的速率\(rate\)，例如TPS。Meters会统计最近1分钟，5分钟，15分钟，还有全部时间的速率。



public class MeterTest {

public static Random random = new Random\(\);

public static void request\(Meter meter\){

System.out.println\("request"\);

meter.mark\(\);

}

public static void request\(Meter meter, int n\){

while\(n &gt; 0\){

request\(meter\);

n--;

}

}

public static void main\(String\[\] args\) throws InterruptedException {

MetricRegistry registry = new MetricRegistry\(\);

ConsoleReporter reporter = ConsoleReporter.forRegistry\(registry\).build\(\);

reporter.start\(1, TimeUnit.SECONDS\);

Meter meterTps = registry.meter\(MetricRegistry.name\(MeterTest.class,"request","tps"\)\);

while\(true\){

request\(meterTps,random.nextInt\(5\)\);

Thread.sleep\(1000\);

}

}

}

运行结果大致如下：



request

15-8-1 16:23:25 ============================================

-- Meters ------------------------------------------------

com.alibaba.wuchong.metrics.MeterTest.request.tps

count = 134

mean rate = 2.13 events/second

1-minute rate = 2.52 events/second

5-minute rate = 3.16 events/second

15-minute rate = 3.32 events/second

注：非常像 Unix 系统中 uptime 和 top 中的 load。



Histograms

Histogram统计数据的分布情况。比如最小值，最大值，中间值，还有中位数，75百分位, 90百分位, 95百分位, 98百分位, 99百分位, 和 99.9百分位的值\(percentiles\)。



比如request的大小的分布：



public class HistogramTest {

public static Random random = new Random\(\);

public static void main\(String\[\] args\) throws InterruptedException {

MetricRegistry registry = new MetricRegistry\(\);

ConsoleReporter reporter = ConsoleReporter.forRegistry\(registry\).build\(\);

reporter.start\(1, TimeUnit.SECONDS\);

Histogram histogram = new Histogram\(new ExponentiallyDecayingReservoir\(\)\);

registry.register\(MetricRegistry.name\(HistogramTest.class, "request", "histogram"\), histogram\);

while\(true\){

Thread.sleep\(1000\);

histogram.update\(random.nextInt\(100000\)\);

}

}

}

运行之后结果大致如下：



-- Histograms --------------------------------------------

java.util.Queue.queue.histogram

count = 56

min = 1122

max = 99650

mean = 48735.12

stddev = 28609.02

median = 49493.00

75% &lt;= 72323.00

95% &lt;= 90773.00

98% &lt;= 94011.00

99% &lt;= 99650.00

99.9% &lt;= 99650.00

Timers

Timer其实是 Histogram 和 Meter 的结合， histogram 某部分代码/调用的耗时， meter统计TPS。



public class TimerTest {

public static Random random = new Random\(\);

public static void main\(String\[\] args\) throws InterruptedException {

MetricRegistry registry = new MetricRegistry\(\);

ConsoleReporter reporter = ConsoleReporter.forRegistry\(registry\).build\(\);

reporter.start\(1, TimeUnit.SECONDS\);

Timer timer = registry.timer\(MetricRegistry.name\(TimerTest.class,"get-latency"\)\);

Timer.Context ctx;

while\(true\){

ctx = timer.time\(\);

Thread.sleep\(random.nextInt\(1000\)\);

ctx.stop\(\);

}

}

}

运行之后结果如下：



-- Timers ------------------------------------------------

com.alibaba.wuchong.metrics.TimerTest.get-latency

count = 38

mean rate = 1.90 calls/second

1-minute rate = 1.66 calls/second

5-minute rate = 1.61 calls/second

15-minute rate = 1.60 calls/second

min = 13.90 milliseconds

max = 988.71 milliseconds

mean = 519.21 milliseconds

stddev = 286.23 milliseconds

median = 553.84 milliseconds

75% &lt;= 763.64 milliseconds

95% &lt;= 943.27 milliseconds

98% &lt;= 988.71 milliseconds

99% &lt;= 988.71 milliseconds

99.9% &lt;= 988.71 milliseconds

其他

初次之外，Metrics还提供了 HealthCheck 用来检测某个某个系统是否健康，例如数据库连接是否正常。还有Metrics Annotation，可以很方便地实现统计某个方法，某个值的数据。感兴趣的可以点进链接看看。



使用经验总结

一般情况下，当我们需要统计某个函数被调用的频率（TPS），会使用Meters。当我们需要统计某个函数的执行耗时时，会使用Histograms。当我们既要统计TPS又要统计耗时时，我们会使用Timers。



参考资料

Metrics Core

Metrics Getting Started



