---
layout: post
title:  "Commons-IO组件:File Monitor监控文件"
date:   2019-11-09 08:38:00
categories: Java 
tags: Commons-IO
---

* content
{:toc}


Commons-IO中提供了监控文件系统事件的组件`File Monitor`，其中包括的事件有目录和文件的创建、修改、删除。实现原理大致为: FileAlterationMonitor中启动一个线程，每隔一定的时间间隔执行所有已注册的FileAlterationObserver对象的checkAndNotify()方法；FileAlterationObserver内部持有
FileAlterationListener(文件监听器)、FileEntry(文件实体状态)、FileFilter(文件过滤器)对象的引用，当对应根目录下的所有`FileEntry`状态发生变化且满足FileFilter中定义的过滤条件时，FileAlterationListener中对应的方法将被回调

`File Monitor`涉及到的相关接口、类如下：



## FileAlterationListener

接收文件系统修改事件的监听器，当监控的目录发生不同的事件时，当前接口中对应的事件方法将会被回调，提供的回调方法如下：




- void onStart(final FileAlterationObserver observer): 文件系统观察者(FileAlterationObserver)对象开始检查事件

- void onDirectoryCreate(File directory): 目录创建事件

- void onDirectoryChange(File directory)： 目录变化事件

- void onDirectoryDelete(final File directory)： 目录删除事件

- void onFileCreate(final File file): 文件创建事件

- void onFileChange(final File file): 文件变化事件

- void onFileDelete(final File file): 文件删除事件

- void onStop(final FileAlterationObserver observer): 文件系统观察者对象完成检查事件


## FileAlterationListenerAdaptor

提供便捷的实现FileAlterationListener接口实现，内部不做任何处理

## FileAlterationMonitor

内部维护一个监视线程，在指定间隔时间内触发任何已注册的文件系统观察者(FileAlterationObserver)对象，提供的构造方法如下：

```java
/**
 * Construct a monitor with a default interval of 10 seconds. 
 */
public FileAlterationMonitor() {
    this(10000);
}

/**
 * Construct a monitor with the specified interval.
 *
 * @param interval The amount of time in milliseconds to wait between
 * checks of the file system
 */
public FileAlterationMonitor(final long interval) {
    this.interval = interval;
}

/**
 * Construct a monitor with the specified interval and set of observers.
 *
 * @param interval The amount of time in milliseconds to wait between
 * checks of the file system
 * @param observers The set of observers to add to the monitor.
 */
public FileAlterationMonitor(final long interval, final FileAlterationObserver... observers) {
    this(interval);
    if (observers != null) {
        for (final FileAlterationObserver observer : observers) {
            addObserver(observer);
        }
    }
}
```

## FileAlterationObserver

代表根目录下所有文件的状态，检查文件系统和通知监听器创建、更改或删除事件。FileAlterationObserver中提供了文件过滤的功能: 当满足指定条件的文件或目录时，才会回调FileAlterationListener中实现的方法


## FileEntry

文件或目录的状态，在某个时间点捕获以下文件的属性：

- File Name: 文件名

- Exists: 文件或目录是否存在

- Directory: 是否为目录 

- Last Modified Date/Time: 最后修改时间

- Length: 文件长度，目录为0

- Children: 目录内容



## 使用示例：

监控、实时读取sysconfig目录下的img-config.properties图片路径配置文件

```java
public class MonitorImgConfigFile {

    public static final String CONFIG_DIRECT_ROOT = MonitorImgConfigFile.class.getClassLoader().getResource("").getPath() + "/sysconfig/";

    public static final String CONFIG_FILE_NAME = "img-config.properties";

    private final FileFilter fileFilter = pathname -> pathname.getName().equals(CONFIG_FILE_NAME);

    private final FileAlterationObserver observer = new FileAlterationObserver(CONFIG_DIRECT_ROOT, fileFilter);

    private final FileAlterationMonitor monitor = new FileAlterationMonitor(TimeUnit.SECONDS.toMillis(10)
            , observer);

    private final FileAlterationListener listener = new FileAlterationListenerAdaptor() {

        @Override
        public void onFileChange(File file) {
            try {
                String contents = FileUtils.readFileToString(file, Charset.forName("UTF-8"));
                System.out.println("change contents:" + contents);
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
    };

    public void startMinitor() throws Exception {
        observer.addListener(listener);
        monitor.start();
    }

    public void stopMinitor() throws Exception {
        monitor.stop();
    }

    public static void main(String[] args) throws Exception {
        MonitorImgConfigFile monitorImgConfigFile = new MonitorImgConfigFile();
        monitorImgConfigFile.startMinitor();
    }

}
```





