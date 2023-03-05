golang 调度器在进行协程切换的时候，类似于函数调用
**关于函数调用：**
函数调用是通过执行 CALL 指令实现的。这条指令在 stack 上新建一个 frame，换句话说，把栈顶指针往上挪出去 frame size 的空间；这个栈顶指针通常存储在 CPU 的一个（虚拟）寄存器 SP（stack pointer）里。创建了新的 stack frame 之后，还需要把函数参数拷贝到新的 frame 里，然后让 CPU 执行被调用的函数 —— 也就是让 CPU 的 PC 寄存器的值变成被调用函数的起始地址；这样 CPU 接下来从 PC 包含的地址取指令来执行，也就开始执行被调用函数了。
CALL 指令修改 SP 和 PC 之前把它俩的值保存起来了，供 RET 指令使用。RET 指令是被调用函数的最后一条指令，它让 CPU 恢复事先保存的老的 SP 和 PC 的值，这样接下来 CPU 会执行调用函数在调用点之后的那条指令 —— 也就是继续执行调用函数了。

**所以，关于goroutine的切换/调度：**
那么 goroutine 切换要做什么呢？其实和函数调用一样，也是要修改 CPU 的 SP 和 PC 寄存器的值 —— 只是 SP 的修改不是简单地在当前 stack 范围里挪动了，而是会指向另一个 goroutine 的 stack 里的某个 frame。类似的，通过 RET 指令恢复 SP 和 PC，则切换回了之前的那个 goroutine。
所以 goroutine 的切换和函数调用如此类似啊！理解了这个相似性，我们就可以逻辑上把 M 执行 schedule 函数这个事儿“想象”成一个 goroutine 在执行 —— 这个想象中的 G 叫做 M 上的 g0。
当 M 被创建的时候，就有 g0 了。这个 g0 用的是 system stack，也就是 M 的 stack，而不是某个 G 的 stack，来执行 schedule 函数。
当 schedule 函数从 LRQ 或者 GRQ 取到一个 G，并且发起 goroutine 切换的时候，我们说 M 不再执行 g0，而是在执行这个新取到的 G 了。因为 G 执行的是一个 Go 函数，而任何函数的最后一条指令是 RET，所以当 G 执行完了之后，SP 和 PC 的值恢复到继续执行 g0。而 g0 仍然在 schedule 函数的无限循环里 —— 它会继续去找下一个可以执行的 G，然后切换过去执行之。

