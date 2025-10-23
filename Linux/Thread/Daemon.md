## 概述
1. 一种在后台运行的计算机程序，它不与任何控制终端相关联。
2. 设计目的是为了在系统启动时运行，在系统关闭时终止，期间持续提供某种服务或执行特定任务，而不需要任何用户的直接交互。

## 特征
1. 后台运行
2. 长生命周期：它们通常随系统启动而启动，随系统关闭而停止。
3. 无控制终端：最关键的一点。由于没有控制终端，它们不会接收到终端发出的信号（如 SIGINT - Ctrl+C，SIGTSTP - Ctrl+Z 或 SIGHUP - 终端关闭）。即使用户登出，守护进程也会继续运行。
3. 父进程通常为 PID 1：父进程通常是 init 进程（在传统 SysVinit 系统中）或 systemd 进程（在现代系统中）。
4. 独立的会话和进程组：守护进程会创建一个全新的会话，并成为该会话的唯一进程组的领头进程（或者通过技巧确保它不是会话领头进程），从而彻底脱离启动它的那个会话。

## 分类
### stand alone
1. 可单独自行启动的 daemon，启动后会一直占用内存和系统资源。
2. 优点是响应速度快，多用于能够随时接受远程请求的服务，如 www 的daemon（httpd）、FTP 的daemon（vsftpd）等
### super daemon
1. 由一个特殊的 daemon 来统一管理，当没有远程请求时，这些服务都是未启动的。
2. 有远程请求时，super daemon 才唤醒相应的服务。当请求结束后，被唤醒的服务会关闭并释放系统资源。

## 创建
### fork()
1. 之后父进程退出，子进程继续执行，daemon 成为了init进程的子进程。原因如下
2. 假设 daemon 是从命令行启动的，父进程的终止会被 shell 发现，shell 在发现之后会显示出另一个 shell 提示符并让子进程继续在后台运行。
3. 子进程被确保不会称为一个进程组组长进程，因为它从其父进程那里继承了进程组 ID 并且拥有了自己的唯一的进程ID，而这个进程 ID 与继承而来的进程组 ID 是不同的，这样才能够成功地执行下面一个步骤。
### setsid()
1. 子进程调用 setsid() 开启一个新回话并释放它与控制终端之间的所有关联关系。
2. 结果就是使子进程: (a)成为新会话的首进程，(b)成为一个新进程组的组长进程，(c)没有控制终端。
### 非控制终端
1. 如果 daemon 从来没有打开过终端设备，那么就无需担心其会重新请求一个控制终端了。
2. 如果 daemon 后面可能会打开一个终端设备，那么必须要采取措施来确保这个设备不会成为控制终端。这可以通过下面两种方式实现：\
（1）在所有可能应用到一个终端设备上的 open() 调用中指定 O_NOCTTY标记。\
（2）在 setsid() 调用之后执行第二个fork()，然后再次让父进程退出并让孙子进程继续执行。这样就确保了子进程不会称为会话组长，因此根据 System V 中获取终端的规则，进程永远不会重新请求一个控制终端。
### 清除 umask
清除进程的 umask 以确保当 daemon 创建文件和目录时拥有所需的权限。
### 修改当前工作目录
1. 修改进程的当前工作目录，通常会改为根目录（/）。
2. daemon 通常会一直运行直至系统关闭为止。如果 daemon 的当前工作目录为不包含/的文件系统，那么就无法卸载该文件系统。
3. 或者 daemon 可以将工作目录改为完成任务时所在的目录或在配置文件中定义一个目录，只要包含这个目录的文件系统永远不会被卸载即可。
### 关闭文件描述符
1. 关闭 daemon 从其父进程继承而来的所有打开着的文件描述符。
2. daemon 可能需要保持继承而来的文件描述的打开状态，因此这一步是可选的或者可变更的。原因如下
3. 由于 daemon 失去了控制终端并且是在后台运行的，因此让 daemon 保持文件描述符 0（标准输入）、1（标准输出）和 2（标准错误）的打开状态毫无意义，因为它们指向的就是控制终端。
4. 无法卸载长时间运行的 daemon 打开的文件所在的文件系统。因此，通常的做法是关闭所有无用的打开着的文件描述符，因为文件描述符是一种有限的资源。
### 设备指向
1. 在关闭了文件描述符0、1 和 2 之后，daemon 通常会打开 /dev/null 并使用 dup2()（或类似的函数）使所有这些描述符指向这个设备。原因如下
2. 确保了当 daemon 调用了在这些描述符上执行 I/O 的库函数时不会出乎意料地失败。
3. 防止了 daemon 后面使用描述符 1 或 2 打开一个文件的情况，因为库函数会将这些描述符当做标准输出和标准错误来写入数据（进而破坏了原有的数据）。

## 操作
### 查看
```
ps axj		#查看系统中的守护进程
ps axj |grep daemon	#使用管道过滤想要查看的守护进程
```

## 创建实例
```
#define _CRT_SECURE_NO_WARNINGS
#include <iostream>
#include <sstream>
#include <ctime>
#include <stdio.h>
#include <cstdlib>
#include <sys/stat.h>
#include <time.h>
#include <assert.h>
#include <unistd.h>

int main()
{
	// 1.在父进程中执行 fork 并 exit 退出；
	pid_t m_pid;
	m_pid = fork();
	if (m_pid < 0) {
		exit(1);
	}
	else if (m_pid > 0) {
		exit(0);//父进程退出
	}
	// 2.在子进程中调用 setsid 函数创建新的会话；
	setsid(); //若当前进程不是进程组长，创建一个新会话；若当前进程已经是进程组长，返回错误
	// 3.调用 getcwd() 函数获取当前目录路径，并在子进程中调用 chdir函数，让根目录 ”/” 成为子进程的工作目录；
	char m_path[1024];
	if (getcwd(m_path, sizeof(m_path)) == NULL)	{
		perror("get path error!");
		exit(1);
	}
	chdir("/");
	// 4.在子进程中调用umask函数，设置进程的 umask 为 0
	umask(0);//设置进程的权限掩码，参数 0 表示：可以设置任何权限（读、写、执行）
	// 5.在子进程中关闭任何不需要的文件描述符
	for (int i = 0; i < getdtablesize(); i++) {
		close(i);
	}
	while (1) {
		/*
		* do anything...
		*/
		FILE* fp = fopen("/tmp/mytest.log", "a");
		assert(fp != NULL);

		time_t rawtime;
		struct tm* ptminfo;

		time(&rawtime);
		ptminfo = localtime(&rawtime);
		printf("current: %02d-%02d-%02d %02d:%02d:%02d\n",
			ptminfo->tm_year + 1900, ptminfo->tm_mon + 1, ptminfo->tm_mday,
			ptminfo->tm_hour, ptminfo->tm_min, ptminfo->tm_sec);

		fprintf(fp, "守护进程测试：%02d-%02d-%02d %02d:%02d:%02d\n\n", ptminfo->tm_year + 1900, ptminfo->tm_mon + 1, ptminfo->tm_mday,
			ptminfo->tm_hour, ptminfo->tm_min, ptminfo->tm_sec);
		fclose(fp);
		sleep(5);
	}
}
```