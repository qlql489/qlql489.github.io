---

title:SimpleDateFormat与DecimalFormat的并发安全
date: 2018-11-23 01:02:38
tags:
	- java
categories:
	- 程序人生
---



![](http://image.nianlun.tech/2022/02/16/1be09268426216f4b6ce9f25dd93961f.jpg)

java在做日期转换时我们会使用SimpleDateFormat做时间转换，但其实SimpleDateFormat不是线程安全的，如果SimpleDateFormat用static声明或只实例化一次被多个线程使用，并发度高时就会出并发异常，看如下例子

```java
public static void main(String[] args) {
        SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        List<String> lists = new ArrayList<String>() {
            {
                add("2018-11-22 01:11:11");
                add("2018-11-22 02:22:22");
                add("2018-11-22 03:33:33");
                add("2018-11-22 04:44:44");
                add("2018-11-22 03:55:55");
                add("2018-11-22 04:55:56");
            }
        };
        ExecutorService executorService = Executors.newCachedThreadPool();
        //用CountDownLatch增加并发度
        CountDownLatch countDownLatch = new CountDownLatch(lists.size());
        for (String list : lists) {
            executorService.submit(() -> {
                try {
                    countDownLatch.await();
                    Date parse = format.parse(list);
                    System.out.println(parse.toString());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
            countDownLatch.countDown();
        }


    }
```
运行日志如下，可能每次报错不一样
```java
java.lang.NumberFormatException: multiple points
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at java.text.DigitList.getDouble(DigitList.java:169)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2056)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at thread.aqs.SimpleDateFormateTest.lambda$main$0(SimpleDateFormateTest.java:42)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
java.lang.NumberFormatException: For input string: "E2E2."
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:2043)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at java.text.DigitList.getDouble(DigitList.java:169)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2056)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:2162)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at thread.aqs.SimpleDateFormateTest.lambda$main$0(SimpleDateFormateTest.java:42)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
java.lang.NumberFormatException: For input string: "E2E"
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:2043)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at java.text.DigitList.getDouble(DigitList.java:169)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2056)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:2162)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at thread.aqs.SimpleDateFormateTest.lambda$main$0(SimpleDateFormateTest.java:42)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
java.lang.NumberFormatException: For input string: "E.4220118E4"
```
或者日期错乱的结果
```java
Thu Nov 22 04:55:56 CST 2018
Sat Nov 30 02:22:22 CST 2024
Tue Nov 30 01:11:11 CST 1
Thu Nov 22 04:55:56 CST 2018
```
### 原因
有两个地方存在并发问题

#### 1、Calendar的共享
SimpleDateFormat的父类持有一个对象Calendar
parse方法流程是这样的：

```java
//解析字符串
...
//清空Calendar
Calendar.clear();
//循环设置每个field
...
```
所以在多线程中会出现A线程设置Calendar值后被B线程修改，
或者A和B线程同时设置Calendar的值，都会出现错误

#### 看一下源码
SimpleDateFormat继承DateFormat
```java
public class SimpleDateFormat extends DateFormat
```
##### 入口
```java
    public Date parse(String source) throws ParseException
    {
        ParsePosition pos = new ParsePosition(0);
        Date result = parse(source, pos);
        if (pos.index == 0)
            throw new ParseException("Unparseable date: \"" + source + "\"" ,
                pos.errorIndex);
        return result;
    }
```
##### parse中解析字符串后的一部分
```java
        Date parsedDate;
        try {
            //出问题的方法
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
```
##### calb.establish方法

```java
Calendar establish(Calendar cal) {
        boolean weekDate = isSet(WEEK_YEAR)
                            && field[WEEK_YEAR] > field[YEAR];
        if (weekDate && !cal.isWeekDateSupported()) {
            // Use YEAR instead
            if (!isSet(YEAR)) {
                set(YEAR, field[MAX_FIELD + WEEK_YEAR]);
            }
            weekDate = false;
        }
        //清空
        cal.clear();
        for (int stamp = MINIMUM_USER_STAMP; stamp < nextStamp; stamp++) {
            for (int index = 0; index <= maxFieldIndex; index++) {
                if (field[index] == stamp) {
                    //设置field
                    cal.set(index, field[MAX_FIELD + index]);
                    break;
                }
            }
        }

        if (weekDate) {
            int weekOfYear = isSet(WEEK_OF_YEAR) ? field[MAX_FIELD + WEEK_OF_YEAR] : 1;
            int dayOfWeek = isSet(DAY_OF_WEEK) ?
                                field[MAX_FIELD + DAY_OF_WEEK] : cal.getFirstDayOfWeek();
            if (!isValidDayOfWeek(dayOfWeek) && cal.isLenient()) {
                if (dayOfWeek >= 8) {
                    dayOfWeek--;
                    weekOfYear += dayOfWeek / 7;
                    dayOfWeek = (dayOfWeek % 7) + 1;
                } else {
                    while (dayOfWeek <= 0) {
                        dayOfWeek += 7;
                        weekOfYear--;
                    }
                }
                dayOfWeek = toCalendarDayOfWeek(dayOfWeek);
            }
            cal.setWeekDate(field[MAX_FIELD + WEEK_YEAR], weekOfYear, dayOfWeek);
        }
        return cal;
    }
```

#### 2、DecimalFormat对象不是线程安全的
SimpleDateFormat持有对象DecimalFormat，而DecimalFormat也不是线程安全的，所以会报上面的那些错

##### parse方法
DecimalFormat对象持有一个DigitList对象，用来标记解析状态
```java
private transient DigitList digitList = new DigitList();
```

```java
    @Override
    public Number parse(String text, ParsePosition pos) {
        // special case NaN
        if (text.regionMatches(pos.index, symbols.getNaN(), 0, symbols.getNaN().length())) {
            pos.index = pos.index + symbols.getNaN().length();
            return new Double(Double.NaN);
        }

        boolean[] status = new boolean[STATUS_LENGTH];
        //这里subparse方法清空和组装digitList,多线程并发会出问题
        if (!subparse(text, pos, positivePrefix, negativePrefix, digitList, false, status)) {
            return null;
        }
        ...
```


### 如何解决
每次使用时都new一个SimpleDateFormat当然可以解决这个问题，但频繁创建销毁对象效能不高，方法上加锁又会降低并发度

因为每个线程自己执行肯定是按顺序执行，所以可以利用ThreadLocal
```java
    public class DateUtil  {

    private static ThreadLocal<DateFormat> threadLocal = ThreadLocal.withInitial(()->{
        return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    });

    public static Date parse(String dateStr) throws ParseException {
        return threadLocal.get().parse(dateStr);
    }

    public static String format(Date date) {
        return threadLocal.get().format(date);
    }

}
```
每个线程持有自己的SimpleDateFormat，再跑测试代码，发现不报错了
```java
//输出
Thu Nov 22 01:11:11 CST 2018
Thu Nov 22 04:55:56 CST 2018
Thu Nov 22 03:33:33 CST 2018
Thu Nov 22 03:55:55 CST 2018
Thu Nov 22 02:22:22 CST 2018
Thu Nov 22 04:44:44 CST 2018
```
#### 使用jodatime

```java
      String date = "2018-11-22 02:22:22";
      DateTime dateTime =  DateTime.parse(date, DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss"));
```
#### 或者使用java8的LocalDateTime
```java
      String str = "1986-04-08 12:30:22";
      DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
      LocalDateTime dateTime = LocalDateTime.parse(str, formatter);
```