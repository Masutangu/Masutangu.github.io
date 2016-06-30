---
layout: post
date: 2016-04-15T16:07:58+08:00
title: Python 进程池的坑：Pickling Error!
category: 编程语言
---

前阵子在跑[Elric](https://github.com/Masutangu/Elric)下的爬虫任务时，发现了worker进程有偶现的异常挂起的现象，通过strace看到worker进程block在futex(…, FUTEX_WAIT,…)这里，查看了worker的标准输出，发现打印了这么一行东西：

File “../multiprocessing/queues.py”, line 266, in _feed send(obj) PickingError: can’t pickle <type ‘thread.lock’>: attribute lookup thread.lock failed

# 解决思路 #
接下来就是艰辛的定位问题之旅：

Step 1：我检查了Elric里面的pickle操作（序列化提交的任务时会使用到pickle），没有发现问题。

Step 2：因为rpc调用会输出到标准输出，上个版本我刚好在master 新增了一个queue用以缓存没来得及处理的job，因此我也特别检查了这部分代码，也没有发现问题。

Step 3：从输出到traceback来看，最上层的调用不是我的代码，因此我猜测是进程池fork进程出来后发生的exception，之所以打印到标准输出来是因为这个exception没有被catch。

Step 4：打算用pdb打断点进行调试，没想到其他比较好的办法。于是我简单粗暴地把queues.py文件拷贝了一份，在第266行的send(obj)设置了断点。把代码里import到的queues文件都替换成我拷贝出来的这份，pdb执行一下，当遇到exception的时候程序就会停住。打印此时的obj，输出<core.my_process._ResultItem object at xxxx>，说明是在进程池往queue里塞进程执行结果的时候，pickle失败了。观察了下_ResultItem的成员，有work_id, exception, result。其中exception和result都有可能包含了thread.lock导致pickle失败。再打印下obj.exception，输出(<request.packages.urllib3.connectionpool.HTTPConnectionPool object at xxxx>, ‘Connection to xxxx timed out.(connect timeoout=3)’)

看起来有可能是HTTPConnectionPool这个对象无法pickle，验证一下：

<img src="/assets/images/python-pickling-error/illustration-1.png" alt="示例1" title="示例1" width="800" />

<img src="/assets/images/python-pickling-error/illustration-2.png" alt="示例2" title="示例2" width="800" />

果然是因为HTTPConnectionPool对象包含了lock导致无法被pickle。

但是在我自己电脑上验证时，pickle却不会报错。比对了线上环境和自己电脑的requests库版本，线上环境是2.3.0, 自己电脑是2.6.0。应该是requests修改了HTTPConnectionPool的实现，去掉了内部的lock。

# 解决方案 #
在使用python进程池提交任务的时候，注意任务执行可能会抛出一些无法pickle的exception，导致进程池拉取任务执行结果的时候pickle失败。建议是在任务代码中catch所有可能的exception，然后reraise自定义的支持pickle的exception。

# 总结 #
这次定位问题的手段太过简单粗暴，但自己也没想出更好的办法。幸好最终是确定了问题所在。如果大家对于解决该问题有什么建议，欢迎留言或发邮件提出。

