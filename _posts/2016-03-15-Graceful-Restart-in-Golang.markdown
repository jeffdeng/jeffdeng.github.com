---
layout: post
title:  "Golang如何做热重启的关键点"
date:   2016-03-16 12:00:00
categories:  🐭go

---

* content
{:toc}

## 什么是热重启:

新老程序（进程）无缝替换，同时可以保持对client的服务。让client端感觉不到你的服务挂掉了。
比如重新加载配置文件，需要重启一下，替换老程序需要重启一下，就需要用到热重启。

## 原理

1.通过发送signal(信号)与进程间交互。
信号可以自己定义，指定拦截系统的信号，改变系统默认行为来自定义操作。

如果收到重启信号，启用新版本的进程 将socket句柄交给新进程，新进程开始接受新连接请求 旧版本服务器停止接受连接，要保持已有的连接，处理完毕后立即停止旧版本服务器,关掉监听（os.Exit(1)）。

## 简化版的重启过程（不处理原有的连接，方便理解关键点）

### 生成一个server监听8888端口，然后拦截系统信号，获取监听器

获取监听器，这里用一个函数来获取，因为父进程和子进程获取到的监听器不一样，父进程获取到的是原来的tcp socket文件，而子进程获取到的只是这个 socket文件的copy。那么，要是你两次都获取tcp socket文件，相当于2次监听同一个端口，会报这个错误`listen tcp :8888: bind: address already in use`.所以，第二次获取的副本就相当于继承了父监听器。这个是实现热重启的关键点。


    func main() {

        server = http.Server{
            Addr:        ":8888",
            Handler:     &MyHandle{},
            ReadTimeout: 6 * time.Second,
        }

        go handleSignals()

        log.Printf("Actual pid is %d\n", syscall.Getpid())

        var err2 error

        listener, err2 = getListener(server.Addr)

        if err2 != nil {
            log.Println(err2)

        }

        log.Printf("isChild : %v ,listener: %v\n", isChild, listener)

        err := server.Serve(listener)

        if err != nil {
            log.Println(err)
        }
    }


### 获取socket监听器和继承socket监听器

isChild就是子进程的标志，假如是子进程那么` f := os.NewFile(3, "")`,为什么是3呢，而不是1 0 或者其他数字？这是因为父进程里给了个fd给子进程了 而子进程里0，1，2是预留给 标准输入、输出和错误的，所以父进程给的第一个fd在子进程里顺序排就是从3开始了；如果fork的时候cmd.ExtraFiles给了两个文件句柄，那么子进程里还可以用4开始，就看你开了几个子进程自增就行。因为我这里就开一个子进程所以把3写死了。`l, err = net.FileListener(f)`这一步只是把 fd描述符包装进`TCPListener`这个结构体。


    func getListener(laddr string) (l net.Listener, err error) {
            if isChild {

                runningServerReg.RLock()
                defer runningServerReg.RUnlock()

                f := os.NewFile(3, "")

                l, err = net.FileListener(f)
                if err != nil {
                    log.Printf("net.FileListener error:", err)
                    return
                }

                log.Printf("laddr : %v ,listener: %v \n", laddr, l)

                syscall.Kill(syscall.Getppid(), syscall.SIGTSTP) //干掉父进程

            } else {
                l, err = net.Listen("tcp", laddr)
                if err != nil {
                    log.Printf("net.Listen error: %v", err)
                    return
                }
            }
            return
    }


### 信号处理

 用signal包的signal.Notify过滤掉自己要定义的信号，给这些信号放行。SIGHUP用来fork子进程，SIGTSTP用来干掉父进程。SIGINT就用来自己CTR+C用了，方便中断测试。
    
    func handleSignals() {
        var sig os.Signal

        signal.Notify(
            sigChan,
            hookableSignals...,
        )

        pid := syscall.Getpid()

        for {
            sig = <-sigChan
            log.Println(pid, "Received SIG.", sig)
            switch sig {
            case syscall.SIGHUP:
                log.Println(pid, "Received SIGHUP. forking.")
                err := fork()
                if err != nil {
                    log.Println("Fork err:", err)
                }
            case syscall.SIGTSTP:
                log.Println(pid, "Received SIGTSTP.")
                shutdown()
            default:
                log.Printf("Received %v \n", sig)
            }

        }
    }

### fork子进程

    func fork() (err error) {
            runningServerReg.Lock()

            defer runningServerReg.Unlock()

            if runningServersForked {
                return errors.New("Another process already forked. Ignoring this one.")
            }

            runningServersForked = true

            log.Println("Restart: forked Start....")

            tl := listener.(*net.TCPListener)
            fl, _ := tl.File()

            path := os.Args[0]
        
            cmd := exec.Command(path, []string{"-continue"})
            cmd.Stdout = os.Stdout
            cmd.Stderr = os.Stderr
            cmd.ExtraFiles = []*os.File{fl} //继承的监听文件

            err = cmd.Start() //开始fork，不是立刻执行
            if err != nil {
                log.Printf("Restart: Failed to launch, error: %v", err)
            }

            return
    }

还有一种方式去fork，和上面本质一样：

    execSpec := &syscall.ProcAttr{
        Env:   os.Environ(),
        Files: []uintptr{os.Stdin.Fd(), os.Stdout.Fd(), os.Stderr.Fd(), lFd},
    }
    pid, err := syscall.ForkExec(os.Args[0], os.Args, execSpec)



### 关闭父进程
 
 其实一个先检查还有没有活动的连接，得先服务完毕才能关掉，用sync.WaitGroup.wait()去阻塞住就是很好的方法，我这里为了演示fork部分就不写这些了。

    func shutdown() {
        log.Printf("shutdown Listener :%v\n", listener)
        err := listener.Close()

        if err != nil {
            log.Println(syscall.Getpid(), "Listener.Close() error:", err)
        } else {
            log.Println(syscall.Getpid(), server.Addr, "Listener closed.")
        }

        os.Exit(1)
    }

### flag包的妙用

用flag包可以自动解析各种命令行参数，比自己去分析字符串简单多了，一定要` flag.Parse()`,这样才开始解析。

    func init() {
        flag.BoolVar(&isChild, "continue", false, "listen on open fd (after forking)")
        flag.Parse()
     }

### 源代码已上传：`https://github.com/jeffdeng/gracefullDemo`
要开箱即用的话，可以用这个`https://github.com/tim1020/godaemon`或者这个`https://github.com/fvbock/endless`，当然明白原理才是最重要的。


参考：
http://my.oschina.net/tim8670/blog/643966
http://grisha.org/blog/2014/06/03/graceful-restart-in-golang/