# 别名

首先，让我们先说一些重要的注意事项：

* 为了便于讨论，我们将使用最广泛的别名定义。Rust 的定义可能会有更多限制，以考虑到可变性和有效性。

* 我们将假设一个单线程的、无中断的执行，我们还将忽略像内存映射硬件这样的东西。Rust 假定这些事情不会发生，除非你明确告诉它会发生。更多细节，请参阅[并发性章节](concurrency.html)。

所以，我们现行的定义是：如果变量和指针指向内存的重叠区域，那么它们就是*别名*。

## 为什么别名很重要

为什么我们需要关注别名呢？

让我们看下这个例子：

```rust
fn compute(input: &u32, output: &mut u32) {
    if *input > 10 {
        *output = 1;
    }
    if *input > 5 {
        *output *= 2;
    }
    // remember that `output` will be `2` if `input > 10`
}
```

我们*希望*能够把它优化成下面这样的函数：

```rust
fn compute(input: &u32, output: &mut u32) {
    let cached_input = *input; // keep `*input` in a register
    if cached_input > 10 {
        // If the input is greater than 10, the previous code would set the output to 1 and then double it,
        // resulting in an output of 2 (because `>10` implies `>5`).
        // Here, we avoid the double assignment and just set it directly to 2.
        *output = 2;
    } else if cached_input > 5 {
        *output *= 2;
    }
}
```

在 Rust 中，这种优化应该是可行的。但对于几乎任何其他语言来说，它都不是这样的（除非是全局分析）。这是因为这个优化依赖于知道别名不会发生，而大多数语言在这方面是相当宽松的。具体来说，我们需要担心那些使“输入”和“输出”重叠的函数参数，如`compute(&x, &mut x)`。

如果按照这样的输入，我们实际上执行的代码如下：

<!-- ignore: expanded code -->
```rust,ignore
                    //  input ==  output == 0xabad1dea
                    // *input == *output == 20
if *input > 10 {    // true  (*input == 20)
    *output = 1;    // also overwrites *input, because they are the same
}
if *input > 5 {     // false (*input == 1)
    *output *= 2;
}
                    // *input == *output == 1
```

我们的优化函数对于这个输入会产生`*output == 2`，所以在这种情况下，我们的优化就无法实现了。

在 Rust 中，我们知道这个输入是不可能的，因为`&mut`不允许被别名。所以我们可以安全地认为这种情况不会发生，并执行这个优化。在大多数其他语言中，这种输入是完全可能的，因此必须加以考虑。

这就是为什么别名分析很重要的原因：它可以让编译器进行有用的优化! 比如：

* 通过证明没有指针访问该值的内存来保持寄存器中的值
* 通过证明某些内存在我们上次读取后没有被写入，来消除读取
* 通过证明某些内存在下一次写入之前从未被读过，来消除写入
* 通过证明读和写之间不相互依赖来对指令进行移动或重排序

这些优化也用于证明更大的优化的合理性，如循环矢量化、常数传播和死代码消除。

在前面的例子中，我们利用`&mut u32`不能被别名的事实来证明对`*output`的写入不可能影响`*input`。这让我们把`*input`缓存在一个寄存器中，省去了读的过程。

通过缓存这个读，我们知道在`> 10`分支中的写不能影响我们是否采取`> 5`分支，使我们在`*input > 10`时也能消除一个读-修改-写（加倍`*output`）。

关于别名分析，需要记住的关键一点是，写是优化的主要危险。也就是说，阻止我们将读移到程序的任何其他部分的唯一原因是我们有可能将其与写到同一位置重新排序。

例如，在下面这个修改后的函数中，我们不需要担心别名问题，因为我们已经将唯一一个写到`*output`的地方移到了函数的最后。这使得我们可以自由地重新排序在它之前发生的对`*input`的读取：

```rust
fn compute(input: &u32, output: &mut u32) {
    let mut temp = *output;
    if *input > 10 {
        temp = 1;
    }
    if *input > 5 {
        temp *= 2;
    }
    *output = temp;
}
```

我们仍然依靠别名分析来假设`temp`没有别名`input`，但是证明要简单得多：局部变量的值不能被在它被声明之前就存在的东西所别名。这是每一种语言都可以自由做出的假设，因此这个版本的函数可以在任何语言中按照我们想要的方式进行优化。

这就是为什么 Rust 将使用的“别名”的定义可能涉及到一些有效性和可变性的概念：如果没有任何实际写入内存的情况发生，我们实际上并不关心别名是否发生。

当然，Rust 的完整别名模型还必须考虑到函数调用（可能会改变我们看不到的东西）、原始指针（它本身没有别名要求）和 UnsafeCell（它让`&`的引用被改变）等东西。
