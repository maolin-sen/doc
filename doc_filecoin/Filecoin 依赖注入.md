# Filecoin 依赖注入

Filecoin 由一个个模块构成， 并采用 fx 实现各个模块间的依赖注入。

一般情况情况下，我们通过一个 New 函数构造一个对象，并通过函数参数方式手动传递各种依赖对象， 在一个大型的项目对象之间依赖关系变得非常复杂，如果我们手动的创建对象， 这是非常繁琐并且容易出错的。 而 fx 依赖框架可以为我们完成上面的工作。

## fx 基础
具体细节可看： https://pkg.go.dev/go.uber.org/fx#example-package

在 fx 中 constructor 构造器任何能够有返回值的函数可以看作是一个 constructor, 因为其创建（返回）了一些对象。 constructor 类似于某些类型的 New*** 函数。

```note
注意 constructor 可以同时返回一个多个对象。如果其中 一个对象的类型是 error 类型， 且值不为 nil 这时我们就知道在构建对象期间发生了错误， 此时就会停止构建。
```

我们启动一个程序过程通常如下：

创建各种对象， 并构建我们的程序。开启一个后台的 goroutine 运行我们的程序。在程序退出进行一些善后工作， 也就是执行 clean() 函数。

```go
srv, err := NewServer() 
if err != nil {
   log.Fatalf("failed to construct server: %v", err)
}

go srv.Start()
defer srv.Stop()
```

fx 框架将上面的过程抽象为 Lifeclye 。 并让程序启动阶段和结束阶段变成了 Hook 对象的 OnStart 和 OnStop 函数了。

```go
func NewMux(lc fx.Lifecycle, logger *log.Logger) *http.ServeMux {
	logger.Print("Executing NewMux.")
	mux := http.NewServeMux()

	server := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}
	lc.Append(fx.Hook{
		OnStart: func(context.Context) error {
			logger.Print("Starting HTTP server.")		
			go server.ListenAndServe()
			return nil
		},
		OnStop: func(ctx context.Context) error {
			logger.Print("Stopping HTTP server.")
			return server.Shutdown(ctx)
		},
	})

	return mux
}
```

在 fx 中用来当作依赖提供给别的对象，叫做 Provde, 你要构建的最终对象， 被称为 Invoke. 在创建 Fx 应用程序的时候， 你必须指定你提供的对象是 Provide 还是 Invoke.

```go
app := fx.New(
	fx.Provide(
		NewLogger,
		NewHandler,
		NewMux,
	),
    fx.Popute(&impl.FullNode),
	fx.Invoke(Register),
)
```

Populate会填充结构体的字段各个字段。

## filecoin 的依赖注入

在 filecoin 中有 ChainNode, MinerNode等角色的程序， 每个程序都包含若干个子系统，每个子系统之间相互独立， 也相互联系。 每个子系统都是我们的需要构建的目标， 相当于 fx.Invoke 对象， 这些子系统之间有构建顺序。 在 filecoin 中采用 setting 结构来容纳所有的构建对象， 最后并按照顺序传递 fx.App 让它按照顺序依次构建我们的子系统， 最后组成我们的程序。

```go
type Settings struct {
    // 模块或者子系统的依赖对象。 
	modules map[interface{}]fx.Option

    //子系统或者模块， 他们的返回类型不会作为其他模块或者对象的依赖对象。 这些模块必须按照顺序进行构建。
	invokes []fx.Option

    // 程序类型
	nodeType repo.RepoType

	Online bool // Online option applied
	Config bool // Config option applied
	Lite   bool // Start node in "lite" mode
}
```

下面是采用 fx 构建程序代码：

```go
// New builds and starts new Filecoin node
func New(ctx context.Context, opts ...Option) (StopFunc, error) {
	settings := Settings{
		modules: map[interface{}]fx.Option{},
		invokes: make([]fx.Option, _nInvokes),
	}

	// apply module options in the right order
	if err := Options(Options(defaults()...), Options(opts...))(&settings); err != nil {
		return nil, xerrors.Errorf("applying node options failed: %w", err)
	}

	// gather constructors for fx.Options
	ctors := make([]fx.Option, 0, len(settings.modules))
	for _, opt := range settings.modules {
		ctors = append(ctors, opt)
	}

	// fill holes in invokes for use in fx.Options
	for i, opt := range settings.invokes {
		if opt == nil {
			settings.invokes[i] = fx.Options()
		}
	}

	app := fx.New(
		fx.Options(ctors...),
		fx.Options(settings.invokes...),

		fx.NopLogger,
	)

	// TODO: we probably should have a 'firewall' for Closing signal
	//  on this context, and implement closing logic through lifecycles
	//  correctly
	if err := app.Start(ctx); err != nil {
		// comment fx.NopLogger few lines above for easier debugging
		return nil, xerrors.Errorf("starting node: %w", err)
	}

	return app.Stop, nil
}
```