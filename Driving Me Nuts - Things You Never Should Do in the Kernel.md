---
title: Driving Me Nuts - Things You Never Should Do in the Kernel
date: 
tags:
- 老文翻译
- linux内核
- 文件读取
comments: true
categories: 老文翻译
---
 
## 那些你永远不应该在内核中做的使人抓狂的事情  
*Linux Journal by Greg Kroah-Hartmanon April 6, 2005*
来自 <https://www.linuxjournal.com/article/8110> 
 
在面向新开发者的linux内核编程邮件列表中（参见在线资源），有很多常见的问题被提了出来。几乎每次都有这样的一些问题被提出，它们的答案总是：“*不要那样做！*”，让迷惑的提问者想知道他们来到了一种什么样奇怪的发展社区。这是偶然系列文章中的第一篇，这些文章将试图解释为什么做这些事情通常不是一个好主意。然后，为了避免不良后果，我们会打破所有规则并展示应该怎么做。
<!-- more -->
***读取文件***  


不要做类别中最常被问到的问题是，“*怎样我才能从我的内核模块中读取文件？*”大多数新的内核开发者来自于用户空间开发环境或其它操作系统，在这些环境中，读取文件是将配置信息引入程序的自然而必要的部分。然而，在linux内核中，禁止从外部文件读取配置信息。这是由于如果开发人员尝试执行此操作可能会导致各种各样的问题。

试图从愚蠢的开发错误中保护内核并不是禁止驱动读取文件最重要的原因。最大的问题是策略。linux开发者尝试尽可能快的从文字策略中逃离。他们从不希望强制内核将可避免的策略强制应用于用户空间。使模块从文件系统的特定位置读取文件将强制设置该文件位置的策略。如果一名linux发行者为系统设计了最简单的方式去处理所有配置文件，将它们放置在/var/black/hole/of/configs，则必须修改此内核模块以支持这种更改。这对linux内核社区来说是不可接受的。  

尝试在内核中读取文件导致的另一个大问题是要弄清文件的确切位置。linux提供了文件系统命名空间，它允许每个进程去包含它自己的文件系统视图。它只允许一些程序去看到整个文件系统的一部分。这是一个强大的功能，但是这使确定你的模块是否存在于正确的文件系统命名空间中是不可能的。  

如果这些大问题还不够的话，最后的问题，如何将配置读取进内核，也是一个策略决策。每次强迫内核模块读取文件，作者都将破坏这种策略。然而，一些开发版可能觉得将系统配置存储在本地数据库中更好，并且有利于程序在恰当的时间将数据集中到内核中。或者，他们可能希望以某种方式去连接到外部计算机来确定此时的正确配置。无论用户采用何种方式去存储配置信息（存储在一个特定文件中），他或她将策略决策强加于用户，这会是一个坏注意。  

***但是我该怎样去配置呐？***

在最终理解了linux内核开发者对策略决定的厌恶，并知道了他们并不在意这些理想主义者后，你真正的疑问并没有得到解答，那就是如何将配置信息读取进内核模块。如何在不引起愤怒的电子邮件骂战的情况下完成此任务？  

把数据传递到特定内核模块的常用方法去是使用一个字符设备和ioctl系统调用。这将允许作者去传输几乎任何种数据到内核，而用户空间程序可以在初始化过程中的适当时间发送数据。然而已经确定ioctl命令具有许多讨厌的负面影响，因此不赞成在内核中创建新的ioctl。尝试正确的处理32位用户空间程序到64位内核的ioctl调用并使用正确的方式转换所有的数据类型同样是一项艰难的任务。因为ioctl是不被允许的，/proc文件系统已被惯用于将配置信息读取进内核。通过将数据写入内核模块在文件系统中创建的文件中，内核模块将可以直接访问它。但是最近内核开发者对proc文件系统进行了限制，因为它被开发者几乎滥用在所有种类数据的传输上。慢慢地，此文件系统被清理为仅包含进程信息，例如文件系统状态名称。作为一个更结构化的系统，sysfs文件系为任意设备和驱动提供了一种方式去创建文件来接受任何可能的配置信息。这种使用/procs的接口比ioctls更好。查看本专栏以前的文章来了解如何在内核模块中创建并使用sysfs文件。  

***无论如何我都想这么做***  

现来你理解禁止从内核模块来读取文件背后的原因了吧，你当然可以跳过这篇文章的剩余部分。剩下的部分跟你没关系了，因为你正忙着转换内核模块以使用sysfs。  

还在？好吧，那你仍然想知道如何从内核模块来读取文件，并且没有可能说服你。你要保证永远不要尝试将要提交在主内核树的代码中执行这种操作，并且我也从来没描述过如何这样做，行吗？  

其实，只要解决了一个小问题，读取文件是非常简单的。许多内核系统的调用被导出供模块使用。这些系统调用以sys_开头。可以使用函数sys_read来查看系统调用。  

读取一个文件的通常方法是尝试像下面那样的代码：  

{% codeblock lang:c %}
fd = sys_open(filename, O_RDONLY, 0);
    if (fd >= 0) {
         /* read the file here */
        sys_close(fd);
    } 
{% endcodeblock %} 

然而，在内核模块中尝试次操作时，sys_open()调用经常返回错误-EFAULT。这导致作者将问题发布到邮件列表，从而引发了上述中“*不要从内核读取文件*”的回复。  

最主要的是，作者忘记考虑到内核期望传输给sys_open()函数调用的指针来自于用户空间。因此，它进行了指针检查来确保它在正确的地址空间中，以便尝试将其转换为可以供内核其余部分可以使用的内核指针。所以，在我们尝试将内核指针传递给该函数时，-EFAULT错误将产生。  

***修复地址空间*** 

为了处理这个地址空间不匹配问题，我们要使用函数get_fs()和set_fs()，这些函数将当前进程地址限制修改为调用方想要的任何内容。在sys_open()的例子中，我们想告诉内核来自于内核地址空间的指针是安全的，所以我们这样调用：  

{% codeblock lang:c %}
set_fs(KERNEL_DS);
{% endcodeblock %} 

set_fs()函数只存在两个有效选项：KERNEL_DS和USER_DS，大致分别代表内核数据段和用户数据段。调用函数get_fs()来在修改它们前确定当前的地址限制。然后，在内核模块完成了对内核API的滥用后，可以使用它(set_fs())来恢复到正确的地址限制。  
根据这些知识，以上代码片段的正确书写方式是：  

{% codeblock lang:c %}
old_fs = get_fs();  
set_fs(KERNEL_DS);  
fd = sys_open(filename, O_RDONLY, 0);  
if (fd >= 0) {
  /* read the file here */
  sys_close(fd);
}
set_fs(old_fs);
{% endcodeblock %} 

这是一个读取文件/etc/shanow并将它转储到内核系统日志中的完整模块示例，它证明了这样做的可能危险，示例如下：  

{% codeblock lang:c %}
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/syscalls.h>
#include <linux/fcntl.h>
#include <asm/uaccess.h>

static void read_file(char *filename)
{
  int fd;
  char buf[1];

  mm_segment_t old_fs = get_fs();
  set_fs(KERNEL_DS);

  fd = sys_open(filename, O_RDONLY, 0);
  if (fd >= 0) {
    printk(KERN_DEBUG);
    while (sys_read(fd, buf, 1) == 1)
      printk("%c", buf[0]);
    printk("\n");
    sys_close(fd);
  }
  set_fs(old_fs);
}

static int __init init(void)
{
  read_file("/etc/shadow");
  return 0;
}

static void __exit exit(void)
{ }

MODULE_LICENSE("GPL");
module_init(init);
module_exit(exit);
{% endcodeblock %} 

***那么怎样写入呐？***  
现在，有了这些关于如何滥用内核系统调用API的新知识，并一心一意的惹恼了内核程序员，那你可以继续加油，在内核中进行文件的写入。启动你最喜欢的编辑器，然后敲出类似以下内容的东西：  

{% codeblock lang:c %}
old_fs = get_fs();
set_fs(KERNEL_DS);

fd = sys_open(filename, O_WRONLY|O_CREAT, 0644);
if (fd >= 0) {
  sys_write(data, strlen(data);
  sys_close(fd);
}
set_fs(old_fs);
{% endcodeblock %} 

这些代码似乎正确的搭建了，没有任何编译警告，但是当你尝试去加载这个模块时，你会得到这样的奇怪错误：  

{% codeblock lang:c %}
insmod: error inserting 'evil.ko': -1 Unknown symbol in module
{% endcodeblock %} 

这意味着你的模块尝试使用的一个函数没有被导出，并在内核中不可用。通过查看内核日志，你可以确定这个函数是：  

{% codeblock lang:c %}
evil: Unknown symbol sys_write
{% endcodeblock %}

因此，即使函数sys_write存在于syscalls.h头文件中，它并没有被导出供内核模块使用。其实，在三个不同的平台中，这个函数被导出了，但是又有谁真正使用了parisc架构【1】？为了解决这个问题，我们需要去利用内核提供给内核模块的可用函数。通过阅读有关如何实现的sys_write函数的代码，导出符号缺失的问题是可以被避免的。下面的内核模块展示了如何做到不使用sys_write调用完成此操作：  
{% codeblock lang:c %}
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/syscalls.h>
#include <linux/file.h>
#include <linux/fs.h>
#include <linux/fcntl.h>
#include <asm/uaccess.h>

static void write_file(char *filename, char *data)
{
  struct file *file;
  loff_t pos = 0;
  int fd;

  mm_segment_t old_fs = get_fs();
  set_fs(KERNEL_DS);

  fd = sys_open(filename, O_WRONLY|O_CREAT, 0644);
  if (fd >= 0) {
    sys_write(fd, data, strlen(data));
    file = fget(fd);
    if (file) {
      vfs_write(file, data, strlen(data), &pos);
      fput(file);
    }
    sys_close(fd);
  }
  set_fs(old_fs);
}

static int __init init(void)
{
  write_file("/tmp/test", "Evil file.\n");
  return 0;
}

static void __exit exit(void)
{ }

MODULE_LICENSE("GPL");
module_init(init);
module_exit(exit);
{% endcodeblock %}
正如你所看到的，通过使用函数fget,fput和vfs_write，我们可以实现我们自己的sys_write功能。  

***我从来没有告诉过你这些***  

总的来说，在内核模块中进行文件的读取和写入是非常非常糟糕的。永远不要这样做。  Linux Journal FTP站点提供了本文中的两个模块以及用于编译它们的Makefile,但是我们希望在日志中看不到下载记录。而且，我从来没有告诉你这样做。你是从别的其他人那里知道的，他从他妹妹最好的朋友那里学到的，她朋友是从他同事那里听说的。  

本文资源下载：https://www.linuxjournal.com/article/8130  

本文作者介绍：  
Greg Kroah-Hartman是《 Linux设备驱动程序》（第三版）的作者之一，并且是内核驱动程序的维护者，他提供了许多他不愿承认的驱动程序子系统。他在SuSE Labs工作，从事各种特定于内核的工作，可以通过greg@kroah.com与他联系，以解决与本文无关的问题。  

译者注：  
【1】PA-RISC，一种处理器指令集架构（ISA），属于精简指令集架构。PA 是指精准指令集架构（英语：Precision Architecture），由惠普公司开发，所以它又被称为惠普精准指令集架构（Hewlett Packard Precision Architecture，缩写为HP/PA）。它首次出现于1986年2月26日，被应用于HP 3000 930系列以及HP 9000 840模式处理器之中。它之后被惠普公司与英特尔联合开发的Itanium架构所取代。