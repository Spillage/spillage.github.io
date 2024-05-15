# 记一次conc踩过的坑

  最近写了一个项目, 开发语言golang, 有用到gocron框架管理定时任务，同时使用goroutines做并发。因为这么写别的项目非常稳定，而且跑了几个月没有异常和crash，看起来很美好，是吧？然后，使用同样的框架和库，在别的项目上线后碰到了非常奇怪的问题，每几天整个项目会死掉，毫无反应。查看了日志，都是在定时任务执行到一半的情况下卡死。

  于是决定加上prometheus进行监控，也加上了pprof一起分析内存。服务刚部署的前12小时，请求量和goroutines的数量正常，内存也是正常分配使用，心想等事故大爷可能没那么容易，放心地出门了，没多久，一看监控，hoho，granfana面板一点数据都没有了，心知事故来了，迅速拉了pprof一看，为空。

  为什么呢？我们真的是百思不得其解，还好定时任务只是起到刷新数据库数据的作用，但我们数据量不大，更新的数据不多，决定冒着数据落后的风险，先把定时任务的代码注释，试试看。

  在本地调试的时候把服务起来了，又观察了几小时，事故大爷如约而至，再仔细看看log，是执行goroutines的时候日志停留在某条goroutines下，有没有可能是gouroutines的管理有问题？

  正题来了，我们使用的是 "github.com/sourcegraph/conc/pool" 来管理goroutines，类似java里的线程池管理。我们看了conc的介绍，大概比自己动手用官方标准库+通道要方便很多，就决定引入这个第三方库，把事故请进来了。


  ```
  func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	p := pool.New().WithContext(ctx).WithCancelOnError()
	p.Go(func(ctx context.Context) error {
		<-ctx.Done() // exit once context is canceled
		time.Sleep(50 * time.Second)
		fmt.Println("1")
		return ctx.Err()
	})

	fmt.Printf("%s\n", p.Wait())
    }
   ```

   这就是出问题的地方，即使设置了上下文超时，实际执行结果还是等goroutines执行完成，才会退出context，这显然是个大bug，看了作者提供的测试用例，居然是通过？？？ 只好把源码拉下来跑一番，发现了问题，作者的测试代码执行时间超短也就几十毫秒，以至于看起来没问题，非常正常，但按照我们的逻辑，把超时时间设置到10秒的级别，就能看出问题了。

   可惜的是作者似乎不太容易找到，所以暂时没有提issue的计划，我们决定换一个别的第三方库"github.com/alitto/pond"，问题解决。

   ```
   func main() {
	p := pond.New(100, 100)
	p.Submit(func()  {
		time.Sleep(50 * time.Second)
		fmt.Println("1")
	
	})
	p.StopAndWaitFor(10 * time.Second)
	fmt.Printf("%s\n", p.Wait())

    }
    ```