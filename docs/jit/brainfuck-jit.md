# Brainfuck Just-In-Time Compiler

## 简介

Brainf*ck 是一个非常有趣的 esolang，它只用 8 种操作符实现了图灵等价性，更多关于该语言的趣味信息在这里不多赘述，可以自行搜索。如下表格展示了每个操作符实际对应的 C/C++ 等价代码：

|Operator|Code in C/C++|
|:----|:----|
|`+`|`buff[ptr]++`|
|`-`|`buff[ptr]--`|
|`>`|`ptr++`|
|`<`|`ptr--`|
|`[`|`if (!buff[ptr]) goto ']'`|
|`]`|`if (buff[ptr] goto '['`|
|`,`|`buff[ptr] = getchar()`|
|`.`|`putchar(buff[ptr])`|

由于 bf 简单的语法，编写该语言的词法和语法分析易如反掌，因此该语言也被广泛用于示范解释器和编译器的实现。同样，由于其实际等价代码片段也非常简单，所以也非常适用于展示如何开发一个最简单的 JIT 编译器。

本文提供了 bf 在 x86_64 windows/linux/macOS 三个平台皆可使用的 JIT 编译器的实例，当然，由于目前 macOS 已经基本迈入 aarch64 时代，但是感谢伟大的 rosetta，x86_64 的 JIT 仍然可以在有 rosetta 的 aarch64 macOS 设备上使用。

在本文中，你可以了解如下重点内容：

- 如何开辟可读可写 / 可读可执行的内存空间？
- brainfuck 操作符到 x86_64 汇编的映射表
- brainfuck 也是可以优化的！但是该怎么优化？

本文使用的样例代码仓库可以到文末查看。

__警告__: 接下来的内容需要一定的前置知识储备：x86_64 汇编 / x86_64 调用约定。

__警告__: AT&T 汇编出没，如有不适请及时回避 ToT

## Brainfuck 解释器

本文样例代码中也提供了 switch-threading 指令分派的简单解释器：

```C++
for (size_t i = 0; i < code.size(); ++i) {
    switch (code[i].op) {
        case op_add: buff[p] += code[i].num; break;
        case op_sub: buff[p] -= code[i].num; break;
        case op_addp: p += code[i].num; break;
        case op_subp: p -= code[i].num; break;
        case op_jt: if (buff[p]) i = code[i].num; break;
        case op_jf: if (!buff[p]) i = code[i].num; break;
        case op_in: buff[p] = getchar(); break;
        case op_out: putchar(buff[p]); break;
    }
}
```

### 基础优化：合并操作符

从加减法指令的操作可以看到我们提供的第一个最基础的优化，那就是合并操作符，该基础优化可以防止不必要的代码重复执行，一定程度上可以提升执行效率：

|bf code|opcode|
|:----|:----|
|`+++`|`buf[p] += 3`|
|`----`|`buf[p] -= 4`|
|`>>>>>`|`p += 5`|
|`<<`|`p -= 2`|

## Just-In-Time 即时编译器

### mmap

JIT 编译器的精华之一在于`mmap`，我们需要使用这个函数来申请一块可修改权限的内存。这块内存在我们生成机器码时，必须拥有可读可写权限；在执行机器码前，我们必须将其设置为可读可执行权限。这里我们同时提供了 windows 平台申请该内存的方式。

通过宏可以看到，windows 对于申请内存的权限管理的相对宽松，可以直接申请可读可写可执行的内存，实际上 linux 平台也可以这样操作，但是 macOS 在某次更新之后禁用了同时可读可写可执行的写法，所以我们对非 windows 平台还是采取比较保险的写法，先设置为可读可写。

```C++
#ifndef MAP_JIT
#define MAP_JIT 0
#endif

#ifdef _WIN32
    mem = (uint8_t*)VirtualAlloc(nullptr, size,
        MEM_COMMIT | MEM_RESERVE,
        PAGE_EXECUTE_READWRITE);
#else
    mem = (uint8_t*)mmap(nullptr, size,
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS | MAP_JIT,
//                                    ^^^^^^^ 注意，linux 没有该宏，需要自己创建一个
        -1, 0);
#endif
    if (!mem) {
        std::cout << "failed to allocate memory\n";
        std::exit(-1);
    }
    memset(mem, 0, size);
```

而执行也是相当简单，我们生成的机器码实际上是一整个“函数”，所以直接作为函数来调用即可，当然，在 unix 系统中我们需要把内存的权限改成可读可执行：

```C++
#ifndef _WIN32
    mprotect(mem, size, PROT_READ | PROT_EXEC);
//                      ^^^^^^^^^^^^^^^^^^^^^ 伟大的 macOS 要求我们这样做
#endif
    ((func)mem)();
```

接下来是机器码生成，首先在函数的入口和出口处需要保存寄存器数据到栈上，当然这里保存的明显有点多了，实际上只需要保存必要的寄存器，在函数结束处记得将寄存器信息恢复，保持栈平衡。

```C++
/* set stack and base pointer */
mem.push({0x55});             // pushq %rbp
mem.push({0x48, 0x89, 0xe5}); // mov %rsp, %rbp

/* save register context */
mem.push({0x57})              // pushq %rdi
   .push({0x56})              // pushq %rsi
   .push({0x53})              // pushq %rbx
   .push({0x52})              // pushq %rdx
   .push({0x51})              // pushq %rcx
   .push({0x50});             // pushq %rax
```

对于 bf 的“纸带”，我们偷个小懒，搞个全局变量糊弄一下`buff[0x20000]`，这里我们将 buff 的地址直接赋给`rbx`寄存器，表示该寄存器接下来将会作为 bf 的读写头使用。

```C++
/* set bf machine's paper pointer */
mem.push({0x48, 0xbb}).push64((uint64_t)buff); // movq $buff, %rbx
```

### 加减法指令

加减法指令相对简单，可以参考注释中的汇编。

```C++
case op_add:  mem.push({0x80, 0x03, (uint8_t)(op.num & 0xff)}); break; // addb $op.num, (%rbx)
case op_sub:  mem.push({0x80, 0x2b, (uint8_t)(op.num & 0xff)}); break; // subb $op.num, (%rbx)
case op_addp: mem.push({0x48, 0x81, 0xc3}).push32(op.num);    break; // add $op.num, %rbx
case op_subp: mem.push({0x48, 0x81, 0xeb}).push32(op.num);    break; // sub $op.num, %rbx
```

### 使用输入输出函数 `putchar` & `getchar`

#### putchar

```C++
int putchar(int);
```

对于输出指令`op_out`我们选择`putchar`，不过也有其他选择，总之能用就行了。但是在函数调用这里我们需要注意一下不同平台的调用约定，如果对调用约定不太确定，可以编写一个简单的 demo 代码，使用 gcc/clang 编译成汇编来查看具体平台会生成什么样的代码。

可以看到，unix 平台会使用`rdi`寄存器来保存第一个参数，但是 windows 平台使用的是`rcx`。不过幸好这是 x86_64，如果是 x86 32 位机器的调用约定，那就头大了……

```C++
mem.push({0x48, 0xb8}).push64((uint64_t)putchar); // movabs $putchar, %rax
#ifndef _WIN32
mem.push({0x0f, 0xbe, 0x3b}); // movsbl (%rbx), %edi
#else
mem.push({0x0f, 0xbe, 0x0b}); // movsbl (%rbx), %ecx
#endif
mem.push({0xff, 0xd0}); // callq *%rax
//                         ^^^^^^^^^^^ 这是间接调用，相当于使用函数指针进行函数调用
```

#### getchar

```C++
int getchar();
```

对于输出函数`op_in`我们选择使用`getchar`。还好，我们需要支持的三个平台对于返回值的调用约定都是一样的…… 我们只需要读取`rax`的低 8 位就可以了，所以这里是`movsbl %al` (这个指令不是在骂人)

```C++
mem.push({0x48, 0xb8})
   .push64((uint64_t)getchar); // movabs $getchar, %rax
mem.push({0xff, 0xd0});        // callq *%rax
mem.push({0x88, 0x03});        // movsbl %al, (%rbx)
```

### 跳转指令

跳转指令`je`和`jne`是代码生成中的难点，我们需要预留跳转地址的空位，在后续生成回跳的机器码时计算偏移量，并将其填入原先预留的空位中，一定要注意字节序！

```C++
amd64jit& amd64jit::je() {
    push({0x0f, 0x84}).push32(0x0); // je
    stk.push(ptr);
    return *this;
}

amd64jit& amd64jit::jne() {
    push({0x0f, 0x85}).push32(0x0); // jne
    uint8_t* je_next = stk.top();
    stk.pop();
    
    uint8_t* jne_next = ptr;
    uint64_t p0 = jne_next - je_next;
    uint64_t p1 = je_next - jne_next;
    jne_next[-4] = p1 & 0xff;
    jne_next[-3] = (p1 >> 8) & 0xff;
    jne_next[-2] = (p1 >> 16) & 0xff;
    jne_next[-1] = (p1 >> 24) & 0xff;
    je_next[-4] = p0 & 0xff;
    je_next[-3] = (p0 >> 8) & 0xff;
    je_next[-2] = (p0 >> 16) & 0xff;
    je_next[-1] = (p0 >> 24) & 0xff;
    return *this;
}
```

另外需要注意的是，`op_jf`也就是 bf 的`[`，使用的是`je`指令（`op_jt`就是反过来的），为啥是`jump equal`呢？因为我们在生成跳转指令之前，还生成了比较指令，用于查看当前读写头指向的数据是否为 0，`test`指令是将两个寄存器的值进行按位与操作，所以仅在al寄存器为 0 时，`test %al, %al`结果为 0，此时 CPU 的`ZF`位会被设置，接下来`je`指令会跳转到`]`后的代码执行。

```C++
case op_jt: // if (al)
    mem.push({0x8a, 0x03}); // mov (%rbx), %al
    mem.push({0x84, 0xc0}); // test %al, %al
    mem.jne();
    break;
case op_jf: // if (!al)
    mem.push({0x8a, 0x03}); // mov (%rbx), %al
    mem.push({0x84, 0xc0}); // test %al, %al
    mem.je();
    break;
```

### 总结

|bf code|opcode|machine code|
|:----|:----|:----|
|`+`|op_add|`addb $op.num, (%rbx)`|
|`-`|op_sub|`subb $op.num, (%rbx)`|
|`>`|op_addp|`add $op.num, %rbx`|
|`<`|op_subp|`sub $op.num, %rbx`|
|`[`|op_jf|`je label`|
|`]`|op_jt|`jne label`|
|`,`|op_in|`callq *%rax` & `movsbl %al, (%rbx)`|
|`.`|op_out|`callq *%rax`|

## 简单优化样例

Brainfuck 也可以通过一些简单的优化来提升执行效率，比如如下两个样例：

```bf
[-]
[+]
```

这两段代码的作用都是相同的，那就是把读写头当前指向的数据变为 0。所以我们可以加一个新的解释器指令，并为其生成新的机器码

|bf code|opcode|machine code|
|:----|:----|:----|
|`[-]`|op_setz|`movb $0, (%rbx)`|
|`[+]`|op_setz|`movb $0, (%rbx)`|

```c++
// movb $0, (%rbx)
case op_setz: mem.push({0xc6, 0x03, 0x00}); break;
```

## 更高级的优化

下列的各种优化 pattern 没有在本文使用的样例仓库中实现，放在此处仅为抛砖引玉。

这个样例来自于 Girls Band Compiler 群的一位大佬：

```bf
-<++>-
```

可以被优化为如下代码，从而通过基础的指令合并优化来进一步减少执行指令数量：

```bf
--<++>
```

这种优化被称为`canonicalization`（规范化？不知道翻译成什么） 。这种优化相当于是在分析 memory 与 variable。

另外这里还有一些其他的 pattern 是可被优化的，还有待各位进一步探索:

```bf
[-<+>]
[-<<->>]
[->>+<<]
[->++>>>+++++>++>+<<<<<<]
```

## 更多

想要适配不同架构的 CPU 吗，你也许需要这个网站: [godbolt.org](https://godbolt.org/)，通过该网站你可以快速通过编写样例代码来获取对应的汇编以及机器码。试试看支持 aarch64 的 macOS 吧！

本文使用的样例仓库，欢迎 star：[github.com/ValKmjolnir/brainfuck-jit](https://github.com/ValKmjolnir/brainfuck-jit)

想要加入 Girls Band Compiler 的激烈 ♂ 讨论吗，那就搜索 670301108
