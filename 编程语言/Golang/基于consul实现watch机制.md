#                       基于consul实现watch机制

## 前言

consul常常被用来作服务注册与服务发现，而它的watch机制则可被用来监控一些数据的更新，实时获取最新的数据。另外，在监控到数据变化后，还可以调用外部处理程序，此处理程序可以是任何可执行文件或HTTP调用，具体说明可见[官网](https://link.juejin.cn?target=https%3A%2F%2Fwww.consul.io%2Fdocs%2Fdynamic-app-config%2Fwatches)。

 当前consul支持以下watch类型如下所示：

- key 监听一个consul kv中的key
- keyprefix 监听consul kv中的key的前缀
- services 监听有效服务的变化
- nodes 监听节点的变化
- service 监听服务的变化
- checks 监听check的变化
- event 监听自定义事件的变化

从以上可以看出consul提供非常丰富的监听类型，通过这些类型我们可以实时观测到consul整个集群中的变化，从而实现一些特别的需求，比如：实时更新、服务告警等功能。



## 基于Golang 实现watch 对服务变化的监控

consul官方提供了[Golang版的watch包](https://link.juejin.cn?target=https%3A%2F%2Fpkg.go.dev%2Fgithub.com%2Fhashicorp%2Fconsul%2Fapi%2Fwatch)。其实际上也是对watch机制进行了一层封装，最终代码实现的只是对consul HTTP API 的 `endpoints`的使用，不涉及数据变化后的相关处理，封装程度不够。

接下来我将基于封装了的相关处理函数的工具包进行解决，详细代码可通过https://github.com/longpi1/consul-tool 进行下载查看。

监听实现可参考：https://github.com/mitchellh/consulstructure

1.客户端client.go 用于初始consul相关配置以及封装consul的api库的基础操作

```golang
package backends

import (
	"encoding/json"
	"fmt"
	"github.com/hashicorp/consul/api"
	errors "github.com/longpi1/consul-tool/pkg/error"
	"github.com/longpi1/consul-tool/pkg/log"
	"strings"
	"sync"
	"time"
)

// Option ...
type Option func(opt *Config)

// NewConfig 初始化consul配置
func NewConfig(opts ...Option) *Config {
	c := &Config{
		conf:     api.DefaultConfig(),
		watchers: make(map[string]*watcher),
		logger:   log.NewLogger(),
	}

	for _, o := range opts {
		o(c)
	}

	return c
}

// Config 相关配置的结构体
type Config struct {
	sync.RWMutex
	logger   log.Logger
	kv       *api.KV
	conf     *api.Config
	watchers map[string]*watcher
	prefix   string
}

// 循环监听
func (c *Config) watcherLoop(path string) {
	c.logger.Info("watcher start...", "path", path)

	w := c.getWatcher(path)
	if w == nil {
		c.logger.Error("watcher not found", "path", path)
		return
	}

	for {
		if err := w.run(c.conf.Address, c.conf); err != nil {
			c.logger.Warn("watcher connect error", "path", path, "error", err)
			time.Sleep(time.Second * 3)
		}

		w = c.getWatcher(path)
		if w == nil {
			c.logger.Info("watcher stop", "path", path)
			return
		}

		c.logger.Warn("watcher reconnect...", "path", path)
	}
}

// 重置consul的watcher
func (c *Config) Reset() error {
	watchMap := c.getAllWatchers()

	for _, w := range watchMap {
		w.stop()
	}

	return c.Init()
}

// Init 初始化consul客户端
func (c *Config) Init() error {
	client, err := api.NewClient(c.conf)
	if err != nil {
		return fmt.Errorf("init fail: %w", err)
	}

	c.kv = client.KV()
	return nil
}

// Put 插入该路径的kv
func (c *Config) Put(path string, value interface{}) error {
	var (
		data []byte
		err  error
	)

	data, err = json.Marshal(value)
	if err != nil {
		data = []byte(fmt.Sprintf("%v", value))
	}

	p := &api.KVPair{Key: c.absPath(path), Value: data}
	_, err = c.kv.Put(p, nil)
	if err != nil {
		return fmt.Errorf("put fail: %w", err)
	}
	return nil
}

// Get 获取该路径的kv
func (c *Config) Get(keys ...string) (ret *KV) {
	var (
		path   = c.absPath(keys...) + "/"
		fields []string
	)

	ret = &KV{}
	ks, err := c.list()
	if err != nil {
		ret.err = fmt.Errorf("get list fail: %w", err)
		return
	}

	for _, k := range ks {
		if !strings.HasPrefix(path, k+"/") {
			ret.err = errors.ErrKeyNotFound
			continue
		}
		field := strings.TrimSuffix(strings.TrimPrefix(path, k+"/"), "/")
		if len(field) != 0 {
			fields = strings.Split(field, "/")
		}

		kvPair, _, err := c.kv.Get(k, nil)
		ret.value = kvPair.Value
		ret.key = strings.TrimSuffix(strings.TrimPrefix(path, c.prefix+"/"), "/")
		if err != nil {
			err = fmt.Errorf("get fail: %w", err)
		}
		ret.err = err
		break
	}

	if len(fields) == 0 {
		return
	}
	ret.key += "/" + strings.Join(fields, "/")
	return
}

// Delete 删除该路径的kv
func (c *Config) Delete(path string) error {
	_, err := c.kv.Delete(c.absPath(path), nil)
	if err != nil {
		return fmt.Errorf("delete fail: %w", err)
	}
	return nil
}

// Watch   实现监听
func (c *Config) Watch(path string, handler func(*KV)) error {
	// 初始化watcher
	watcher, err := newWatcher(c.absPath(path))
	if err != nil {
		return fmt.Errorf("watch fail: %w", err)
	}
	// 对应的路径发生变化时，调用对应的处理函数
	watcher.setHybridHandler(c.prefix, handler)
	// 相应路径下添加对应的wathcer用于实现watch机制
	err = c.addWatcher(path, watcher)
	if err != nil {
		return err
	}
	// 调用协程循环监听
	go c.watcherLoop(path)
	return nil
}

// StopWatch 停止监听
func (c *Config) StopWatch(path ...string) {
	if len(path) == 0 {
		c.cleanWatcher()
		return
	}

	for _, p := range path {
		wp := c.getWatcher(p)
		if wp == nil {
			c.logger.Info("watcher already stop", "path", p)
			continue
		}

		c.removeWatcher(p)
		wp.stop()
		for !wp.IsStopped() {
		}
	}
}

// 获取绝对路径
func (c *Config) absPath(keys ...string) string {
	if len(keys) == 0 {
		return c.prefix
	}

	if len(keys[0]) == 0 {
		return c.prefix
	}

	if len(c.prefix) == 0 {
		return strings.Join(keys, "/")
	}

	return c.prefix + "/" + strings.Join(keys, "/")
}

func (c *Config) list() ([]string, error) {
	keyPairs, _, err := c.kv.List(c.prefix, nil)
	if err != nil {
		return nil, err
	}

	list := make([]string, 0, len(keyPairs))
	for _, v := range keyPairs {
		if len(v.Value) != 0 {
			list = append(list, v.Key)
		}
	}

	return list, nil
}

// WithPrefix ...
func WithPrefix(prefix string) Option {
	return func(c *Config) {
		c.prefix = prefix
	}
}

// WithAddress ...
func WithAddress(address string) Option {
	return func(c *Config) {
		c.conf.Address = address
	}
}

// Withlogger ...
func Withlogger(logger log.Logger) Option {
	return func(c *Config) {
		c.logger = logger
	}
}


// CheckWatcher ...
func (c *Config) CheckWatcher(path string) error {
	c.RLock()
	defer c.RUnlock()

	if _, ok := c.watchers[c.absPath(path)]; ok {
		return errors.ErrAlreadyWatch
	}

	return nil
}

func (c *Config) getWatcher(path string) *watcher {
	c.RLock()
	defer c.RUnlock()

	return c.watchers[c.absPath(path)]
}

func (c *Config) addWatcher(path string, w *watcher) error {
	c.Lock()
	defer c.Unlock()

	if _, ok := c.watchers[c.absPath(path)]; ok {
		return errors.ErrAlreadyWatch
	}

	c.watchers[c.absPath(path)] = w
	return nil
}

func (c *Config) removeWatcher(path string) {
	c.Lock()
	defer c.Unlock()

	delete(c.watchers, c.absPath(path))
}

func (c *Config) cleanWatcher() {
	c.Lock()
	defer c.Unlock()

	for k, w := range c.watchers {
		w.stop()
		delete(c.watchers, k)
	}
}

// 获取所有的watcher
func (c *Config) getAllWatchers() []*watcher {
	c.RLock()
	defer c.RUnlock()

	watchers := make([]*watcher, 0, len(c.watchers))
	for _, w := range c.watchers {
		watchers = append(watchers, w)
	}

	return watchers
}


```

2.watcher.go实现对watch机制相关函数的封装。

```
package backends

import (
   "bytes"
   "fmt"
   "strings"
   "sync"

   "github.com/hashicorp/consul/api"
   "github.com/hashicorp/consul/api/watch"
)

//初始化对应的watcher ，这里设置的是监听路径的类型，也可以支持service、node等，通过更改type
//支持的type类型有
//key - Watch a specific KV pair
//keyprefix - Watch a prefix in the KV store
//services - Watch the list of available services
//nodes - Watch the list of nodes
//service- Watch the instances of a service
//checks - Watch the value of health checks
//event - Watch for custom user events
func newWatcher(path string) (*watcher, error) {
   wp, err := watch.Parse(map[string]interface{}{"type": "keyprefix", "prefix": path})
   if err != nil {
      return nil, err
   }

   return &watcher{
      Plan:       wp,
      lastValues: make(map[string][]byte),
      err:        make(chan error, 1),
   }, nil
}

func newServiceWatcher(serviceName string) (*watcher, error) {
   wp, err := watch.Parse(map[string]interface{}{"type": "service", "service": serviceName})
   if err != nil {
      return nil, err
   }
   return &watcher{
      Plan:       wp,
      lastValues: make(map[string][]byte),
      err:        make(chan error, 1),
   }, nil
}

type watcher struct {
   sync.RWMutex
   *watch.Plan
   lastValues    map[string][]byte
   hybridHandler watch.HybridHandlerFunc  // 当对于路径发生变化时，调用相应函数
   stopChan      chan struct{}
   err           chan error
}

//获取value
func (w *watcher) getValue(path string) []byte {
   w.RLock()
   defer w.RUnlock()

   return w.lastValues[path]
}

//更新value
func (w *watcher) updateValue(path string, value []byte) {
   w.Lock()
   defer w.Unlock()

   if len(value) == 0 {
      delete(w.lastValues, path)
   } else {
      w.lastValues[path] = value
   }
}

//用于设置对应的处理函数
func (w *watcher) setHybridHandler(prefix string, handler func(*KV)) {
   w.hybridHandler = func(bp watch.BlockingParamVal, data interface{}) {
      kvPairs := data.(api.KVPairs)
      ret := &KV{}

      for _, k := range kvPairs {
         path := strings.TrimSuffix(strings.TrimPrefix(k.Key, prefix+"/"), "/")
         v := w.getValue(path)

         if len(k.Value) == 0 && len(v) == 0 {
            continue
         }

         if bytes.Equal(k.Value, v) {
            continue
         }

         ret.value = k.Value
         ret.key = path
         w.updateValue(path, k.Value)
         handler(ret)
      }
   }
}

//运行watcher机制
func (w *watcher) run(address string, conf *api.Config) error {
   w.stopChan = make(chan struct{})
   w.Plan.HybridHandler = w.hybridHandler

   go func() {
      w.err <- w.RunWithConfig(address, conf)
   }()

   select {
   case err := <-w.err:
      return fmt.Errorf("run fail: %w", err)
   case <-w.stopChan:
      w.Stop()
      return nil
   }
}

func (w *watcher) stop() {
   close(w.stopChan)
}
```

3.main.go，初始化consul配置信息后，实现对test路径下的Key进行监听；

```
package main

import (
	"github.com/longpi1/consul-tool/internal/backends"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	// 初始化consul配置信息
	cli := backends.NewConfig(backends.WithPrefix("kvTest"))
	if err := cli.Init(); err != nil {
		log.Fatalln(err)
	}
	//监听consul中的key： test
	err := cli.Watch("test", func(r *backends.KV) {
		log.Printf("该key： %s 已经更新", r.Key())
	})
	if err != nil {
		log.Fatalln(err)
	}
	//插入key
	if err := cli.Put("test", "value"); err != nil {
		log.Fatalln(err)
	}
	//读取key
	if ret := cli.Get("test"); ret.Err() != nil {
		log.Fatalln(ret.Err())
	} else {
		println(ret.Value())
	}

	c := make(chan os.Signal, 1)
	// 监听退出相关的syscall
	signal.Notify(c, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)
	for {
		s := <-c
		log.Printf("exit with signal %s", s.String())
		switch s {
		case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
			//停止监听对应的路径
			cli.StopWatch("test")
			time.Sleep(time.Second * 2)
			close(c)
			return
		case syscall.SIGHUP:
		default:
			close(c)
			return
		}
	}
}

```

## 参考链接

1. JackBai233，[使用Consul的watch机制监控注册的服务变化](https://juejin.cn/post/6984378158347157512)
2. 风车，[深入Consul Watch功能](https://zhuanlan.zhihu.com/p/111673886)

