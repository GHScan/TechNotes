# CPU并行计算入门 #
### 基本原理 ###
- 现代CPU一般有复数的运算部件，包括多组算数逻辑单元、并行的数据通路、多个处理器核心，要充分挖掘这些硬件的并行能力，需要从原问题中分离无依赖子问题、分派到不同的硬件单元上。典型的手段包括：
    1. 数据级并行(Data-Level Parallelism)：常见的技术叫SIMD(Single Instruction Multiple Data)，即处理器提供专门的指令集，允许用户在向量寄存器上做并行计算。
        - 用SIMD改写Map算子：Map算子对输入流中的每个元素应用一个单参数函数，得到一个变换后的输出流，这是非常适合SIMD优化的典型问题，许多数据/图像处理里面的简单算法都是Map算子的模型。
            - 标量版本：
            ```c++
                float transform(float value) {
                    return (value + 5) / 7;
                }

                vector<float> map(vector<float> const &input) {
                    vector<float> output(inupt.size());

                    for (size_t i = 0; i < input.size(); ++i) {
                        float input_value = input[i];

                        float output_value = transform(input_value);

                        output[i] = output_value;
                    }

                    return output;
                }
            ```
            - 向量版本：
            ```c++
                float4 float4_transform(float4 value) {
                    // float4_div(float4_add(value, (float4)5), (float4)value);
                    return (value + 5) / 7; 
                }

                vector<float> map(vector<float> const &input) {
                    assert(input.size() % 4 == 0);

                    vector<float> output(input.size());

                    for (size_t i = 0; i < input.size(); i += 4) {
                        float4 input_value = float4_load(&input[i]);

                        float4 output_value = float4_transform(input_value);

                        float4_store(&output[i], output_value);
                    }

                    return output;
                }
            ```
            可以看到，因为Map算子的所有元素不存在计算上的依赖，很适合做数据级并行，只需要将相邻元素放到SIMD的不同Lane当中计算即可。当要并行计算的元素不在连续地址上时，可以使用指令集提供的Gatther指令来进行vload、Scatter指令来作vstore。
        - 用SIMD改写Reduce算子：Reduce算子接受一个二元运算符，将整个输入数据流归纳为单个值，如果这里的二元运算符满足结合律甚至交换律，那也可以做并行加速。
            - 标量版本：
            ```c++
                float combine(float a, float b) {
                    return a + b;
                }

                float reduce(vector<float> const &input, float zero) {
                    float value = zero;

                    for (size_t i = 0; i < input.size(); ++i) {
                        float input_value = input[i];

                        value = combine(value, input_value);
                    }

                    return value;
                }
            ```
            - 向量版本：
            ```c++
                float4 float4_combine(float4 a, float4 b) {
                    // float4_add(a, b)
                    return a + b;
                }

                float horizontally_combine(float4 value) {
                    // value.val[0] + value.val[1] + value.val[2] + value.val[3]
                    return float4_horizontally_add(value);
                }

                float reduce(vector<float> const &input, float zero) {
                    assert(input.size() % 4 == 0);

                    float4 value = float4_dup(zero);

                    for (size_t i = 0; i < input.size(); i += 4) {
                        float4 input_value = float4_load(&input[i]);

                        value = float4_combine(value, input_value);
                    }

                    return horizontally_combine(value);
                }
            ```
            注意这里要求初始值是零元、二元运算符可交换、可结合，而C语言中的浮点运算由于精度限制不具备这些性质，故这里的SIMD改写引入了误差、在某些应用中不可接受。基于相同的理由，除非指定特定的编译选项，否则浮点数上的标量Reduce操作通常不会被编译器自动向量化。
        - 鉴于Map/Reduce算子强大的表达力、很多简单问题都可以规约为Map/Reduce序列，故有了以上对Map/Reduce算子的向量化手段，再辅以算子融合，非访存/分支密集的相当一部分算法可以被加速最多4倍(假设SIMD Lane数是4)。
    2. 指令级并行(Instruction-Level Parallelism)：由于现代CPU中往往有多套发射单元、算数逻辑/访存单元，故无论是标量还是向量算法，适当地循环展开，可以引入更多无关计算流、进一步挖掘指令级并行机会。
        - 这里将上面SIMD版本的map算子进一步改写，挖掘指令级并行能力：
        ```c++
            always_inline float4 float4_transform(float4 value) {
                // float4_div(float4_add(value, (float4)5), (float4)value);
                return (value + 5) / 7; 
            }

            vector<float> map(vector<float> const &input) {
                assert(input.size() % 8 == 0);

                vector<float> output(input.size());

                for (size_t i = 0; i < input.size(); i += 8) {
                    float4 input_value_0 = float4_load(&input[i + 0]);
                    float4 input_value_1 = float4_load(&input[i + 4]);

                    float4 output_value_0 = float4_transform(input_value_0);
                    float4 output_value_1 = float4_transform(input_value_1);

                    float4_store(&output[i + 0], output_value_0);
                    float4_store(&output[i + 4], output_value_1);
                }

                return output;
            }
        ```
        如果CPU有至少两套向量读写单元、加法单元、除法单元，支持双发射，辅以编译器的函数内联和指令调度优化，以上改写会很有效；即使CPU只有部分硬件单元有多套，如果默认的编译优化级别没有自动实施以上改写，那这里的改写都将因为指令级并行获得不同程度的加速。如果特定平台不支持向量Intrinsics或者编译器优化质量不高，以上优化必须由汇编程序员手工实施。更精密的指令级并行能力的挖掘，需要针对特定CPU微架构，基于硬件手册或测得的指令Latency/Throughput/Execution Port占用表来实施，参数数据的Cache驻留情况、特定级别的Cache读写延迟也需要纳入考虑。
    3. 线程级并行(Thread-Level Parallelism)：线程级并行是指将问题在高层次上拆分成无关子问题，分派给CPU的多个核心，以利用硬件多线程来掩盖访存延迟、或利用多核心的计算单元做并行计算。
        - 多线程版的Map算子：
        ```c++
            vector<float> multi_threaded_map(vector<float> const &input) {
                assert(input.size() % thread_count == 0);

                vector<float> output(input.size());

                size_t work_per_thread = input.size() / thread_count;

                vector<Job> jobs(thread_count);
                for (size_t tid = 0; tid < thread_count; ++i) {
                    Job job = 
                        thread_pool.launch_job([&]() {
                            size_t begin_idx = tid * work_per_thread;
                            size_t end_idx = begin_idx + work_per_thread;

                            map(&input[begin_idx], &input[end_idx], &output[begin_idx]);
                    });

                    jobs[tid] = job;
                }

                thread_pool.join_all(jobs);

                return output;
            }
        ```
        显然，这里作用在子问题的上的单线程版的map函数又可以做数据级/指令级并行优化。
        - 多线程版的Reduce算子：
        ```c++
            float multi_threaded_reduce(vector<float> const &input, float zero) {
                assert(input.size() % thread_count == 0);

                size_t work_per_thread = input.size() / thread_count;

                vector<float> tmp_result(thread_count);

                vector<Job> jobs(thread_count);
                for (size_t tid = 0; tid < thread_count; ++i) {
                    Job job = 
                        thread_pool.launch_job([&]() {
                            size_t begin_idx = tid * work_per_thread;
                            size_t end_idx = begin_idx + work_per_thread;

                            tmp_result[tid] = reduce(&input[begin_idx], &input[end_idx], zero);
                    });

                    jobs[tid] = job;
                }

                thread_pool.join_all(jobs);

                return reduce(&tmp_result[0], &tmp_result[thread_count], zero);
            }
        ```
        这里的线程级并行至少要求Reduce的二元运算符满足结合性。
    4. 任务级并行(Request-Level Parallelism)：任务级并行通常在并发系统中讨论，当系统中确实存在大量无关任务时(如服务器)，通常任务级并行比线程级并行的并行效率更高(更高的吞吐)；除了并行的CPU资源外，任务级并行也被用于复用多路IO资源和多种硬件资源的流水化。
- 本文主要侧重上面前三种并行，尤其是ARM和x86-64的数据级并行(SIMD)。
### ARM上的SIMD ###
- 用编译器Intrinsics做向量化
    - NEON Intrinsics的介绍：Advanced SIMD(NEON)指令集是ARM的SIMD扩展，用来在一条指令中操作2到4路数据。[NEON Intrinsics Reference](https://developer.arm.com/technologies/neon/intrinsics)中包含了编译器提供的NEON内置函数的参考，这些函数在ARMv7/ARMv8指令集下会被分别编译到不同的NEON指令、并做适当的寄存器分配/编译器调度等优化，故要想一次编码支持两个指令集，NEON Intrinsics是最快捷的选项。
        - NEON内置函数的参数数据类型：int32x4_t，该种变量通常会被存储到4路int32的SIMD寄存器中；float32x2_t，被置于2路float32的NEON寄存器中；uint64x2_t，置于2路uint64的NEON寄存器中。
        - NEON内置函数的命名规范：v**op**{q}_**type**，其中op是操作名，type是每个SIMD Lane中的数据类型，可选的q表示操作数是否是128 bits的NEON寄存器(对应4路float32)，如vaddq_f32用于两个4路fp32的加法。由于近年的ARM处理器都支持4路SIMD的功能单元，所以编码上最好都用带q的函数。
    - 常用访存函数
        - 向量读写：vld1q_f32/vst1q_f32，从内存连续读写4个float32到float32x4_t的变量中。
        - 交错读写：vld3q_f32/vst3q_f32，将连续的4个三元组读到3个float32x4_t的变量中，4个三元组的第0/1/2个元素分别放到第0/1/2个变量中。比如如果内存中有4个RGB的float数的话(12个浮点数)，vld3q_f32会返回三个变量，其中第一个float32x4_t中是4个点的R通道数据。类似的还有vld2q_f32/vld4q_f32。
        - 标量广播：vld1q_dup_f32，从内存中读取一个浮点数，把值广播到float32x4_t的4个Lane中。
    - 常用计算函数
        - 算数运算：vaddq_f32/vsubq_f32/vmulq_f32/vdivq_f32分别用于加减乘除，其中除法开销巨大应该尽量避免(改为查倒数表或用大误差的倒数指令)。vmlaq_f32/vmlsq_f32分别表示乘加(`c=a*b+c`)、乘减指令，因为这种融合指令有专门的硬件单元，所以是比先乘后加的两次调用更好的选择。
        - 特殊函数：vmaxq_f32/vminq_f32/vsqrt_f32/vabsq_f32/vrecpeq_f32分别用于求最大、最小、开方、绝对值、倒数估计。
        - 位运算：vandq_u32/vorrq_u32/veorq_u32/vmvnq_u32分别用于位与、或、异或、非。
        - 移位：vshlq_n_u32/vshrq_n_u32分别用于左移右移。
        - 比较和条件选择：vcgeq_f32/vcltq_f32/vceqq_f32分别用于大于等于、小于、等于的比较，返回uint32x4_t，每个Lane全1表示比较算子结果为true，这个比较结果可以用在vbslq_f32作条件选择。
        - 类型转换：vcvt_f32_u32/vreinterpret_f32_u32分别用于u32到f32的转换和重新解释，后者不生成任何指令。
        - 跨Lane重排：vget_low_f32/vtrn1q_f32/vzip1q_f32分别用于获取2个低SIMD Lane、转置、Zip。
- 用内联汇编在ARMv7下做向量化
    - 由于ARMv7指令集寄存器个数更少、4通道SIMD指令的寻址方式受限，使用NEON Intrinsics函数依靠编译器输出的目标码质量往往不高，所以ARMv7下需要自己编写内联汇编来改进性能。[Arm NEON programming quick reference](https://community.arm.com/android-community/b/android/posts/arm-neon-programming-quick-reference)是一篇ARM内联汇编的很好的入门文章，在了解内联汇编的基本语法和ARMv7的寄存器/指令格式后，要将一个新算法向量化，可以先为它编写NEON Intrinsics版本的实现，直接用于ARMv8平台，另外再编译一份到ARMv7下然后反编译，根据反编译的结果，看编译器具体使用了什么样的指令，再在此基础上重新规划寄存器使用和做手工指令调度。
- 更多资料：
    - [ARM® Cortex™-A Series Programmer’s Guide](https://www.macs.hw.ac.uk/~hwloidl/Courses/F28HS/Docu/DEN0013D_cortex_a_series_PG.pdf)，尤其是第17章Optimizing Code to Run on ARM Processors。
    - [ARM® Compiler armasm User Guide](http://infocenter.arm.com/help/topic/com.arm.doc.dui0801g/DUI0801G_armasm_user_guide.pdf)，同时提供了ARMv7/ARMv8最全的指令参考(A32/A64)。
    - [NEON Programmers Guide](https://static.docs.arm.com/den0018/a/DEN0018A_neon_programmers_guide_en.pdf)：6/7/8章有些简单的NEON例子。
    - [ARM® Cortex®-A57 Software Optimization Guide](http://infocenter.arm.com/help/topic/com.arm.doc.uan0015b/Cortex_A57_Software_Optimization_Guide_external.pdf)，[ARM® Cortex®-A55 Software Optimization Guide](https://static.docs.arm.com/epm128372/20/arm_cortex_a55_software_optimization_guide_v2.pdf)：CPU微架构相关的优化建议。
### x86-64上的SIMD ###
- x86-64上因为有[Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#)这个功能强大的网站的存在，只需要按照本文原理一章讲的向量化方法，结合该网站中的指令集筛选(SSE4.2/AVX2)、指令类型筛选(Load/Store/Arithmetic/Bit Manipulation/Compare)功能，就能很好的向量化算法、选择合适的指令了。
- 更多资料：
    - [Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 1: Basic Architecture](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-1-manual.pdf)，[The microarchitecture of Intel, AMD and VIA CPUs](https://www.agner.org/optimize/microarchitecture.pdf)：CPU微架构相关信息。
    - [Optimizing software in C++](https://www.agner.org/optimize/optimizing_cpp.pdf)，[Optimizing subroutines in assembly language](https://www.agner.org/optimize/optimizing_assembly.pdf)：C++、x86-64汇编优化讨论。
    - [Instruction tables](https://www.agner.org/optimize/instruction_tables.pdf)：各CPU微架构下的指令Latency/Througput数据。
### OpenMP的使用 ###
- OpenMP API提供了DLP/TLP/RLP的能力，在大多数C/C++编译器上都支持，在不追求极致性能的场合下是一种便捷的并行方案。
- 数据级并行：可以用`#pragma omp simd`来指定某个循环做数据级并行，这在很多时候是比编译器的自动向量化更可靠的方法：
    ```c++
    #pragma omp simd
    for (size_t i = 0; i < N; ++i) {
        a[i] = a[i] + b[i] * c[i];
    }
    ```
- 线程级并行：可以通过`#pragma omp parallel for`来将一个大循环分解成多个子循环、交给不同线程去计算：
    ```c++
    #pragma omp parallel for
    for (size_t i = 0; i < N; ++i) {
        a[i] = a[i] + b[i] * c[i];
    }
    ```
    如果是Reduce算子，除了自己维护thread local的归纳变量外，也可以直接用OpenMP的reduction clause。
- 更多资料
    - [Guide into OpenMP: Easy multithreading programming for C++](https://bisqwit.iki.fi/story/howto/openmp/)：一文罗列了更多OpenMP的Advanced用法。
    - [OpenMP Specifications](https://www.openmp.org/specifications/)：OpenMP标准。
    - [OpenMP Compilers & Tools](https://www.openmp.org/resources/openmp-compilers-tools/)：各编译器的OpenMP支持情况。
### 总结 ###
- 本文介绍了CPU并行计算的基本原理，讲了基本SIMD向量化框架在ARM/x86-64两种指令集下的对应物、手册的查询和使用方法，然后简单介绍了下用OpenMP做数据级/线程级并行的语法。由于本文定位于入门，虽也罗列了部分进阶资料，但进一步深入CPU并行计算，除了研读这些硬件/指令级手册外，还需要学习高性能计算相关理论和方法，这些方法在某些场景中能取得进一步地加速。
