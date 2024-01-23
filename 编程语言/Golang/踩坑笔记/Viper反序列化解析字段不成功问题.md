# Viper反序列化解析字段不成功问题

## 问题背景

通过viper解析文件内容映射config一直失败，相关代码如下

```go

type Config struct {
	DBConf    *DBConf    `toml:"db"`
	RedisConf *RedisConf `toml:"redis"`
	WebConfig *WebConfig `toml:"app"`
}

type DBConf struct {
	Read struct {
		Dsn string `toml:"dsn"` // 最高优先级
	} `toml:"read"`
	Write struct {
		Dsn string `toml:"dsn"` // 最高优先级
	} `toml:"write"`
	Base struct {
		Type            string        `toml:"type"`
		MaxOpenConn     int           `toml:"maxOpenConn"`
		MaxIdleConn     int           `toml:"maxIdleConn"`
		ConnMaxLifeTime time.Duration `toml:"connMaxLifeTime"`
	} `toml:"base"`
}

type CacheType string

type RedisConf struct {
	Address  string `toml:"addr"`
	DB       string `toml:"db"`
	Password string `toml:"password"`
}

type WebConfig struct {
	Port        string `toml:"port"`
	IsDebug     bool   `toml:"debug"`
	LogFilePath string `toml:"path"`
}
	appFullPath := "./conf/" + filePath
	// 解析 config
	viper.SetConfigName("web")
	viper.AddConfigPath(appFullPath)
	viper.SetConfigType("yaml")
	err := viper.ReadInConfig()
	if err != nil {
		log.Fatal("解析文件失败: ", err)
	}
	if err := viper.Unmarshal(&conf); err != nil {
		log.Fatal("解析文件失败: ", err)
	}
	// 监听配置更新
	viper.WatchConfig()
	viper.OnConfigChange(func(e fsnotify.Event) {
		if err := viper.Unmarshal(&conf); err != nil {
			log.Fatal("解析文件失败: ", err)
		}
	})
```



## 问题原因

![image.png](https://s2.loli.net/2024/01/22/v7LP5Dlaegyrzwu.png)

Viper使用的是 github.com/mitchellh/mapstructure来解析值
mapstructure 用于将通用的map[string]interface{}解码到对应的 Go 结构体中
默认情况下，mapstructure 使用结构体中字段的名称做这个映射,不区分大小写,比如 Name 字段可以映射到
name、NAME、NaMe 等等，如果名称不一致没有指定 tagName ，则默认为 mapstructure,这也是为什么带下划线或者名称不一致的字段不加 mapstructure标签无法解析的原因

## 解决方案

### mapstructure方法

通过mapstructure映射名称

![image.png](https://s2.loli.net/2024/01/22/jHh1QiVOlfMzF8S.png)