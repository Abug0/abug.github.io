# GDB-Python

> ### Unable to locate gdb frame for python bytecode interpreter
>
> Unable to locate gdb frame for python bytecode interpreter. Also tried: sudo gdb /usr/bin/python3.5-dbg -p <PID> and same error.
>
> Function: Frame.pc Returns the frame’s resume address. Function: Frame.block Return the frame’s code block. See Blocks In Python.If the frame does not have a block – for example, if there is no debugging information for the code in question – then this will throw an exception.
>
> gdb is not always able to read the relevant frame information, depending on the optimization level with which CPython was compiled. Internally, the commands look for C frames that are executing PyEval_EvalFrameEx (which implements the core bytecode interpreter loop within CPython) and look up the value of the related PyFrameObject *.

## 参考

[gdb --with-python](https://zditect.com/blog/273043.html)