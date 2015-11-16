---
layout: post
title: Qt 多线程解决阻塞
category: Tech
---

这学期选了一门物理实验的小课题「宇宙射线μ子探测」，于是需要给实验用到的程序写个 GUI。因为目标平台是 Windows，何宇就直接去写 WPF 了；而我本身不是 Windows 用户，当然倾向于寻找一个跨平台的解决方案，目前主流的 GUI 框架中 Qt 大概是最优雅的选择（[知乎上也有对此的讨论](http://www.zhihu.com/question/23480014)）。

除了原生的 C++，Qt 也支持其他许多语言的绑定，譬如 PyQt、QtRuby 等等；Qt 近年新推的 Qt Quick 也改用了可以内嵌 JS 的新语言 QML。不过因为我恰好在上 C++ 的面向对象程序设计课，正想借此机会实践一下，所以我还是选择了原生的 C++。

因为实验模拟是个计算密集型的任务，整个计算函数要跑很长时间，如果直接调用它，必然会**阻塞**事件循环。这样一来，GUI 所有的绘制和交互都被阻塞在事件队列中，整个程序就失去响应了。

对于这样的阻塞一般有两种解决办法：一是在计算任务中不停地调用静态成员函数 `QCoreApplication::processEvents()` 来手动运行事件循环，它会在处理完队列中所有事件后返回。不过这样做毕竟没有从根本上解决问题，另外如果两次函数调用之间间隔的时间不够短，用户仍能明显感觉到程序卡顿。

第二种解决办法就是为任务新开一个线程，这样就能在不干扰 GUI 线程的情况下完成计算了。Qt 提供了三种控制线程的方式：QThread、QRunnable / QThreadPool、QtConcurrent，其中最通用、也是最常见的是 QThread。

<!--more-->

**QRunnable** 是一个非常轻量的抽象类，它的主体是纯虚函数 `QRunnable::run()`，我们需要继承它并实现这个函数。使用时需要将其子类的实例放进 **QThreadPool** 的执行队列，线程池默认会在运行后自动删除这个实例。每个应用都有一个全局的线程池，我们可以直接使用它，这就不需要手动创建和管理线程池了。不过因为 QRunnable 不是 QObject 的子类，它没有内建与外界通信的手段，所以真正在使用时没那么实用。

```c++
class Task : public QRunnable
{
public:
    void run()
    {
        /* Implementation */
    }
};

/* ... */
QThreadPool::globalInstance()->start(new Task);
```

而 **QtConcurrent** 不是一个类，而是一个命名空间，内含一系列高度封装的多线程 API，基于前面提到的 QThreadPool。它可以并行地处理 MapReduce / FilterReduce，并能根据处理器实际的核心数调整所用的线程数，在执行过程中可以通过 QFuture 来获取结果、查询运行状态、暂停或取消任务。另外 `QtConcurrent::run(Function function, ...)` 可以启动一个新线程来运行所给的函数（可后接参数表），不过这时返回的 QFuture 不支持暂停和中止。使用这些 API 能完成大多数的多线程任务。

**QThread** 是 Qt 多线程调度中最核心的底层类，也是最常见的多线程实现方法。跟前面两者相比，QThread 的优势在于能够开启线程内的事件循环，为线程中所有 QObject 分发事件，以及能够设置自身的线程优先级。在 Qt 4.4 之前，QThread 跟 QRunnable 一样是一个抽象类，需要在子类中实现 `QThread::run()`，再将其实例化并调用成员函数 `QThread::start()` 即可运行。

```c++
class WorkerThread : public QThread
{
protected:
    void run()
    {
        /* Implementation */
    }
};

/* ... */
WorkerThread *workerThread = new WorkerThread;
workerThread->start();
```

但现在的 Qt 版本中 `QThread::run()` 不再是纯虚函数，其默认实现是调用 `QThread::exec()` 开启一个事件循环。因此继承 QThread 实现多线程已不再是推荐的做法，，**更加优雅**的做法是将计算任务和线程管理分离，即在一个 QObject 中处理任务，并使用 `QObject::moveToThread` 改变其线程关联（thread affinity）。

```c++
class Worker : public QObject
{
    Q_OBJECT

public slots:
    void doWork(const QString &parameter)
    {
        QString result;
        /* Blocking calculations */
        emit resultReady(result);
    }

signals:
    void resultReady(const QString &result);
};
```

```c++
class Controller : public QObject
{
    Q_OBJECT
    QThread workerThread;

public:
    Controller()
    {
        Worker *worker = new worker;
        worker->moveToThread(&workerThread);
        connect(this, &QThread::finished, worker, &QObject::deleteLater);
        connect(this, &Controller::operate, worker, &Worker::doWork);
        connect(worker, &Worker::resultReady, this, &Worker::handleResult);
        workerThread.start();
    }
    ~Controller()
    {
        workerThread.quit();
        workerThread.wait();
    }

public slots:
    void handleResult(const QString &result);

signals:
    void operate(const QString &parameter);
};
```

在新线程中执行计算任务时，我们会发现这时不能再访问 UI 了，这是因为 QWidget 及其子类都不是**可重入的**（reentrant），只能通过主线程访问。这样的设计虽然对开发者来说有些麻烦，但避免了可能导致的死锁或是复杂的 UI 同步。总之若要更新 UI 或是做其他交互，我们需要进行线程间通信。一种方法是调用静态函数 `QMetaObject::invokeMethod()`：

```c++
QMetaObject::invokeMethod(obj, "methodName",
                          Qt::QueuedConnection,
                          Q_RETURN_ARG(QString, retVal),
                          Q_ARG(int, 48));
```

其中 `Qt::QueuedConnection` 意味着向对象所在进程发送事件，进入目标线程的事件循环以待执行。另一种方法则更加简单——使用跨线程的信号槽，`QObject::connect()` 的最后一个参数可以指定连接类型，默认值 `Qt::AutoConnection` 表示如果目标线程就是当前进程则用 `Qt::DirectConnection`，否则采用 `Qt::QueuedConnection` 连接。

另外值得注意的是，如果信号槽的参数类型不是内建数据类型、不属于 QVariant，会抛出错误「QObject::connect: Cannot queue arguments of type '...'」，即该类型的参数无法进入信号队列。这时需要我们在类的声明之后调用宏 `Q_DECLARE_METATYPE(MyClass)`，当然前提是该类提供了公有的构造函数、拷贝构造函数和析构函数，并且要在跨线程通信前使用函数 `qRegisterMetaType<MyClass>("MyClass")` 来注册这个类型。

> 参考文档：
> 
> [Threads Events QObjects - Qt Wiki](http://wiki.qt.io/Threads_Events_QObjects)  
> [QThread Class - Qt Documentation](http://doc.qt.io/qt-5/qthread.html)  
> [QRunnable Class - Qt Documentation](http://doc.qt.io/qt-5/qrunnable.html)  
> [QThreadPool Class - Qt Documentation](http://doc.qt.io/qt-5/qthreadpool.html)  
> [Qt Concurrent - Qt Documentation](http://doc.qt.io/qt-5/qtconcurrent-index.html)
