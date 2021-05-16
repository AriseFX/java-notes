从一行代码开始：

```java
    printf("Name: %s",parm)
```

先根据参数格式化文本，接着将文本打印到屏幕。找到linux中printf函数的实现：

```java
int printf(const char*fmt,...){
    va_list args;
    int i;
    va_start(args,fmt);
    write(1,printbuf,i=vsprintf(printbuf,fmt,args));
    va_end(args);
    return i;
}
```
最终会使用write()这一系统调用，本文会围绕着write的底层实现与设计思想来展开。

---

- 打印到屏幕如何实现？

下面是cpu、内存、外设组成的图像：

```markdown
                +-------+    +--------+
                |  CPU  |    | Memory |
                +---+---+    +---+----+
                    |            |
CPU-Memory Bus  +---+------+-----+----+
                           |
                  +--------+-------+
                  | Bus Controller |
                  +--------+-------+
                           |
       PCI Bus  +----------+---------------------------------------+---------------+
                                                                   |
                                                            +------+-------+
                                                            | Video memory |
                                                            +--------------+
```

流程：CPU发出写命令``->``CPU睡眠 ``->`` 写外设对应的寄存器 ``->`` 外设完事后向CPU发中断信号``->``CPU执行对应的中断处理程序。

    对于我们来说，最为核心的还是第一步。考虑的点非常多：

        是否统一编址，mov还是out？
        写哪个寄存器？
        先写高位还是先写低位？
        不同外设的标准还不一样。


为了解决这一问题，需要一个统一的接口，所以就有了『文件视图』：

    1.任何外设的操作都是open、read、write、close这几个系统调用
    2.不同的设备对应不同的设备文件（/dev/xxx）

那么操作IO都从IO接口开始，在系统调用层产生不同的分支，在设备驱动层操作硬件：
```
         +------+   +------+  +-------+   +-------+
         | open |   | read |  | write |   | close |
         +--+---+   +---+--+  +---+---+   +---+---+
            |           |         |           |
            +-----------+---------+-----------+
                           解释
                            |
         +------------------v---------------------+
  Driver |    command           interrupt handler |
         +-------+-----------------------^--------+
                 |                       ^
                 |                       |
         +-------v-----------------------+--------+
         |      Device controller(ex:disk)        |
Hardware +----------------------------------------+
         |                 Device                 |
         +----------------------------------------+
```
那么``write(1,printbuf,i=vsprintf(printbuf,fmt,args));``是怎么将文本打印到屏幕上的呢？

直接看sys_write的关键逻辑：

```cgo
    int sys_write(unsigned int fd, char * buf, int count){
        struct file * file;
        struct m_inode * inode;
        //current为当前进程，fd是是filp的下标（file的索引）
        if (fd >= NR_OPEN || count < 0 || !(file = current->filp[fd])) {
            return -EINVAL;
        }
        if (!count) {
            return 0;
        }
        //根据文件类型执行相应的写操作
        inode = file->f_inode;
        if (inode->i_pipe) { //管道
            //file->f_mode & 2 即是否有写的权限
            return (file->f_mode & 2) ? write_pipe(inode, buf, count) : -EIO;
        }
        if (S_ISCHR(inode->i_mode)) {//字符设备
            return rw_char(WRITE, inode->i_zone[0], buf, count, &file->f_pos);
        }
        if (S_ISBLK(inode->i_mode)) {//块设备
            return block_write(inode->i_zone[0], &file->f_pos, buf, count);
        }
        if (S_ISREG(inode->i_mode)) {//文件
            return file_write(inode, file, buf, count);
        }
        ...
        return -EINVAL;
    }
```
从代码中可以看出来，sys_write会先从当前进程的filp数组中找到具体的file，然后根据文件类型执行相应的写操作系。以上代码会有一些点值得注意：
    
    filp是怎么来的？
    f_inode是啥？
    f_mode是啥？

既然是从当前进程中获取的，那一定是从fork系统调用的『copy_process』中来的：

```cgo
    struct task_struct {
        ...                             
        int tty;                        //进程使用tty终端的子设备号。-1表示没有使用	
        unsigned short umask;           //文件创建属性屏蔽位 
        struct m_inode * pwd;           //当前工作目录i节点结构指针 
        struct m_inode * root;          //根目录i节点结构指针 
        struct m_inode * executable;	//执行文件i节点结构指针 
        struct m_inode * library;       //被加载库文件i节点结构指针 
        unsigned long close_on_exec;    //执行时关闭文件句柄位图标志 
        struct file * filp[NR_OPEN];    //文件结构指针表，最多32项。表项号即是文件描述符的值 
        struct desc_struct ldt[3];      //局部描述符表(段表), 0-空，1-代码段cs，2-数据和堆栈段ds&ss 
        struct tss_struct tss;          //进程的任务状态段信息结构 
    };
    
    int copy_process(int nr, /*省略一堆寄存器*/){
        ...
        //申请空闲的物理页
        p = (struct task_struct *) get_free_page();
        ...
        //复制父进程的内存页表，没有分配物理内存，共享父进程内存（虚拟内存部分看本人的其他文章）
        if (copy_mem(nr,p)) {
            task[nr] = NULL;
            free_page((long) p);
            return -EAGAIN;
        }
        //修改打开的文件，当前工作目录，根目录，执行文件，被加载库文件的使用数
        for (i = 0; i < NR_OPEN; i++) {
            if ((f = p->filp[i])) {
                f->f_count ++;//引用计数
            }
        }
        ...
    }
```
从代码中可以看出来filp是『进程描述符』的一个字段，而且是从父进程中拷贝过来的。我们知道进程是从init进程开始一层层fork出来的，
所以需要看linux初始化部分的逻辑。
```cgo
    void init(void){
        int pid, i;
        setup((void *) &drive_info);
        //tty是终端设备，open会返回fd
        (void) open("/dev/tty1", O_RDWR, 0);	//stdin
        (void) dup(0);                          //stdout
        (void) dup(0);                          //stderr
        ...
        //fork出任务2
        if (!(pid = fork())) {
            ...
            execve("/bin/sh", argv_rc, envp_rc);
            ...
        }
        ...
    }
```
init中打开了``/dev/tty``字符设备文件(终端)，并使用``dup``系统调用复制两次，返回0、1、2三个fd分别作为标准输入、标准输出、标准错误。
后续使用fd进行write就能写到对应的设备，可以得出``open``系统调用里一定有fd与设备的隐射关系！

看``sys_open``的逻辑：

