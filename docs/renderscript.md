RenderScript简介

我们首先尝试了RenderScript。在Google的介绍中， RenderScript是一种基于异构平台，高效计算的强大工具，尤其擅长图像，音频处理以及复杂的科学计算。RenderScript包括一个内核（kernel）编写语言和一组Java SDK。内核语言的语法类似C99标准，google为kernel提供了运行环境以及“标准库”。标准库主要提供了数学计算，向量/矩阵计算，以及一些OpenGL的功能（在4.2上已经被舍弃了），和log（调试用）。Java SDK让开发者在应用程序中管理RenderScript的整个lifecycle, 包括创建上下文，script对象，设置script全局变量，运行script等等。

通过合理地编写kernel，Android可以将可并行的计算分布到CPU的多个内核，或者GPU，甚至DSP之上。Android声称，任务非配的策略取决于当前各个硬件的负载以及kernel中的逻辑（适合CPU还是GPU）。Google的LLVM机制可以保证Renderscript在各个系统平台上运行，并保持类似native(assembler)的执行速度。从用途，以及代码的编写方式来看，RenderScript和OpenCL非常类似。

相关demo: 
<https://github.com/qylk/renderscript_examples>
<http://tangzm.com/blog/?p=18>