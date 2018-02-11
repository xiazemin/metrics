# section5

Metrics核心

翻译自Metrics官方文档: http://metrics.codahale.com/manual/core/



JAVA Metrics是一个用于度量的一个JAVA的类库，使用请参见  &lt; 



Java Metric使用介绍1 &gt; http://blog.csdn.net/scutshuxue/article/details/8350135

或者官方的快速入门：http://metrics.codahale.com/getting-started/

 





在Metrics中最重要的包就是metrics-core，它提供以下几个基本的功能：



l 5种度量的类型：Gauges, Counters, Histograms, Meters,Timers.



l  健康检查\(Health Checks\)



l  可以通过JMX，终端，CSV文件来报告Metrics指标



 



所有的度量类型都是在Metrics或者MetricsRegistry类中，如果你的应用运行在另外一个独立的JVM应用中的话（如多个WARS部署在同一个应用服务上），你应该使用MetricsRegistry这个实例。如果你的应用是单独一个JVM进程的话，你可以使用Metrics中的静态工厂方法。



 



本文档假设你在使用Metrics，并且所有的接口都是一致的。



 



Metric Names-度量名

每一个度量（Metric）都有自己的名字，它包括以下几个内容



l  Group



Metric最上层的分类，如果Metric是一个类的话，那么默认值是这个类所在包的名称（例如：com.example.proj.auth）



l  Type



Metric第二层的名字，如果Metric是一个类的话，那么默认值就是这个类的名字（如SessionStore）



l  Name



描述Metric信息的一个简短的描述



l  Scope



可选的，表示Metric范围的描述信息，当你在一个类中有多个实例的时候会有用



Metrics跟MetricsRegistry中的工厂方法，接受class/name,class/name/scope作为参数调用，或者是使用MetricName这个类进行封装。



 



Gauges

Gauge是最简单的度量类型，只有一个简单的返回值，例如，你的应用中有一个由第三方类库中保持的一个度量值，你可以很容易的通过Gauge来度量他，代码如下：



\[java\] view plain copy

Metrics.newGauge\(SessionStore.class,"cache-evictions", new Gauge&lt;Integer&gt;\(\) {  

    @Override  

    publicInteger value\(\) {  

       return cache.getEvictionsCount\(\);  

    }  

}\);  





那么Metrics会创建一个叫做com.example.proj.auth.SessionStore.cache-evictions的Gauge，返回缓存中Eviction的个数。



 



JMX Gauges

Metrics提供一个JmxGauge类，可以供很多第三方的类库通过JMX来展示度量值，通过Metric的newGauge方法可以初始化他，参数为JMX MBean的Object名和属性名，还有一个继承了Gauge的类，返回值为那个属性的值。



\[java\] view plain copy

Metrics.newGauge\(SessionStore.class,"cache-evictions",  

                new JmxGauge\("net.sf.ehcache:type=Cache,scope=sessions,name=eviction-count","Value"\)\);  

 



Ratio Gauges

Ratio（比率） Gauge是一种计算两个数字之间比例的度量方法



\[java\] view plain copy

public class CacheHitRatio extends RatioGauge {  

    privatefinal Meter hits;  

    privatefinal Timer calls;  

   

    publicCacheHitRatio\(Meter hits, Timer calls\) {  

       this.hits = hits;  

       this.calls = calls;  

    }  

   

    publicdouble getNumerator\(\) {  

       return hits.oneMinuteRate\(\);  

    }  

   

    publicdouble getDenominator\(\) {  

       return calls.oneMinuteRate\(\);  

    }  

}  

这个定制的Gauge返回一个通过Meter跟Timer计算实现的缓存命中率



Percent Gauges

跟Ratio Gauge同样的接口，只是将Ratio Gauge以百分比的方式展现。



Counters

Counter是一个简单64位的计数器：



\[java\] view plain copy

final Counter evictions =Metrics.newCounter\(SessionStore.class, "cache-evictions"\);  

evictions.inc\(\);  

evictions.inc\(3\);  

evictions.dec\(\);  

evictions.dec\(2\);  

所有的Counter都是从0开始



Histograms-直方图

Histrogram是用来度量流数据中Value的分布情况，例如，每一个搜索中返回的结果集的大小：



\[java\] view plain copy

final Histogram resultCounts =Metrics.newHistogram\(ProductDAO.class, "result-counts"\);  

resultCounts.update\(results.size\(\)\);  

Histrogram 的度量值不仅仅是计算最大/小值、平均值，方差，他还展现了分位数（如中位数，或者95th分位数）



传统上，中位数（或者其他分位数）是在一个完整的数据集中进行计算的，通过对数据的排序，然后取出中间值（或者离结束1%的那个数字，来计算99th分位数）。这种做法是在小数据集，或者是批量计算的系统中，但是在一个高吞吐、低延时的系统中是不合适的。



一个解决方案就是从数据中进行抽样，保存一个少量、易管理的数据集，并且能够反应总体数据流的统计信息。使我们能够简单快速的计算给定分位数的近似值。这种技术称作reservoir sampling。



Metrics中提供两种类型的直方图：uniform跟biased。



Uniform Histograms

Uniform Histogram提供直方图完整的生命周期内的有效的中位数，它会返回一个中位值。例如：这个中位数是对所有值的直方图进行了更新（这句话翻译不通），它使用了一种叫做Vitter’s R的算法，随机选择了一些线性递增的样本。



当你需要长期的测量，请使用Uniform Histograms。在你想要知道流数据的分布中是否最近变化的话，那么不要使用这种。



Biased Histograms

Biased Histogram提供代表最近5分钟数据的分位数，他使用了一种forward-decayingpriority sample的算法，这个算法通过对最新的数据进行指数加权，不同于Uniform算法，Biased Histogram体现的是最新的数据，可以让你快速的指导最新的数据分布发生了什么变化。Timers中使用了Biased Histogram。



Meters

Meter度量一系列事件发生的比率：



\[java\] view plain copy

final Meter getRequests =Metrics.newMeter\(WebProxy.class, "get-requests","requests", TimeUnit.SECONDS\);  

getRequests.mark\(\);  

getRequests.mark\(requests.size\(\)\);  

Meter需要除了Name之外的两个额外的信息，事件类型（enent type）跟比率单位（rate unit）。事件类型简单的描述Meter需要度量的事件类型，在上面的例子中，Meter是度量代理请求数，所以他的事件类型也叫做“requests”。比率单位是命名这个比率的单位时间，在上面的例子中，这个Meter是度量每秒钟的请求次数，所以他的单位就是秒。这两个参数加起来就是表述这个Meter，描述每秒钟的请求数。



Meter从几个角度上度量事件的比率，平均值是时间的平均比率，它描述的是整个应用完整的生命周期的情况（例如，所有的处理的请求数除以运行的秒数），它并不描述最新的数据。幸好，Meters中还有其他3个不同的指数方式表现的平均值，1分钟，5分钟，15分钟内的滑动平均值



Hint:这个平均值跟Unix中的uptime跟top中秒数的Load的含义是一致的



 



Timers

Timer是Histogram跟Meter的一个组合：



\[java\] view plain copy

final Timer timer = Metrics.newTimer\(WebProxy.class,"get-requests", TimeUnit.MILLISECONDS, TimeUnit.SECONDS\);  

   

final TimerContext context = timer.time\(\);  

try {  

    // handlerequest  

} finally {  

   context.stop\(\);  

}  

Timer需要的参数处理Name之外还需要，持续时间单位跟比率时间单位，持续时间单位是要度量的时间的期间的一个单位，在上面的例子中，就是MILLISECONDS，表示这段周期内的数据会按照毫秒来进行度量。比率时间单位跟Meters的一致。



注：度量消耗的时间是通过java中高进度的System.nanoTime\(\)方法，他提供的是一种纳秒级别的度量。



Health Checks\(健康检查\)

Meters提供一种一致的、统一的方法来对应用进行健康检查，健康检查是一个基础的对应用是否正常运行的自我检查。



要创建一个Health Check，必须继承HealthChck类：



\[java\] view plain copy

public class DatabaseHealthCheck extends HealthCheck {  

    private final Database database;  

   

    public DatabaseHealthCheck\(Databasedatabase\) {  

        super\("database"\);  

        this.database = database;  

    }  

   

    @Override  

    protected Result check\(\) throws Exception {  

        if \(database.ping\(\)\) {  

            return Result.healthy\(\);  

        }  

        return Result.unhealthy\("Can'tping database"\);  

    }  

}  

在这个例子中，我们对Database类创建了一个健康检查，这个Database是应用中依赖的。我们虚构的Database类中有一个\#ping（）方法，这个方法执行了一个安全的检查语句（例如select 1）。\#ping\(\)如果语句结构正确则返回true，否则返回false，如果出现一个严重的错误则跑出异常。



DatabaseHealthCheck中有一个Database的实例，它有一个\#check\(\)方法尝试去连接数据库，如果连接成功，则返回health结果，如果不成功，则返回unhealthy结果。



在健康检查\#check\(\)中抛出的异常被捕获了，并且伴随着全部堆栈返回不健康结果。



注册一个健康检查，可以是HealthCheck单例也可以是一个HealthCheckRegistry实例：



\[java\] view plain copy

  

\[java\] view plain copy

HealthChecks.register\(newDatabaseHealthCheck\(database\)\);  

也可以注册一串的健康检查：  

for\(Entry&lt;String, Result&gt; entry : HealthChecks.run\(\).entrySet\(\)\) {  

    if \(entry.getValue\(\).isHealthy\(\)\) {  

        System.out.println\(entry.getKey\(\) +": PASS"\);  

    } else {  

        System.out.println\(entry.getKey\(\) +": FAIL"\);  

    }  

}  

Reporters报告

Reporters是将你的应用中所有的度量指标展现出来的一种方式，metrics-core中用了三种方法来导出你的度量指标，JMX，Console跟CSV文件



JMX

默认的，Metrics一直将你的所有指标注册成JMX的MBeans，你可以通过安装了VisualVM-MBeans的VisualVM（大部分JDK自带的jvisualvm）或者Jconsole（大部分JDK自带的jconsole）







提示：你可以双击meteric属性，VisualVM会将数据以图形的方式展现出来。



这种Report必须是JMX一直都是打开的，由于JMX的RPC API是不可靠的，我们不建议你在生产环境中通过JMX来手机度量指标。对于开发者来说，最好是通过网页来浏览，这会非常好用。



Console

对于一些简单的基准，Metrics提供了ConsoleReporter，这个周期性的打印出注册的metric到控制台上。命令如下：



\[java\] view plain copy

ConsoleReporter.enable\(1,TimeUnit.SECONDS\);  

CSV

对于比较复杂的基准，Metrics提供了CsvReporter，他周期性的提供了一连串的给定目录下.csv文件。



\[java\] view plain copy

CsvReporter.enable\(newFile\("work/measurements"\), 1, TimeUnit.SECONDS\);  

上面语句的意思是，每一个Metric指标，都会对应有一个.csv文件创建，每秒钟都会有一行记录被写入到.csv文件中。



OtherReporters

Metrics还有其他的Reporters：



l  MetricsServlet  将你的度量指标以JSon的格式展现的Servlet，他还可以进行检查检查，Dump线程，暴露出有价值的JVM层面跟OS层面的信息



l  GanliaReporter  将度量指标以流式的方式返回给Ganglia服务器



l  GraphiteReporter  将度量指标以流式的方式返回给Graphite服务器

