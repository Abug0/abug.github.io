# GDB-Python

## gdb-python 编码问题

进程中存在汉字的时候，使用py-bt、py-locals等命令会提示编码错误，如图：

![image-20221102163038846](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/202211030952351.png)

此时可以打开相应的python-gdb.py（名字不一定完全一样）,可以看到PyLocals:

![image-20221102164054895](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/202211030952217.png)

可以看到lines 1917-1920，在遍历栈帧中的局部变量并打印，可以修改此处，如图：

![image-20221102164238198](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/202211030952323.png)

重新source一下python-gdb.py，变量已经可以正常打印:

![image-20221102164349013](https://raw.githubusercontent.com/Abug0/Typora-Pics/master/pics/202211030948083.png)





> ### Unable to locate gdb frame for python bytecode interpreter
>
> Unable to locate gdb frame for python bytecode interpreter. Also tried: sudo gdb /usr/bin/python3.5-dbg -p <PID> and same error.
>
> Function: Frame.pc Returns the frame’s resume address. Function: Frame.block Return the frame’s code block. See Blocks In Python.If the frame does not have a block – for example, if there is no debugging information for the code in question – then this will throw an exception.
>
> gdb is not always able to read the relevant frame information, depending on the optimization level with which CPython was compiled. Internally, the commands look for C frames that are executing PyEval_EvalFrameEx (which implements the core bytecode interpreter loop within CPython) and look up the value of the related PyFrameObject *.

## 参考

[gdb --with-python](https://zditect.com/blog/273043.html)

[用 GDB 调试卡住的 Python 进程](https://cosven.me/blogs/85)

[查看cpython的编译参数](https://stackoverflow.com/questions/10192758/how-to-get-the-list-of-options-that-python-was-compiled-with)

[Use GDB for Debugging Running Python Processes](https://geronimo-bergk.medium.com/use-gdb-to-debug-running-python-processes-a961dc74ae36)

[DebuggingWithGdb](https://wiki.python.org/moin/DebuggingWithGdb)

[使用gdb调试CPython进程](https://github.com/ictar/python-doc/blob/master/Others/%E4%BD%BF%E7%94%A8gdb%E8%B0%83%E8%AF%95CPython%E8%BF%9B%E7%A8%8B.md)