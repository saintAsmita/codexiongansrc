---
title: golang实现redis lock
date: 2020-01-09 21:30:39
tags:  golang redis
---

一般redis lock需要满足：
1、不同有多个进程抢到锁，这个通过 SETNX 实现；
2、锁会自动过期，防止某些原因导致锁永远不能释放，通过SETNX 带时间戳实现；
3、释放自己的锁，通过Lua脚本判断准备释放的锁中的值是否为自己设置的。
<!--more-->

## golang实现redis lock

一般redis lock需要满足：
1、不同有多个进程抢到锁，这个通过 SETNX 实现；
2、锁会自动过期，防止某些原因导致锁永远不能释放，通过SETNX 带时间戳实现；
3、释放自己的锁，通过Lua脚本判断准备释放的锁中的值是否为自己设置的。

使用时，调用RetryLock或者GetLock即可。
释放锁，使用LockRes.ReleaseLock()。
代码
```goalng
package lock

import (
	"crypto/rand"
	"encoding/base64"
	"errors"
	"github.com/go-redis/redis"
	"io"
	"time"
)

var luaRelease = redis.NewScript(`if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`)

type LockRes struct {
	key string
	val string
}

var (
	// ErrNotGetLock is returned when a lock cannot be obtained.
	ErrNotGetLock = errors.New("redis lock: not get lock")
	// ErrLockNotHeld is returned when trying to release an inactive lock.
	ErrLockNotHeld = errors.New("redis lock: not held lock")
)

// 获取锁，可指定重试次数，每次重试间隔时间
func RetryLock(c *redis.Client, key string, ttl time.Duration, retryCount int, backOff time.Duration) (*LockRes, error) {
	val, err := randomVal()
	if err != nil {
		return nil, err
	}

	if retryCount == 0 {
		return tryLock(c, key, ttl, val)
	}

	for ; retryCount > 0; retryCount-- {
		res, err := tryLock(c, key, ttl, val)
		if err != nil {
			if err == ErrNotGetLock {
				if backOff > 0 {
					time.Sleep(backOff)
				}
				continue
			}

			return nil, err
		}

		// get lock
		return res, nil
	}

	return nil, ErrNotGetLock
}

// 获取锁，获取不到立即返回
func GetLock(c *redis.Client, key string, ttl time.Duration) (*LockRes, error) {

	val, err := randomVal()
	if err != nil {
		return nil, err
	}

	return tryLock(c, key, ttl, val)
}

func (l *LockRes) ReleaseLock(c *redis.Client) error {

	res, err := luaRelease.Run(c, []string{l.key}, l.val).Int64()
	if err != nil {
		return err
	}

	if res != 1 {
		return ErrLockNotHeld
	}
	return nil
}

func tryLock(c *redis.Client, key string, ttl time.Duration, val string) (*LockRes, error) {
	ok, err := lock(c, key, val, ttl)
	if err != nil {
		return nil, err
	}

	if ok {
		return &LockRes{
			key: key,
			val: val,
		}, nil
	} else {
		return nil, ErrNotGetLock
	}
}

func lock(c *redis.Client, key, val string, ttl time.Duration) (bool, error) {
	ok, err := c.SetNX(key, val, ttl).Result()
	if err != nil {
		return false, err
	}

	return ok, nil
}

func randomVal() (string, error) {

	tmp := make([]byte, 16)

	if _, err := io.ReadFull(rand.Reader, tmp); err != nil {
		return "", err
	}
	return base64.RawURLEncoding.EncodeToString(tmp), nil
}
```