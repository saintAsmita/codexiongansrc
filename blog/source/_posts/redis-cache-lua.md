---
title: redis-cache-lua
date: 2019-09-06 19:23:47
tags: [redis,go]
---

在使用Redis的过程中，难免会用到lua，如果lua脚本很多，可以在项目启动是将lua缓存到redis里面。本文将介绍golang下，如果在项目启动时缓存lua脚本。
<!--more-->

## 介绍

对于redis本身来说，通过命令`SCRIPT LOAD "你的lua脚本"`，可以将脚本缓存到redis中，并且返回此脚本的SHA1校验和。
另外，执行命令 `EVALSHA sha1 numkeys key [key …] arg [arg …]`即可以调用你缓存的脚本。
基于此，我们可以做五件事情：
1、项目启动时，通过`SCRIPT LOAD`将整个`lua`脚本缓存到Redis
2、记录下返回的SHA1到redis的一个key中，定时去检查这个key，这样其他进程更新了此SHA1，可以发现并同步。
3、包装一个方法，执行EVALSHA
4、对于lua脚本，由于执行的是整个脚本，所以暴露一个入口，供调用
5、项目启动时初始化

## 代码

一、代码
```golang
package myredis

import (
	"fmt"
	"io/ioutil"
	"os"
	"path"
	"time"
)

var DefRedisLuaEngine *RedisLuaEngine

type RedisLuaEngine struct {
	redisClient *redis.Client
	LuaScriptKey   string
	LuaFile        string
	LuaSHA1        string
}

func InitRedisLuaEngine(client  *redis.Client, luaFile string, luaScriptKey string) *RedisLuaEngine {
	// 多找几个地方，定位到脚本文件
	if !fileExist(luaFile) {
		luaFile = "./conf/script/" + path.Base(luaFile)
	}

	if !fileExist(luaFile) {
		luaFile = "../conf/script/" + path.Base(luaFile)
	}

	luaData, err := ioutil.ReadFile(luaFile)
	if err != nil {
		// todo output log
		panic(fmt.Errorf("read lua error, luaFile: %s, %s", luaFile, err))
	}

	if len(luaData) == 0 {
		// todo output log
		panic(fmt.Errorf("script is empty, luaFile: %s", luaFile))
	}

	// 缓存到redis并且记录SHA1
	luaSHA1, err := client.ScriptLoad(string(luaData)).Result()
	if err != nil {
		// todo output log
		panic(fmt.Errorf("load lua error, luaFile: %s, %s", luaFile, err))
	}

	// 保存SHA1 到redis
	_, err = client.Set(luaScriptKey, luaSHA1, 0).Result()
	if err != nil {
		// todo output log
		panic(fmt.Errorf("set SHA1 to redis error, luaFile: %s, %s, %s", luaFile, luaScriptKey, err))
	}

	redisLuaEngine := &RedisLuaEngine{
		redisClient: client,
		LuaScriptKey:   luaScriptKey,
		LuaFile:        luaFile,
		LuaSHA1:        luaSHA1,
	}

	// 定时更新SHA1
	go func() {
		ticker := time.NewTicker(10 * time.Minute)

		// TODO close ticker
		tools.WhenClose(func(_ os.Signal) {
			ticker.Stop()
		})
		for {
			<-ticker.C
			refreshSHA1(redisLuaEngine)
		}
	}()

	// TODO output success
	return redisLuaEngine
}

func refreshSHA1(engine *RedisLuaEngine) {
	sha1InRedis, err := engine.redisClient.Get(engine.LuaScriptKey).Result()
	if err != nil {
		// TODO log out put error and return
	}

	if engine.LuaSHA1 != sha1InRedis {
		// TODO log out put old new SHA1
		engine.LuaSHA1 = sha1InRedis
	}
}

// 不传入key，第一个参数作为lua中方法名，由lua脚本去根据methodName进行选取正确的代码段
func (engine RedisLuaEngine) Execute(method string, args ...interface{}) *redis.Cmd {
	return engine.redisClient.EvalSha(engine.LuaSHA1, []string{}, method, args)
}

func fileExist(filePath string) bool {
	if _, err := os.Stat(filePath); err == nil {
		return true
	}
	return false
}

```

二、lua
```lua
local CALL = redis.call
local ERROR = redis.error_reply

local function yourFunc(参数1, 参数2, 参数3)
    
end

fns["yourFunc"] = yourFunc

-- 程序入口
if #ARGV == 0 then
    ERROR("invalid args!")
end

return fns[ARGV[#ARGV]](unpack(ARGV))
```

三、main
```golang
redis.DefRedisLuaEngine  := InitRedisLuaEngine(client, "/conf/yourLua.lua", "yourkeyToStoreSHA1")

redis.DefRedisLuaEngine.Execute("yourFunc", "参数1", "参数2")
```