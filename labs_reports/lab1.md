# Lab1  Xv6 and Unix utilities

## 实验目的

1. 运行xv6系统
2. 通过实现`sleep`，`pingpong`，和`find`等函数，体会系统调用

## 实验步骤

### Boot xv6

```bash
junyu@Junyu-R9000P:/mnt/d/L1KEAB0T_xv6_lab$ git clone git://g.csail.mit.edu/xv6-labs-2021
Cloning into 'xv6-labs-2021'...
remote: Enumerating objects: 7051, done.
remote: Counting objects: 100% (7051/7051), done.
remote: Compressing objects: 100% (3423/3423), done.
remote: Total 7051 (delta 3702), reused 6830 (delta 3600), pack-reused 0
Receiving objects: 100% (7051/7051), 17.20 MiB | 3.07 MiB/s, done.
Resolving deltas: 100% (3702/3702), done.
warning: remote HEAD refers to nonexistent ref, unable to checkout.

junyu@Junyu-R9000P:/mnt/d/L1KEAB0T_xv6_lab$ cd xv6-labs-2021
junyu@Junyu-R9000P:/mnt/d/L1KEAB0T_xv6_lab$ git remote set-url origin git@github.com:L1KEAB0T/L1KEAB0T_xv6_lab.git
junyu@Junyu-R9000P:/mnt/d/L1KEAB0T_xv6_lab$ git push
Enumerating objects: 6932, done.
Counting objects: 100% (6932/6932), done.
Delta compression using up to 16 threads
Compressing objects: 100% (3285/3285), done.
Writing objects: 100% (6932/6932), 17.16 MiB | 1.70 MiB/s, done.
Total 6932 (delta 3650), reused 6897 (delta 3638), pack-reused 0
remote: Resolving deltas: 100% (3650/3650), done.
To github.com:L1KEAB0T/L1KEAB0T_xv6_lab.git
 * [new branch]      util -> util
 make qemu # 运行
```

### sleep

**要求**：实现xv6的UNIX程序**`sleep`**：您的**`sleep`**应该暂停到用户指定的计时数。一个滴答(tick)是由xv6内核定义的时间概念，即来自定时器芯片的两个中断之间的时间。您的解决方案应该在文件\*user/sleep.c\*中

代码编辑器采用的是VS code

此实验难度不大，参考 xv6 的相关文档和代码示例，只需要调用一个sleep的系统调用：

```c
//sleep.h
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]){
    if (argc < 2){
        fprintf(2,"Error: Lacking an argument...\n");
        exit(1);
   }

    sleep(atoi(argv[1]));

    exit(0);
}
```

```makefile
//Makefile中增加
//为了方便，将之后的都加上
UPROGS=\
	$U/_sleep\
	$U/_pingpong\
	$U/_find\
	$U/_xargs\
	$U/_primes\
```

在运行xv6后输入sleep 10，系统会停顿一会儿之后才能继续输入

## pingpong

**要求**：编写一个使用UNIX系统调用的程序来在两个进程之间“ping-pong”一个字节，请使用两个管道，每个方向一个。父进程应该向子进程发送一个字节;子进程应该打印“`<pid>: received ping`”，其中`<pid>`是进程ID，并在管道中写入字节发送给父进程，然后退出;父级应该从读取从子进程而来的字节，打印“`<pid>: received pong`”，然后退出。

```c
//pingpong.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]){
    if (argc != 1){
        fprintf(2,"Error: No need for arguments...\n");
        exit(1);
    }

    int p[2];
    pipe(p);

    if (fork() == 0){// child
        close(p[0]); // close write
        char temp = 'x';
        if (write(p[1],&temp,1))
            fprintf(0,"%d: received ping\n",getpid());    

        close(p[1]);
   }
    else{
        wait((int *)0);
        close(p[1]); // close read

        char temp;
        if (read(p[0],&temp,1))
           fprintf(0,"%d: received pong\n",getpid());    

        close(p[0]);    
    }

    exit(0);
}
```

![image-20240715205731469](.\images\image-20240715205731469.png)

### primes

**要求**：使用管道编写筛选素数

![img](D:\大学学习\大二下\操作系统小学期\labs\images\p1.png)

```c
//primes.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void primes(int *input, int count)
{
  if (count == 0) {
    return;
  }

  int p[2], i = 0, prime = *input;
  pipe(p);
  char buff[4];
  printf("prime %d\n", prime);
  if (fork() == 0) {
	close(p[0]);
	for (; i < count; i++) {
	  write(p[1], (char *)(input + i), 4);
	}
	close(p[1]);
	exit(0);
  } else {
	close(p[1]);
	count = 0;
	while (read(p[0], buff, 4) != 0) {
	  int temp = *((int *)buff);
	  if (temp % prime) {
	    *input++ = temp;
		  count++;
	  }
	}
	primes(input - count, count);
	close(p[0]);
	wait(0);
  }
}

int main(int argc, char *argv[]) {
  int input[34], i = 0;
  for (; i < 34; i++) {
    input[i] = i + 2;
  }
  primes(input, 34);
  exit(0);
}
```

![image-20240715210416540](.\images\image-20240715210416540.png)

### find

**要求**:查找目录树中具有特定名称的所有文件

```c
//find.c
#include "kernel/types.h"
#include "kernel/fcntl.h"
#include "kernel/stat.h"
#include "kernel/fs.h"
#include "user/user.h"

/*
	将路径格式化为文件名
*/
char* fmt_name(char *path){
  static char buf[DIRSIZ+1];
  char *p;

  // Find first character after last slash.
  for(p=path+strlen(path); p >= path && *p != '/'; p--);
  p++;
  memmove(buf, p, strlen(p)+1);
  return buf;
}
/*
	系统文件名与要查找的文件名，若一致，打印系统文件完整路径
*/
void eq_print(char *fileName, char *findName){
	if(strcmp(fmt_name(fileName), findName) == 0){
		printf("%s\n", fileName);
	}
}
/*
	在某路径中查找某文件
*/
void find(char *path, char *findName){
	int fd;
	struct stat st;	
	if((fd = open(path, O_RDONLY)) < 0){
		fprintf(2, "find: cannot open %s\n", path);
		return;
	}
	if(fstat(fd, &st) < 0){
		fprintf(2, "find: cannot stat %s\n", path);
		close(fd);
		return;
	}
	char buf[512], *p;	
	struct dirent de;
	switch(st.type){	
		case T_FILE:
			eq_print(path, findName);			
			break;
		case T_DIR:
			if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
				printf("find: path too long\n");
				break;
			}
			strcpy(buf, path);
			p = buf+strlen(buf);
			*p++ = '/';
			while(read(fd, &de, sizeof(de)) == sizeof(de)){
				//printf("de.name:%s, de.inum:%d\n", de.name, de.inum);
				if(de.inum == 0 || de.inum == 1 || strcmp(de.name, ".")==0 || strcmp(de.name, "..")==0)
					continue;				
				memmove(p, de.name, strlen(de.name));
				p[strlen(de.name)] = 0;
				find(buf, findName);
			}
			break;
	}
	close(fd);	
}

int main(int argc, char *argv[]){
	if(argc < 3){
		printf("find: find <path> <fileName>\n");
		exit(0);
	}
	find(argv[1], argv[2]);
	exit(0);
}

```

![image-20240715212650641](.\images\image-20240715212650641.png)

### xargs

**要求**:将上个命令输出的每行作为参数，拼接到xargs后面的指令后面

```c
//xargs.c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char *argv[]){
    int i;
    int j = 0;
    int k;
    int l,m = 0;
    char block[32];
    char buf[32];
    char *p = buf;
    char *lineSplit[32];
    for(i = 1; i < argc; i++){
        lineSplit[j++] = argv[i];
    }
    while( (k = read(0, block, sizeof(block))) > 0){
        for(l = 0; l < k; l++){
            if(block[l] == '\n'){
                buf[m] = 0;
                m = 0;
                lineSplit[j++] = p;
                p = buf;
                lineSplit[j] = 0;
                j = argc - 1;
                if(fork() == 0){
                    exec(argv[1], lineSplit);
                }                
                wait(0);
            }else if(block[l] == ' ') {
                buf[m++] = 0;
                lineSplit[j++] = p;
                p = &buf[m];
            }else {
                buf[m++] = block[l];
            }
        }
    }
    exit(0);
}

```

![image-20240715212919807](.\images\image-20240715212919807.png)

## 实验中遇到的问题以及解决办法

#### 编写 `sleep.c` 程序时的问题

**问题**：程序需要暂停指定的时间，但是不清楚如何使用 `sleep` 系统调用。

**解决办法**：参考 xv6 的文档，找到 `sleep` 系统调用的使用方法，使用 `atoi(argv[1])` 将命令行参数转换为整数，作为 `sleep` 的参数传递。



#### 编写 `primes.c` 程序时的问题

**问题**：如何使用管道递归地筛选素数。

**解决办法**：使用递归函数 `primes` 处理管道通信，父进程筛选当前的素数并将剩余数据写入管道，子进程读取数据并继续筛选。



xv6中的命名都在平常不是很常见，加大理解难度。多查阅资料



## 实验心得

通过这次实验，我深入理解了 xv6 操作系统的基本机制和 UNIX 系统调用的使用方法。在编写 `sleep.c` 时，我学会了如何正确使用系统调用来实现程序暂停，体会到系统调用是用户程序与操作系统内核交互的重要接口。编写 `pingpong.c` 时，我掌握了父子进程间通过管道进行通信的方法，确保了数据在正确的方向上传递。处理 `primes.c` 的过程中，我理解了如何递归使用管道进行数据筛选，通过不断的递归筛选出素数。编写 `find.c` 时，我学会了如何递归遍历目录树并查找特定文件，熟悉了文件系统操作的基本方法。在 `xargs.c` 中，我体验了将上一个命令的输出作为参数传递给另一个命令的过程，理解了标准输入输出的处理方式。整个实验让我对操作系统的底层机制有了更深刻的认识，增强了编写系统级程序的能力。