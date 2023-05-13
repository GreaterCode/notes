[github](https://so.csdn.net/so/search?q=github&spm=1001.2101.3001.7020): [https://github.com/containerd/containerd](https://github.com/containerd/containerd)

![image](assets/image-20221227023859-xo28izv.png)

# 1. 前言

* dockerd 是 docker engine 守护进程，dockerd 启动时会启动 containerd 子进程，dockerd 与 containerd 通过 rpc 进行通信
* ctr 是 containerd 的 cli
* containerd 通过 shim 操作 runc，runc 真正控制容器生命周期，启动一个容器就会启动一个 shim 进程
* shim 直接调用 runc 的包函数,shim 与 containerd 之前通过 rpc 通信
* 真正用户想启动的进程由 runc 的 init 进程启动，即 runc init [args ...]

```
docker     ctr
  |         |
  V         V
dockerd -> containerd ---> shim -> runc -> runc init -> process
                      |-- > shim -> runc -> runc init -> process
                      +-- > shim -> runc -> runc init -> process
```

containerd 只是一个守护进程，容器的实际运行时由 runC 控制。***containerd 主要职责是镜像管理（镜像、元信息等）、容器执行***（调用最终运行时组件执行）

![image](assets/image-20221227024423-1aw2ge0.png)​

# 2. 源码编译

需要安装依赖包：btrfs-tools，直接 make 即可生成 ctr containerd containerd-shim binaries 可执行文件

## 2.1 containerd main 函数

入口目录为 cmd/containerd/main.go 中 main 函数，默认配置文件 /etc/containerd/config.toml，包含三个子命令，configCommand，publishCommand，ociHook

```
func main() {
	app := command.App()
	if err := app.Run(os.Args); err != nil {
		fmt.Fprintf(os.Stderr, "containerd: %s\n", err)
		os.Exit(1)
	}
}
```

### 2.1.1 command App 函数

```
//  App 函数返回 *cli.App 实例
func App() *cli.App {
	app := cli.NewApp()
	app.Name = "containerd"
	app.Version = version.Version
	app.Usage = usage
	app.Description = `
containerd is a high performance container runtime whose daemon can be started
by using this command. If none of the *config*, *publish*, or *help* commands
are specified, the default action of the **containerd** command is to start the
containerd daemon in the foreground.


A default configuration is used if no TOML configuration is specified or located
at the default file location. The *containerd config* command can be used to
generate the default configuration for containerd. The output of that command
can be used and modified as necessary as a custom configuration.`
// 如果未指定 TOML 配置或位于默认文件位置，则使用默认配置。容器配置命令可用于生成容器的默认配置。该命令的输出可以根据需要作为自定义配置使用和修改
	app.Flags = []cli.Flag{
		cli.StringFlag{
			Name:  "config,c",
			Usage: "path to the configuration file",
			Value: filepath.Join(defaults.DefaultConfigDir, "config.toml"),
		},
		cli.StringFlag{
			Name:  "log-level,l",
			Usage: "set the logging level [trace, debug, info, warn, error, fatal, panic]",
		},
		cli.StringFlag{
			Name:  "address,a",
			Usage: "address for containerd's GRPC server",
		},
		cli.StringFlag{
			Name:  "root",
			Usage: "containerd root directory",
		},
		cli.StringFlag{
			Name:  "state",
			Usage: "containerd state directory",
		},
	}
	app.Flags = append(app.Flags, serviceFlags()...)
	// configCommand 用于生成配置文件 containerd config default > /etc/containerd/config.toml，publishCommand，ociHook  
	// publishCommand 二进制方式推送数据到containerd
	// ociHook 为 OCI 运行时钩子提供基础，以允许注入参数
       	app.Commands = []cli.Command{
		configCommand,
		publishCommand,
		ociHook,
	}
	//未指定子命令时要执行的操作 
        // 需要“cli.ActionFunc”，但可以接受“func（cli.Context） {}” 
	// 注意：对已弃用的“Action”的支持将在将来的版本中删除
	app.Action = func(context *cli.Context) error {
		var (
			start       = time.Now()
			signals     = make(chan os.Signal, 2048)
			serverC     = make(chan *server.Server, 1)
			ctx, cancel = gocontext.WithCancel(gocontext.Background())
			config      = defaultConfig()
		)

		defer cancel()

 		// 仅当配置存在或用户明确告诉我们加载此路径时，才尝试加载配置。
		configPath := context.GlobalString("config")
		_, err := os.Stat(configPath)
		if !os.IsNotExist(err) || context.GlobalIsSet("config") {
			if err := srvconfig.LoadConfig(configPath, config); err != nil {
				return err
			}
		}

		// 将传入参数应用于配置
		if err := applyFlags(context, config); err != nil {
			return err
		}

		//  确定根目录被创建
		if err := server.CreateTopLevelDirectories(config); err != nil {
			return err
		}

		// Stop if we are registering or unregistering against Windows SCM.
		stop, err := registerUnregisterService(config.Root)
		if err != nil {
			logrus.Fatal(err)
		}
		if stop {
			return nil
		}

		done := handleSignals(ctx, signals, serverC, cancel)
		// start the signal handler as soon as we can to make sure that
		// we don't miss any signals during boot
		signal.Notify(signals, handledSignals...)

		// 清理挂载点
		if err := mount.SetTempMountLocation(filepath.Join(config.Root, "tmpmounts")); err != nil {
			return fmt.Errorf("creating temp mount location: %w", err)
		}
		// unmount all temp mounts on boot for the server
		warnings, err := mount.CleanupTempMounts(0)
		if err != nil {
			log.G(ctx).WithError(err).Error("unmounting temp mounts")
		}
		for _, w := range warnings {
			log.G(ctx).WithError(w).Warn("cleanup temp mount")
		}
		// 配置文件中grpc address 不能为空
		if config.GRPC.Address == "" {
			return fmt.Errorf("grpc address cannot be empty: %w", errdefs.ErrInvalidArgument)
		}
		if config.TTRPC.Address == "" {
			// If TTRPC was not explicitly configured, use defaults based on GRPC.
			config.TTRPC.Address = fmt.Sprintf("%s.ttrpc", config.GRPC.Address)
			config.TTRPC.UID = config.GRPC.UID
			config.TTRPC.GID = config.GRPC.GID
		}
		log.G(ctx).WithFields(logrus.Fields{
			"version":  version.Version,
			"revision": version.Revision,
		}).Info("starting containerd")

		type srvResp struct {
			s   *server.Server
			err error
		}

		// run server initialization in a goroutine so we don't end up blocking important things like SIGTERM handling
		// while the server is initializing.
		// As an example opening the bolt database will block forever if another containerd is already running and containerd
		// will have to be be `kill -9`'ed to recover.
		// 在 goroutine 中运行服务器初始化，这样我们就不会在服务器初始化时阻止重要的事情，例如 SIGTERM 处理。
                // 例如，如果另一个 containerd 已经在运行，则打开 bolt 数据库将永远阻塞，并且 containerd 必须被“kill -9”才能恢复。
		chsrv := make(chan srvResp)
		go func() {
			defer close(chsrv)

			server, err := server.New(ctx, config)
			if err != nil {
				select {
				case chsrv <- srvResp{err: err}:
				case <-ctx.Done():
				}
				return
			}

			// Launch as a Windows Service if necessary
			if err := launchService(server, done); err != nil {
				logrus.Fatal(err)
			}
			select {
			case <-ctx.Done():
				server.Stop()
			case chsrv <- srvResp{s: server}:
			}
		}()

		var server *server.Server
		select {
		case <-ctx.Done():
			return ctx.Err()
		case r := <-chsrv:
			if r.err != nil {
				return r.err
			}
			server = r.s
		}

		// We don't send the server down serverC directly in the goroutine above because we need it lower down.
		select {
		case <-ctx.Done():
			return ctx.Err()
		case serverC <- server:
		}

		if config.Debug.Address != "" {
			var l net.Listener
			if isLocalAddress(config.Debug.Address) {
				if l, err = sys.GetLocalListener(config.Debug.Address, config.Debug.UID, config.Debug.GID); err != nil {
					return fmt.Errorf("failed to get listener for debug endpoint: %w", err)
				}
			} else {
				if l, err = net.Listen("tcp", config.Debug.Address); err != nil {
					return fmt.Errorf("failed to get listener for debug endpoint: %w", err)
				}
			}
			serve(ctx, l, server.ServeDebug)
		}
		if config.Metrics.Address != "" {
			l, err := net.Listen("tcp", config.Metrics.Address)
			if err != nil {
				return fmt.Errorf("failed to get listener for metrics endpoint: %w", err)
			}
			serve(ctx, l, server.ServeMetrics)
		}
		// setup the ttrpc endpoint
		tl, err := sys.GetLocalListener(config.TTRPC.Address, config.TTRPC.UID, config.TTRPC.GID)
		if err != nil {
			return fmt.Errorf("failed to get listener for main ttrpc endpoint: %w", err)
		}
		serve(ctx, tl, server.ServeTTRPC)

		if config.GRPC.TCPAddress != "" {
			l, err := net.Listen("tcp", config.GRPC.TCPAddress)
			if err != nil {
				return fmt.Errorf("failed to get listener for TCP grpc endpoint: %w", err)
			}
			serve(ctx, l, server.ServeTCP)
		}
		// setup the main grpc endpoint
		l, err := sys.GetLocalListener(config.GRPC.Address, config.GRPC.UID, config.GRPC.GID)
		if err != nil {
			return fmt.Errorf("failed to get listener for main endpoint: %w", err)
		}
		serve(ctx, l, server.ServeGRPC)

		if err := notifyReady(ctx); err != nil {
			log.G(ctx).WithError(err).Warn("notify ready failed")
		}

		log.G(ctx).Infof("containerd successfully booted in %fs", time.Since(start).Seconds())
		<-done
		return nil
	}
	return app

```

## 2.2 server.New 函数创建以及初始化 containerd server

### 2.2.1 初始化 contaienrd server，加载 timeout 配置

```
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
	if err := apply(ctx, config); err != nil {
		return nil, err
	}
	for key, sec := range config.Timeouts {
		d, err := time.ParseDuration(sec)
		if err != nil {
			return nil, fmt.Errorf("unable to parse %s into a time duration", sec)
		}
		timeout.Set(key, d)
	}
```

```shell
[timeouts]
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"
```

### 2.2.2 LoadPlugins 加载插件

```shell

// LoadPlugins loads all plugins into containerd and generates an ordered graph
// of all plugins.
func LoadPlugins(ctx context.Context, config *srvconfig.Config) ([]*plugin.Registration, error) {
	// load all plugins into containerd
	path := config.PluginDir
	if path == "" {
		path = filepath.Join(config.Root, "plugins")
	}
	if err := plugin.Load(path); err != nil {
		return nil, err
	}

```

* **注册插件 io.containerd.content.v1，** store 结构体 **实现了 Store 接口**

```shell
路径 containerd/content/local/store.go

// Store combines the methods of content-oriented interfaces into a set that
// are commonly provided by complete implementations.
type Store interface {
    Manager
    Provider
    IngestManager
    Ingester
}

```

```shell
// load additional plugins that don't automatically register themselves
	plugin.Register(&plugin.Registration{
		Type: plugin.ContentPlugin,
		ID:   "content",
		InitFn: func(ic *plugin.InitContext) (interface{}, error) {
			ic.Meta.Exports["root"] = ic.Root
			return local.NewStore(ic.Root)
		},
	})

```
