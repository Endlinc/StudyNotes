# Keepalived 参数解析

## Abstract

Keepalived 是一个提供负载均衡和高可用软件，负载均衡框架依赖以及广泛使用的Linux虚拟服务器Kernel（IPVS）提供4层负载均衡，同时Keepalived 实现了健康检查，具有根据负载均衡服务器池的健康度来动态管理功能。接下来我们将要讨论Keepalived 初始化时，是怎么解析参数，而且配置内容将会被如何保存。

Keepalived项目完全使用 ANSI/ISO C便携。运行时周期被分成三个独立的进程：

- 父进程负责新建子进程，监控子进程
- 两个子进程，一个负责VRRP矿建，另一个负责健康检查

当运行时可以看到下列的进程列表:

```
PID         111     Keepalived  <-- Parent process monitoring children
            112     \_ Keepalived   <-- VRRP child
            113     \_ Keepalived   <-- Healthchecking child
```

## [Control Plane][atomic-elements]

Keepalived配置是通过便携文件`keepalived.conf`完成的。解析文件适用了编译器设计，解析器适用关键字树等级用于映射每个配置关键字和它专门的处理器。设计了一个中央多层递归函数用于读取配置文件，和遍历整个关键字树。在解析过程中，配置文件的内容被翻译变成内存中的对应结构体。

![1wih](D:\Reports\1wih.png)

## Code

本小节着重分析重要代码片段。

Keepalived会先初始化`global_data`内容，通过调用函数`alloc_global_data`，为`global_data`初试化内存。然后再调用`read_config_file`函数，传入另一个函数`global_init_keywords`作为参数，开始解析主进程所必要的参数。解析完成后再传入`init_global_data`函数检查是否存在主进程需要参数未配置的情况，如果有则补充上。

```c
/* Entry point */
int
keepalived_main(int argc, char **argv)
{
    ...
    global_data = alloc_global_data();

	read_config_file();

	init_global_data(global_data, NULL, false);
    ...
}

static void
read_config_file(void)
{
	init_data(conf_file, global_init_keywords);
}

static const vector_t *
global_init_keywords(void)
{
	/* global definitions mapping */
	init_global_keywords(true);

#ifdef _WITH_VRRP_
	init_vrrp_keywords(false);
#endif
#ifdef _WITH_LVS_
	init_check_keywords(false);
#endif
#ifdef _WITH_BFD_
	init_bfd_keywords(false);
#endif

	return keywords;
}
```

在主线程配置成功后，会检查是否有子线程残余，比如检查残留的pid文件。检查子进程不存在后会开始启动子进程，然后子进程的`global_data`开始初试化和解析配置文件内容。

```c
/* Entry point */
int
keepalived_main(int argc, char **argv)
{
    ...
	/* Init daemon */
	if (!start_keepalived())
		log_message(LOG_INFO, "Warning - keepalived has no configuration to run");
	...
}
```

主进程根据宏确定是否启动某些进程，运行正常情况下会启动VRRP和healthcheck两个子进程。

```c
/* Daemon init sequence */
static int
start_keepalived(void)
{
	bool have_child = false;

#ifdef _WITH_BFD_
	/* must be opened before vrrp and bfd start */
	open_bfd_pipes();
#endif

#ifdef _WITH_LVS_
	/* start healthchecker child */
	if (running_checker()) {
		start_check_child();
		have_child = true;
	}
#endif
#ifdef _WITH_VRRP_
	/* start vrrp child */
	if (running_vrrp()) {
		start_vrrp_child();
		have_child = true;
	}
#endif
#ifdef _WITH_BFD_
	/* start bfd child */
	if (running_bfd()) {
		start_bfd_child();
		have_child = true;
	}
#endif

	return have_child;
}
```

主线程启动Healthcheck子线后，同样会初始化`global_data`并解析和Healthcheck相关的参数。

```c
/* Register CHECK thread */
int
start_check_child(void)
{
#ifndef _ONE_PROCESS_DEBUG_
	pid_t pid;
	const char *syslog_ident;

	/* Initialize child process */
#ifdef ENABLE_LOG_TO_FILE
	if (log_file_name)
		flush_log_file();
#endif

	pid = fork();
    ...
    
	/* Start Healthcheck daemon */
	start_check(NULL, NULL);
    ...
}

/* Daemon init sequence */
static void
start_check(list old_checkers_queue, data_t *prev_global_data)
{
	init_checkers_queue();

	/* Parse configuration file */
	if (reload)
		global_data = alloc_global_data();
	check_data = alloc_check_data();
	if (!check_data) {
		stop_check(KEEPALIVED_EXIT_FATAL);
		return;
	}

	init_data(conf_file, check_init_keywords);

	if (reload)
		init_global_data(global_data, prev_global_data, true);
    ...
}
```

同样的，VRRP进程启动也是需要进行相同的操作。

```c
/* Register VRRP thread */
int
start_vrrp_child(void)
{
#ifndef _ONE_PROCESS_DEBUG_
	pid_t pid;
	const char *syslog_ident;

	/* Initialize child process */
#ifdef ENABLE_LOG_TO_FILE
	if (log_file_name)
		flush_log_file();
#endif

	pid = fork();
	...
	
	/* Start VRRP daemon */
	start_vrrp(NULL);
	...
}
```



[atomic-elements]: <https://keepalived.readthedocs.io/en/latest/software_design.html#atomic-elements>

