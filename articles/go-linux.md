<!---
markmeta_author: wongoo
markmeta_date: 2019-09-10
markmeta_title: Go Linux
markmeta_categories: 编程语言
markmeta_tags: golang,linux
-->

# go linux

## 接收父进程死掉的信号

```
// Child can ask kernel to deliver SIGHUP (or other signal) when parent dies
// by specifying option PR_SET_PDEATHSIG in prctl() syscall like this:
//
//prctl(PR_SET_PDEATHSIG, SIGHUP);
//
//See man 2 prctl for details.
//
//Edit: This is Linux-only
func setParentDeathSignal(sig uintptr) error {
	if err := unix.Prctl(unix.PR_SET_PDEATHSIG, sig, 0, 0, 0); err != nil {
		return err
	}
	return nil
}


err = setParentDeathSignal(uintptr(syscall.SIGINT))
if err != nil {
	log.Fatalf("setParentDeathSignal error: %v", err)
}
```

