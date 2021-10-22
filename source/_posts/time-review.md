---
title: 关于时间的思考和总结
date: 2021-10-22 16:09:55
tags:
  - java
categories:
  - interview
---
## 关于时间的解释

### GMT时间
格林尼治平时(Greenwich Mean Time，GMT)，又称为格林尼治标准时间。
格林尼治平时的正午是指当平太阳横穿格林尼治子午线时（也就是在格林尼治上空最高点时）的时间。自1924年2月5日开始，格林尼治天文台负责每隔一小时向全世界发放调时信息。由于地球每天的自转是有些不规则的，而且正在缓慢减速，因此格林尼治平时基于天文观测本身的缺陷，已经被原子钟报时的协调世界时（UTC）所取代。

### UT时间
世界时(Universal Time，UT)，是一种以格林尼治子夜起算的平太阳时。
由于1925年以前人们在天文观测中，常常把每天的起始（0时）定为正午，而不是通常民用的午夜，给格林尼治平时的意义造成含糊，人们使用世界时一词来明确表示每天从午夜开始的格林尼治平时。

### 时区
时区是指地球上的某一个区域使用同一个时间定义。GMT时间或者UT时间，都是表示地球自转速率的一种形式。从太阳升起到太阳落下，时刻从0到24变化。这样，不同经度的地方时间自然会不相同。为了解决这个问题，人们把地球按经度划分为不同的区域，每个区域内使用同一个时间定义，相邻的区域时间差为1个小时。时区又分为理论时区和法定时区。

### UTC时间
协调世界时(Coordinated Universal Time)。是主要的世界时间标准，以原子钟所定义的秒长为基础，在时刻上尽量接近GMT时间。UTC时间认为一个太阳日总是86400秒。在大多数情况下，UTC时间能与GMT时间互换。

#### UTC与时区
本初子午线所在的时区的时间后面加上字符Z，表示UTC时间。Z即为0时区的标志，读做Zulu。例如09:30 UTC就写作0930Z，14:45:15 UTC则为14:45:15Z或144515Z

#### UTC偏移量
UTC偏移量用以下形式表示: ±[hh]:[mm]、±[hh][mm]、或者±[hh]。例如UTC时间为09:30z，此时北京时间就是1730 +0800，纽约时间是0430 -0500。
UTC时间表示的格式一般为Sat, 20 May 2018 12:45:57 +0800表示东八区(北京时间)2018年5月20号 12:45:57星期六。

#### 时差
某个地方的时刻与0时区的时刻差称为时差，时差东正西负。以本初子午线为中心，每向东一跨过一个时区，时刻增加一个小时，每向西跨过一个时区，时刻减少一个小时。
+ 如何理解向东时区增加
> 由于地球是自西向东转，在地球的某一个地方观察，东边的时间比西边的时间早(东边的人们先看到太阳升起)。
想象一下某一个时刻，太阳在你的正上空，此时你所在的地点的时间为正午12点。这时住在你东边的人们，他们看到太阳已经在西边了，他们的时刻是下午，所以往东，时刻增加。

#### UTC时间与本地时间的转换
UTC时间 + 时差 = 本地时间

### CST时间
CST (China Standard Time，中国标准时间) 是UTC+8时区的知名名称之一，比UTC（协调世界时)提前8个小时与UTC的时间偏差可写为+08:00.
## http协议里respond的header日期
![avatar](/images/time_knowledge/4.png)
> All HTTP date/time stamps MUST be represented in Greenwich Mean Time (GMT), without exception.
格林尼治标准时间。 在HTTP协议中，时间都是用格林尼治标准时间来表示的，而不是本地时间。
[RFC 7231, section 7.1.1.2: Date](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.1.2)

## 在java里用到的时间

![avatar](/images/time_knowledge/2.png)
### SimpleDateFormat工具

+ 例子如下
```
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class UTCTimeFormatTest {
    public static void main(String[] args) throws ParseException {
        //Z代表UTC统一时间:2017-11-27T03:16:03.944Z
        SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'");
        Date date = new Date();
        System.out.println(date);
        String str = format.format(date);
        System.out.println(str);
        SimpleDateFormat dayformat = new SimpleDateFormat("yyyy-MM-dd");
        String source ="2018-09-18";        //先将年月日的字符串日期格式化为date类型
        Date day = dayformat.parse(source);　　　　 //然后将date类型的日期转化为UTC格式的时间
        String str2= format.format(day);
        System.out.println(str2);
    }
}
```
打印结果
![avatar](/images/time_knowledge/1.png)
### SimpleDateFormat线程不安全
#### 例子
```
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SimpleDateFormatTest extends Thread {
    private static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
    private String name;
    private String dateStr;

    public SimpleDateFormatTest(String name, String dateStr) {
        this.dateStr = dateStr;
        this.name = name;
    }

    @Override
    public void run() {
        try {
            Date date = simpleDateFormat.parse(dateStr);
            System.out.println(name + ": date : " + date);
        } catch (ParseException exception) {
            exception.printStackTrace();
        }
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        executorService.execute(new SimpleDateFormatTest("A", "2017-01-01"));
        executorService.execute(new SimpleDateFormatTest("B", "2020-12-12"));
        executorService.shutdown();
    }
}

```
运行结果
![avatar](/images/time_knowledge/3.png)
#### 原理解释
+ SimpleDateFormat构造函数
```
public SimpleDateFormat(String pattern, Locale locale)
{
    if (pattern == null || locale == null) {
        throw new NullPointerException();
    }

    initializeCalendar(locale);
    this.pattern = pattern;
    this.formatData = DateFormatSymbols.getInstanceRef(locale);
    this.locale = locale;
    initialize(locale);
}

```
+ initializeCalendar初始化calendar方法
```
private void initializeCalendar(Locale loc) {
    if (calendar == null) {
        assert loc != null;
        // The format object must be constructed using the symbols for this zone.
        // However, the calendar should use the current default TimeZone.
        // If this is not contained in the locale zone strings, then the zone
        // will be formatted using generic GMT+/-H:MM nomenclature.
        calendar = Calendar.getInstance(TimeZone.getDefault(), loc);
    }
}
```
+ calendar变量的构造过程
```
public static Calendar getInstance(TimeZone zone, Locale aLocale)
{
    return createCalendar(zone, aLocale);
}

private static Calendar createCalendar(TimeZone zone, Locale aLocale)
{
    CalendarProvider provider =
        LocaleProviderAdapter.getAdapter(CalendarProvider.class, aLocale)
                                .getCalendarProvider();
    if (provider != null) {
        try {
            return provider.getInstance(zone, aLocale);
        } catch (IllegalArgumentException iae) {
            // fall back to the default instantiation
        }
    }

    Calendar cal = null;

    if (aLocale.hasExtensions()) {
        String caltype = aLocale.getUnicodeLocaleType("ca");
        if (caltype != null) {
            switch (caltype) {
            case "buddhist":
            cal = new BuddhistCalendar(zone, aLocale);
                break;
            case "japanese":
                cal = new JapaneseImperialCalendar(zone, aLocale);
                break;
            case "gregory":
                cal = new GregorianCalendar(zone, aLocale);
                break;
            }
        }
    }
    if (cal == null) {
        // If no known calendar type is explicitly specified,
        // perform the traditional way to create a Calendar:
        // create a BuddhistCalendar for th_TH locale,
        // a JapaneseImperialCalendar for ja_JP_JP locale, or
        // a GregorianCalendar for any other locales.
        // NOTE: The language, country and variant strings are interned.
        if (aLocale.getLanguage() == "th" && aLocale.getCountry() == "TH") {
            cal = new BuddhistCalendar(zone, aLocale);
        } else if (aLocale.getVariant() == "JP" && aLocale.getLanguage() == "ja"
                    && aLocale.getCountry() == "JP") {
            cal = new JapaneseImperialCalendar(zone, aLocale);
        } else {
            cal = new GregorianCalendar(zone, aLocale);
        }
    }
    return cal;
}
```
> 通过查看源码发现，原来SimpleDateFormat类内部有一个Calendar对象引用,它用来储存和这个SimpleDateFormat相关的日期信息,例如sdf.parse(dateStr),sdf.format(date) 诸如此类的方法参数传入的日期相关String,Date等等, 都是交由Calendar引用来储存的.这样就会导致一个问题,如果你的SimpleDateFormat是个static的, 那么多个thread 之间就会共享这个SimpleDateFormat, 同时也是共享这个Calendar引用。

#### 正确姿势
+ 将SimpleDateFormat定义成局部变量
> 缺点是每次调用方法后都会实例化一个SimpleDateFormat对象，方法结束后会被垃圾回收
+ 如果要定义成静态变量，一定要加锁，保证同一个时刻就只有一个线程可以访问到SimpleDateFormat对象
> 缺点是性能变差，每次都得等待锁释放后其他线程才能访问SimpleDateFormat对象
+ 使用ThreadLocal来保存SimpleDateFormat对象，每个线程拥有自己的SimpleDateFormat对象
```
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

public class DateUtils {
    // 单例
    private static Map<String, ThreadLocal<SimpleDateFormat>> localMap = new HashMap<>();
    private static final Object lockObject = new Object();

    public static SimpleDateFormat getSimpleDateFormat(String pattern) {
        ThreadLocal<SimpleDateFormat> threadLocal = localMap.get(pattern);
        if (threadLocal == null) {
            // 加锁，不重复初始化已经存在的pattern
            synchronized (lockObject) {
                // 再取一次是为了防止localMap被重复多次put已存在的pattern
                threadLocal = localMap.get(pattern);
                if (threadLocal == null) {
                    System.out.println("put new sdf of pattern " + pattern + " to map");
                    threadLocal = new ThreadLocal<SimpleDateFormat>() {
                        @Override
                        protected SimpleDateFormat initialValue() {
                            System.out.println("thread: " + Thread.currentThread() + " init pattern: " + pattern);
                            return new SimpleDateFormat(pattern);
                        }
                    };
                    localMap.put(pattern, threadLocal);
                }
            }
        }
        return threadLocal.get();
    }

    public static String format(Date date, String pattern) {
        return getSimpleDateFormat(pattern).format(date);
    }

    public static Date parse(String dateString, String pattern) throws ParseException {
        return getSimpleDateFormat(pattern).parse(dateString);
    }
}

```
#### ThreadLocal解析
![avatar](/images/time_knowledge/5.png)
> ThreadLocal是用哈希表实现的，每个线程里都有一个ThreadLocalMap，就是以Map的形式存储多个ThreadLocal对象，当线程调用ThreadLocal操作方法时，都会通过当前线程Thread对象拿到ThreadLocalMap，再通过ThreadLocal对象从ThreadLocalMap中锁定数据实体（ThreadLocal.Entry）

+ ThreadLocal.set方法
```
public void set(T value) {
    // 取出当前线程
    Thread t = Thread.currentThread();
    // 取出Thread.ThreadLocal.ThreadLocalMap threadLocals = null;
    ThreadLocalMap map = getMap(t);
    // 已存在，则更新
    if (map != null)
        map.set(this, value);
    else
    // 否则创建map
        createMap(t, value);
}
```
+ ThreadLocalMap.set方法
```
/**
* Set the value associated with key.
*
* @param key the thread local object
* @param value the value to be set
*/
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
+ ThreadLocal.createMap方法
```
/**
* Create the map associated with a ThreadLocal. Overridden in
* InheritableThreadLocal.
*
* @param t the current thread
* @param firstValue value for the initial entry of the map
*/
void createMap(Thread t, T firstValue) {
    // 实例化一个新ThreadLocalMap对象
    // this就是操作的ThreadLocal对象，firstValue就是要保存的值
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### SimpleDateFormat挖坑自跳
#### 坑王举例
```
import java.text.ParseException;
import java.text.SimpleDateFormat;

public class SimpleDateFormatErrorTest {
    public static void main(String[] args) {
        try {
            String date1 = "2021-05-01";
            SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyyMMdd");
            System.out.println(simpleDateFormat.parse(date1));
        } catch (ParseException exception) {
            exception.printStackTrace();
        }
    }
}
```
打印结果如下
![avatar](/images/time_knowledge/6.png)
这个代码预期是会走到catch的exception里，但是却正常打印输出了
#### SimpleDateFormat.parse源码解析
```
@Override
public Date parse(String text, ParsePosition pos)
{
    checkNegativeNumberExpression();

    int start = pos.index;
    int oldStart = start;
    int textLength = text.length();

    boolean[] ambiguousYear = {false};

    CalendarBuilder calb = new CalendarBuilder();

    for (int i = 0; i < compiledPattern.length; ) {
        int tag = compiledPattern[i] >>> 8;
        int count = compiledPattern[i++] & 0xff;
        if (count == 255) {
            count = compiledPattern[i++] << 16;
            count |= compiledPattern[i++];
        }

        switch (tag) {
        case TAG_QUOTE_ASCII_CHAR:
            if (start >= textLength || text.charAt(start) != (char)count) {
                pos.index = oldStart;
                pos.errorIndex = start;
                return null;
            }
            start++;
            break;

        case TAG_QUOTE_CHARS:
            while (count-- > 0) {
                if (start >= textLength || text.charAt(start) != compiledPattern[i++]) {
                    pos.index = oldStart;
                    pos.errorIndex = start;
                    return null;
                }
                start++;
            }
            break;
        // 进入默认配置
        default:
            // Peek the next pattern to determine if we need to
            // obey the number of pattern letters for
            // parsing. It's required when parsing contiguous
            // digit text (e.g., "20010704") with a pattern which
            // has no delimiters between fields, like "yyyyMMdd".
            boolean obeyCount = false;

            // 在阿拉伯语中，负数的减号可以放在数字的后面（1111-）
            // In Arabic, a minus sign for a negative number is put after
            // the number. Even in another locale, a minus sign can be
            // put after a number using DateFormat.setNumberFormat().
            // If both the minus sign and the field-delimiter are '-',
            // subParse() needs to determine whether a '-' after a number
            // in the given text is a delimiter or is a minus sign for the
            // preceding number. We give subParse() a clue based on the
            // information in compiledPattern.
            boolean useFollowingMinusSignAsDelimiter = false;

            if (i < compiledPattern.length) {
                int nextTag = compiledPattern[i] >>> 8;
                if (!(nextTag == TAG_QUOTE_ASCII_CHAR ||
                        nextTag == TAG_QUOTE_CHARS)) {
                    obeyCount = true;
                }

                if (hasFollowingMinusSign &&
                    (nextTag == TAG_QUOTE_ASCII_CHAR ||
                        nextTag == TAG_QUOTE_CHARS)) {
                    int c;
                    if (nextTag == TAG_QUOTE_ASCII_CHAR) {
                        c = compiledPattern[i] & 0xff;
                    } else {
                        c = compiledPattern[i+1];
                    }

                    if (c == minusSign) {
                        useFollowingMinusSignAsDelimiter = true;
                    }
                }
            }
            start = subParse(text, start, tag, count, obeyCount,
                                ambiguousYear, pos,
                                useFollowingMinusSignAsDelimiter, calb);
            if (start < 0) {
                pos.index = oldStart;
                return null;
            }
        }
    }

    // At this point the fields of Calendar have been set.  Calendar
    // will fill in default values for missing fields when the time
    // is computed.

    pos.index = start;

    Date parsedDate;
    try {
        parsedDate = calb.establish(calendar).getTime();
        // If the year value is ambiguous,
        // then the two-digit year == the default start year
        if (ambiguousYear[0]) {
            if (parsedDate.before(defaultCenturyStart)) {
                parsedDate = calb.addYear(100).establish(calendar).getTime();
            }
        }
    }
    // An IllegalArgumentException will be thrown by Calendar.getTime()
    // if any fields are out of range, e.g., MONTH == 17.
    catch (IllegalArgumentException e) {
        pos.errorIndex = start;
        pos.index = oldStart;
        return null;
    }

    return parsedDate;
}
```
+ 正确姿势
```
import java.text.ParseException;
import java.text.SimpleDateFormat;

public class SimpleDateFormatErrorTest {
    public static void main(String[] args) {
        String date1 = "2021-05-01";
        try {
            SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyyMMdd");
            // 设置为严格模式,Calendar类默认把lenient设置为true，意思是宽松模式解析
            simpleDateFormat.setLenient(false);
            System.out.println(simpleDateFormat.parse(date1));
        } catch (ParseException exception) {
            exception.printStackTrace();
        }
    }
}
```
运行结果，看到已经抛出异常了
![avatar](/images/time_knowledge/7.png)
#### 坑爹举例
+ 代码如下
```
import java.text.ParseException;
import java.text.SimpleDateFormat;

public class SimpleDateFormatErrorTest {
    public static void main(String[] args) {
        String date1 = "2021-05-01";
        try {
            SimpleDateFormat simpleDateFormat = new SimpleDateFormat("YYYY-MM-dd");
            // 设置为严格模式
            simpleDateFormat.setLenient(false);
            System.out.println(simpleDateFormat.parse(date1));
        } catch (ParseException exception) {
            exception.printStackTrace();
        }
    }
}
```
打印结果，输出是2020-12-27
![avatar](/images/time_knowledge/7.png)
+ 详解
[官方文档](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html)  

| 字母 | 日期含义 | 举例 |
| :-----| ----: | :----: |
| y | year | 正常年份 |
| Y | week year | 按周算的年份，比如2018年12月31日，正好是2019 Week-year的第一周第一天 |
| D | Day in year | 一年中的第几天 |
| d | Day in month | 正常日期的日 |

