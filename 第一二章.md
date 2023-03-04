时间：2022-11-21

第一章

主要介绍uvm发展史，验证内容

第二章

+ 验证平台组成
+ 最简单的验证平台：my_driver与top_tb组成

## 第一章

### 1.2 学了UVM之后能做什么

+ 使用sequence机制、factory机制、callback机制、寄存器模型（register model）等
+ 验证基本常识
+ 如何实现代码可重复性

### 机制索引

+ phase机制（5.1）
+ objection机制（5.2）
+ sequence机制（2.4, 6）
+ facory机制（8）
+ file automation机制
+ config_db机制
+ callback机制（9）

### 特性检索

+ TLM
+ drain_time，撤销objection的延时（5.2.4）
+ domain（5.3）
+ register model

## 第二章

### 2.1 验证平台组成

验证用于找出DUT中的bug，这个过程通常是把DUT放入一个验证平台中来实现的

一个验证平台要实现如下基本功能：

+ 激励的功能：driver
  + 各种激励：正常激励、异常激励
+ 判断是否符合预期：scoreboard/checker
+ 收集DUT输出并传递给sb：monitor
+ reference model

![image-20221020170035636](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221020170035636.png)

### 2.2 只有driver的验证平台

#### 2.2.1 最简单的验证平台

**1. driver是验证平台最基本的组件**

> **如何搭建driver?**
>
> + UVM是一个库，在这个库中，几乎所有的东西都是使用类（class）来实现的，如：driver、 monitor、reference model、scoreboard等组成部分都是类
> + **类有函数（function）和任务（task）**，通过这些函数和任务可以完成driver的输出激励功能
> + 使用UVM的第一条原则是：**验证平台中所有的组件应该派生自UVM中的类**

**2. 定义一个my_driver类**

```verilog
`ifndef MY_DRIVER__SV
`define MY_DRIVER__SV
class my_driver extends uvm_driver;

   function new(string name = "my_driver", uvm_component parent = null);
      super.new(name, parent);
   endfunction
   extern virtual task main_phase(uvm_phase phase);
endclass

task my_driver::main_phase(uvm_phase phase);
   top_tb.rxd <= 8'b0; 
   top_tb.rx_dv <= 1'b0;
   while(!top_tb.rst_n)
      @(posedge top_tb.clk);
   for(int i = 0; i < 256; i++)begin
      @(posedge top_tb.clk);
      top_tb.rxd <= $urandom_range(0, 255);
      top_tb.rx_dv <= 1'b1;
      `uvm_info("my_driver", "data is drived", UVM_LOW)
   end
   @(posedge top_tb.clk);
   top_tb.rx_dv <= 1'b0;
endtask
`endif
```

> new参数
>
> + uvm_driver的类的new函数有两个参数（这两个参数是由uvm_component要求的），一个是string类型的name，一个是uvm_component类型的parent
> + 每一个派生自uvm_component或其派生类的类在其new函数中要指明两个参数：name和parent

> phase
>
> + driver所做的事情几乎都在main_phase中完成
> + UVM由phase来管理验证平台的运行，这些phase统一以xxxx_phase来命名，且都有一个类型为uvm_phase、名字为phase的参数
>
> + **可以简单地认为， 实现一个driver等于实现其main_phase**

> ``uvm_info`宏：与Verilog中display语句的功能类似，比display语句更加强大
>
> + 三个参数，第一个参数是字符串，用于把打印的信息归类；第二个参数也是字符串，是具体需要打印的信息；第三个参数则是冗余级别
> + 非常关键可设置为UVM\_LOW，可有可无可设置为UVM_HIGH，介于两者之间是UVM_MEDIUM。UVM默认只显示UVM_MEDIUM或者UVM_LOW的信息，本书3.4.1节会讲
>
> > **uvm_info打印内容**
> >
> > ```
> > UVM_INFO my_driver.sv(20)@48500000：drv[my_driver]data is drived
> > ```
> >
> > + my_driver.sv(20)：指明此条打印信息的来源，其中括号里的数字表示原始的uvm_info打印语句在my_driver.sv中的行号
> > + 48500000：表明此条信息的打印时间
> > + drv：这是driver在UVM树中的路径索引
> >   + UVM采用树形结构，对于树中任何一个结点，都有一个与其相应的字符串类型的 路径索引。路径索引可以通过`get_full_name`函数来获取，把下列代码加入任何UVM树的结点中就可以得知当前结点的路径索引：`$display("the full name of current component is: %s", get_full_name());`
> >     + [my_driver]：方括号中显示的信息即调用uvm_info宏时传递的第一个参数
> >     + data is drived：表明宏最终打印的信息

**3. 实例化一对象**

```verilog
`timescale 1ns/1ps
`include "uvm_macros.svh"

import uvm_pkg::*;
`include "my_driver.sv"

module top_tb;

reg clk;
reg rst_n;
reg[7:0] rxd;
reg rx_dv;
wire[7:0] txd;
wire tx_en;

dut my_dut(.clk(clk),
           .rst_n(rst_n),
           .rxd(rxd),
           .rx_dv(rx_dv),
           .txd(txd),
           .tx_en(tx_en));

initial begin
   my_driver drv;
    drv = new("drv", null);//new的时候第二个参数一般不为null,这里省略所以为null
   drv.main_phase(null);
   $finish();
end

initial begin
   clk = 0;
   forever begin
      #100 clk = ~clk;
   end
end

initial begin
   rst_n = 1'b0;
   #1000;
   rst_n = 1'b1;
end

endmodule
```

+ ``include uvm_macros.svh` 包含了众多的宏定义，只需要包含一 次
+ `import uvm_pkg::*;`导入uvm_pkg库
+ new的时候第二个参数一般不为null
+ 显示调用`main_phase`
+ `$finish();`结束仿真

#### 2.2.2 加入factory机制

上节的实例化及main_phase的显式调用，只使用简单的SystemVerilog也可完成

本节介绍：自动创建一个类的实例并调用其中的函数（function）和任务（task）

---

**1. 使用factory机制重写my_driver**

```verilog
`ifndef MY_DRIVER__SV
`define MY_DRIVER__SV
class my_driver extends uvm_driver;

   `uvm_component_utils(my_driver)
   function new(string name = "my_driver", uvm_component parent = null);
      super.new(name, parent);
      `uvm_info("my_driver", "new is called", UVM_LOW);
   endfunction
   extern virtual task main_phase(uvm_phase phase);
endclass

task my_driver::main_phase(uvm_phase phase);
   `uvm_info("my_driver", "main_phase is called", UVM_LOW);
   top_tb.rxd <= 8'b0; 
   top_tb.rx_dv <= 1'b0;
   while(!top_tb.rst_n)
      @(posedge top_tb.clk);
   for(int i = 0; i < 256; i++)begin
      @(posedge top_tb.clk);
      top_tb.rxd <= $urandom_range(0, 255);
      top_tb.rx_dv <= 1'b1;
      `uvm_info("my_driver", "data is drived", UVM_LOW);
   end
   @(posedge top_tb.clk);
   top_tb.rx_dv <= 1'b0;
endtask
`endif

```

> factory机制之宏``uvm_component_utils`
>
> 这个宏所做的事情非常多，其中之一就是将my_driver登记在 UVM内部的一张表中
>
> <mark>所有派生自uvm_component及其派生类的类都应该使用uvm_component_utils宏注册</mark>
>
> > 这张表是factory功能实现的基础。只要在定义一个新的类时使用这个宏，就相当于把这个类注册到了这张表中。有关内容深入解释在后面

**2. 使用factory机制重写top_tb**

```verilog
...

module top_tb;

...
    
initial begin
   run_test("my_driver");
end

endmodule
```

运行top_tb后输出：

```
new is called
main_phased is called
```

并没有输出："data is drived"，关于这个问题，牵涉UVM的objection机制

> **`run_test("注册的类名")`**
>
> 使用`run_test("注册的类名")`代替my_driver实例化与显式调用
>
> + run_test创建一个my_driver的实例，并且会自动调用my_driver的main_phase

#### 2.2.3 加入objection机制

上一节只输出了“main_phase is called”，但是“data is drived”并没有输出，因为main_phase被杀死了

---

**1. objection机制**

> UVM中通过objection机制来控制验证平台的关闭，在每个phase中，UVM会检查是否有objection被提起 (`raise_objection`)。如果有，则等待这个objection被撤销(`drop_objection`)后停止仿真；如果没有，则马上结束当前phase
>
> + `raise_objection`和`drop_objection`总是成对出
> + `raise_objection`语句必须在main_phase中第一个消耗仿真时间 [1]的语句之前
> + 
>
> > 所谓仿真时间，是指$time函数打印出的时间。与之相对的还有实际仿真中所消耗的CPU时间，通常说一个测试用例的运行时间 即指CPU 时间，为了与仿真时间相区分，本书统一把这种时间称为运行时间。
> >
> > 如`$display`语句是不消耗仿真时间的，这些语句可 以放在`raise_objection`之前，但是类似`@(posedge top.clk)`等语句是要消耗仿真时间的。按照如下的方式使用`raise_objection`是无法
> > 起到作用的

2. 使用objection机制的my_driver

```verilog
`ifndef MY_DRIVER__SV
`define MY_DRIVER__SV
class my_driver extends uvm_driver;

	...
endclass

task my_driver::main_phase(uvm_phase phase);
   	phase.raise_objection(this);
   
    ...
   
    phase.drop_objection(this);
endtask
`endif
```

#### 2.2.4 加入virtual interrface

**1. 绝对路径遇到的问题：**

driver中等待时钟事件（@posedge top.clk）、给DUT中输入端口赋值（top.rx_dv<=1‘b1）都是使用绝对路径，绝对路径的使用大大减弱了验证平台的可移植性。一个最简单的例子就是假如clk信号的层次从top.clk变成了top.clk_inst.clk，那么就需要对driver中的相关代码做大量修改。因此，从根本上来说，应该尽量杜绝在验证平台中使用绝对路径

**2. 解决方案①使用宏**

使用``TOP`代替上文中的top，但是如果clk_inst变了照样解决不了，还要改

**3. 解决方法②使用接口**

定义接口

```verilog
`ifndef MY_IF__SV
`define MY_IF__SV

interface my_if(input clk, input rst_n);

   logic [7:0] data;
   logic valid;
endinterface

`endif

```

在top_tb中实例化DUT时，就可以直接使用接口

```verilog
module top_tb;

...

my_if input_if(clk, rst_n);
my_if output_if(clk, rst_n);

dut my_dut(.clk(clk),
           .rst_n(rst_n),
           .rxd(input_if.data),
           .rx_dv(input_if.valid),
           .txd(output_if.data),
           .tx_en(output_if.valid));

...

endmodule
```

**4. 如何在driver（类）中使用接口**

不能直接，这种方式只能在module模块中才能实现

```verilog
class my_driver extends uvm_driver;
    my_if drv_if;
    ...
endclass
```

在类中应该使用虚接口virtual interface：

```verilog
class my_driver extends uvm_driver;

   virtual my_if vif;
   ...
endclass
```

替代后的全部my_driver代码如下，可见绝对路径已消除

```verilog
class my_driver extends uvm_driver;

   virtual my_if vif;

   `uvm_component_utils(my_driver)
   function new(string name = "my_driver", uvm_component parent = null);
      super.new(name, parent);
      `uvm_info("my_driver", "new is called", UVM_LOW);
   endfunction

   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      `uvm_info("my_driver", "build_phase is called", UVM_LOW);
      if(!uvm_config_db#(virtual my_if)::get(this, "", "vif", vif))
         `uvm_fatal("my_driver", "virtual interface must be set for vif!!!")
   endfunction

   extern virtual task main_phase(uvm_phase phase);
endclass

task my_driver::main_phase(uvm_phase phase);
   phase.raise_objection(this);
   `uvm_info("my_driver", "main_phase is called", UVM_LOW);
   vif.data <= 8'b0; 
   vif.valid <= 1'b0;
   while(!vif.rst_n)
      @(posedge vif.clk);
   for(int i = 0; i < 256; i++)begin
      @(posedge vif.clk);
      vif.data <= $urandom_range(0, 255);
      vif.valid <= 1'b1;
      `uvm_info("my_driver", "data is drived", UVM_LOW);
   end
   @(posedge vif.clk);
   vif.valid <= 1'b0;
   phase.drop_objection(this);
endtask
```

**5. 如何链接top_tb的input_if与my_driver中的vif(<mark>config_db机制</mark>)**

使用`run_test`进行实例化，从而无法直接引用my_driver实例

**使用`config_db`机制**

> <mark>**config_db机制：**</mark>
>
> config_db用来在端口间传递数据（端口级），分为set和get两步操作
>
> + `uvm_config_db#(?)::set`操作，可以简单地理解成是“寄信”
> + `uvm_config_db#(?)::get`则相当于是“收信”

在top_tb中执行set操作：

```verilog
module top_tb;

...

my_if input_if(clk, rst_n);
my_if output_if(clk, rst_n);

dut my_dut(.clk(clk),
           .rst_n(rst_n),
           .rxd(input_if.data),
           .rx_dv(input_if.valid),
           .txd(output_if.data),
           .tx_en(output_if.valid));

...

initial begin
   uvm_config_db#(virtual my_if)::set(null, "uvm_test_top", "vif", input_if);
end

endmodule
```

在my_driver中，执行get操作：

```verilog
class my_driver extends uvm_driver;

   virtual my_if vif;

   ...

   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      `uvm_info("my_driver", "build_phase is called", UVM_LOW);
      if(!uvm_config_db#(virtual my_if)::get(this, "", "vif", vif))
         `uvm_fatal("my_driver", "virtual interface must be set for vif!!!")
   endfunction

endclass
```

> `build_phase`
>
> + 定义：与main_phase一样，build_phase也是UVM中内建的一个phase
> + 执行顺序：当UVM启动后，会自动执行 build_phase，运行在new之后，main_phase之前
> + 主要作用：主要通过config_db的set和get操作来传递一些数据， 以及实例化成员变量等
> + 使用注意事项：需要在build_phase中调用`super.build_phase(phase)`，因为父类的build_phase中执行了一些必要的操作，这里必须显式调用并执行
> + 与main_phase不同：main_phase是任务，build_phase是函数不消耗仿真时间

> **``uvm_fatal`宏**
>
> + 参数：两个与``uvm_info`前两个参数意义一样
> + 主要作用：表示验证平台出现了重大问题而无法继续下去，必须停止仿真并做相应的检查
> + 与``uvm_info`不同：会调用`$finish()`结束仿真

> `config_db`的参数
>
> + config_db的set和get函数都有四个参数，且第三个参数必须完全一致
>
> `uvm_config_db::set`
>
> + 第二个参数表示的是路径索引(2.2.1节提到``uvm_info`中的路径索引)
>   + run_test创建的实例名为`uvm_test_top`，无论传给run_test参数是什么创建的实例名都是uvm_test_top
> + 第四个参数表示要将哪个interface 通过config_db传递给my_driver
>
> `uvm_config_db::get`
>
> + 第四个参数表示把得到的interface传递给哪个my_driver的成员变量
>
> 参数化：
>
> + `uvm_config_db#(virtual my_if)`是一个参数化的类，其参数就是要寄信的类型
> + 如果要传递一个int类型：`uvm_config_db#(int)::set`

### 2.3 为验证平台加入各个组件

2.2节的操作基于信号级，本节引入scoreboard、reference model、monitor这些组件是基于transaciton

#### 2.3.1 加入<mark>transaction</mark>

transaction的概念：transaction就是用于模拟物理协议/以太网协议/...，一 笔transaction就是一个包

**1. 一个简单的transaction定义**

```verilog
`ifndef MY_TRANSACTION__SV
`define MY_TRANSACTION__SV

class my_transaction extends uvm_sequence_item;

   rand bit[47:0] dmac;
   rand bit[47:0] smac;
   rand bit[15:0] ether_type;
   rand byte      pload[];
   rand bit[31:0] crc;

   constraint pload_cons{
      pload.size >= 46;
      pload.size <= 1500;
   }

   function bit[31:0] calc_crc();
      return 32'h0;
   endfunction

   function void post_randomize();
      crc = calc_crc;
   endfunction

   `uvm_object_utils(my_transaction)

   function new(string name = "my_transaction");
      super.new();
   endfunction
endclass
`endif

```

> `uvm_sequence_item`
>
> my_transaction的基类是uvm_sequence_item，uvm_sequence_item的祖先是`uvm_object`

> `uvm_object_utils`
>
> + 不使用``uvm_component_utils ``而使用`` ` uvm_object_utils``  的原因：
>   + transaction不同于driver等组件，具有生命周期，这种类一般派生自uvm_object或uvm_object的派生类

> **`post_randomize`**
>
> + post_randomize是SystemVerilog中提供的一个函数，
>
> + 当某个类的实例的randomize函数被调用后，post_randomize会紧随其后无条件地被调用

**2. driver驱动transaction**

```verilog
`ifndef MY_DRIVER__SV
`define MY_DRIVER__SV
class my_driver extends uvm_driver;

   virtual my_if vif;

   `uvm_component_utils(my_driver)
   function new(string name = "my_driver", uvm_component parent = null);
      super.new(name, parent);
   endfunction

   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      if(!uvm_config_db#(virtual my_if)::get(this, "", "vif", vif))
         `uvm_fatal("my_driver", "virtual interface must be set for vif!!!")
   endfunction

   extern task main_phase(uvm_phase phase);
   extern task drive_one_pkt(my_transaction tr);
endclass

task my_driver::main_phase(uvm_phase phase);
   my_transaction tr;
   phase.raise_objection(this);
   vif.data <= 8'b0;
   vif.valid <= 1'b0;
   while(!vif.rst_n)
      @(posedge vif.clk);
   for(int i = 0; i < 2; i++) begin 
      tr = new("tr");
      assert(tr.randomize() with {pload.size == 200;});
      drive_one_pkt(tr);
   end
   repeat(5) @(posedge vif.clk);
   phase.drop_objection(this);
endtask

task my_driver::drive_one_pkt(my_transaction tr);
   bit [47:0] tmp_data;
   bit [7:0] data_q[$]; 
  
   //push dmac to data_q
   tmp_data = tr.dmac;
   for(int i = 0; i < 6; i++) begin
      data_q.push_back(tmp_data[7:0]);
      tmp_data = (tmp_data >> 8);
   end
   //push smac to data_q
   tmp_data = tr.smac;
   for(int i = 0; i < 6; i++) begin
      data_q.push_back(tmp_data[7:0]);
      tmp_data = (tmp_data >> 8);
   end
   //push ether_type to data_q
   tmp_data = tr.ether_type;
   for(int i = 0; i < 2; i++) begin
      data_q.push_back(tmp_data[7:0]);
      tmp_data = (tmp_data >> 8);
   end
   //push payload to data_q
   for(int i = 0; i < tr.pload.size; i++) begin
      data_q.push_back(tr.pload[i]);
   end
   //push crc to data_q
   tmp_data = tr.crc;
   for(int i = 0; i < 4; i++) begin
      data_q.push_back(tmp_data[7:0]);
      tmp_data = (tmp_data >> 8);
   end

   `uvm_info("my_driver", "begin to drive one pkt", UVM_LOW);
   repeat(3) @(posedge vif.clk);

   while(data_q.size() > 0) begin
      @(posedge vif.clk);
      vif.valid <= 1'b1;
      vif.data <= data_q.pop_front(); 
   end

   @(posedge vif.clk);
   vif.valid <= 1'b0;
   `uvm_info("my_driver", "end drive one pkt", UVM_LOW);
endtask

`endif
```

> 流程：
>
> ① 随机化：main_phase中执行transactin的randomize()
>
> ② 数据入列：main_phase执行数据入列函数，把transaction压入data_q
>
> > SystemVerlog提供的流操作符实现
> >
> > IEEE Standard for SystemVerilog—Unified Hardware Design，Specification，and Verification Language 的11.4.14章

#### 2.3.2 加入env

我们在哪对各种组件例化？

+ （不可行）top_tb中使用run_test不行，因为run_test只能实例化一个实例
+ （不可行）top_tb中直接new不行，这样run_test没意义了
+ （正确的）引入一个容器类，在这个类里面例化各种组件，此时run_test的参数不再是my_driver而是容器类。uvm中使用继承自<mark>`uvm_env`</mark>的子类，来表示这个容器类

**1. 定义my_env**

```verilog
class my_env extends uvm_env;

   my_driver drv;

   function new(string name = "my_env", uvm_component parent);
      super.new(name, parent);
   endfunction

   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      drv = my_driver::type_id::create("drv", this); 
   endfunction

   `uvm_component_utils(my_env)
endclass
```

> `类名::type_id::create`
>
> + factory机制带来的独特的实例化方式，只有factory注册过的类可以这种方式例化（代替new）
> + 好处：后面可以用到factory中强大的重载功能

**2. 树形结构——回顾my_driver参数2**

```verilog
function new(string name = "my_driver", uvm_component parent = null);
    super.new(name, parent);
endfunction
```

+ name：实例名称
+ parent：由于是在uvm_env例化，则my_driver的drv实例是my_env
  + <mark>**建立了树形结构**</mark>

![image-20221021221512291](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221021221512291-16663617135971.png)

**3. build_phase顺序**

<mark>build_phase的执行遵照从树根到树叶的顺序</mark>，当把整棵树的build_phase都执行完毕后，再执行后面的phase

因此，先执行my_env的build_phase，再执行my_driver的build_phase

**4. 修改config_db**

<mark>树根始终是run_test创建的uvm_test_top</mark>，这里uvm_test_top对象代表了一个my_env类

我们为啥要修改config_db？

答：我们加了一个env，你忘了吗

---

假如在my_env中实例化一个my_driver为my_drv：

```verilog
class my_env extends uvm_env
    ...
    drv = my_driver::type_id::create("my_drv", this);
endclass
```

则在top_tb.sv中修改为：

```verilog
module top_tb;
    ...
initial begin
    uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.my_drv", "vif", input_if);
    //原来是
    //uvm_config_db#(virtual my_if)::set(null, "uvm_test_top", "vif", input_if);
end

endmodule
```

#### 2.3.3 加入monitor

monitor作用：收集DUT行为

monitor内容：收集DUT端口数据，转换为transaction后交给后续组件（与driver相反）

driver内容：把transaction数据变为DUT端口级别，并驱动到DUT

---

```verilog
`ifndef MY_MONITOR__SV
`define MY_MONITOR__SV
class my_monitor extends uvm_monitor;

   virtual my_if vif;

   `uvm_component_utils(my_monitor)//my_monitor
   function new(string name = "my_monitor", uvm_component parent = null);
      super.new(name, parent);
   endfunction

   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      if(!uvm_config_db#(virtual my_if)::get(this, "", "vif", vif))
         `uvm_fatal("my_monitor", "virtual interface must be set for vif!!!")
   endfunction

   extern task main_phase(uvm_phase phase);
   extern task collect_one_pkt(my_transaction tr);
endclass

task my_monitor::main_phase(uvm_phase phase);
   my_transaction tr;
   while(1) begin
      tr = new("tr");
      collect_one_pkt(tr);
   end
endtask

task my_monitor::collect_one_pkt(my_transaction tr);
   bit[7:0] data_q[$]; 
   int psize;
   while(1) begin
      @(posedge vif.clk);
      if(vif.valid) break;
   end

   `uvm_info("my_monitor", "begin to collect one pkt", UVM_LOW);
   while(vif.valid) begin
      data_q.push_back(vif.data);
      @(posedge vif.clk);
   end
   //pop dmac
   for(int i = 0; i < 6; i++) begin
      tr.dmac = {tr.dmac[39:0], data_q.pop_front()};
   end
   //pop smac
   for(int i = 0; i < 6; i++) begin
      tr.smac = {tr.smac[39:0], data_q.pop_front()};
   end
   //pop ether_type
   for(int i = 0; i < 2; i++) begin
      tr.ether_type = {tr.ether_type[7:0], data_q.pop_front()};
   end

   psize = data_q.size() - 4;
   tr.pload = new[psize];
   //pop payload
   for(int i = 0; i < psize; i++) begin
      tr.pload[i] = data_q.pop_front();
   end
   //pop crc
   for(int i = 0; i < 4; i++) begin
      tr.crc = {tr.crc[23:0], data_q.pop_front()};
   end
   `uvm_info("my_monitor", "end collect one pkt, print it:", UVM_LOW);
    tr.my_print();
endtask


`endif

```

> 流程：
>
> 数据入列：main_phase执行数据入列函数，把transaction压入data_q

> <mark>永不停歇</mark>
>
> 由于monitor需要时刻收集数据，永不停歇，所以在main_phase中使用`while(1)`来实现

> 对比monitor中的collect_one_pkt与driver中的drv_one_pkt：
>
> + 两者代码非常相似
> + 当收集完一个transaction后， 通过my_print函数将其打印出来
>   + my_printf在my_transaction中定义

transaction中定义的my_printf：

```verilog
   function void my_print();
      $display("dmac = %0h", dmac);
      $display("smac = %0h", smac);
      $display("ether_type = %0h", ether_type);
      for(int i = 0; i < pload.size; i++) begin
         $display("pload[%0d] = %0h", i, pload[i]);
      end
      $display("crc = %0h", crc);
   endfunction
```

env中对组件的例化：

```verilog
class my_env extends uvm_env;

   my_driver drv;
   my_monitor i_mon;
   
   my_monitor o_mon;

   function new(string name = "my_env", uvm_component parent);
      super.new(name, parent);
   endfunction

   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      drv = my_driver::type_id::create("drv", this); 
      i_mon = my_monitor::type_id::create("i_mon", this);
      o_mon = my_monitor::type_id::create("o_mon", this);
   endfunction

    `uvm_component_utils(my_env)//注册my_env
endclass
```

> **实例化两个monitor**
>
> + 一个用于检测DUT输入
> + 一个用于检测DUT输出
>
> 为什么输入也monitor？
>
> 这个答案仁者见仁，智者见智。这里还是推荐使用monitor，原因是：
>
> + 第一，在一个大型的项目中，driver根据某一协议发送数据，而 monitor根据这种协议收集数据，如果driver和monitor由不同人员实现，那么可以大大减少其中任何一方对协议理解的错误
> + 第二，在后文将会看到，在实现**代码重用**时，使用monitor是非常有必要的

**2. 现在的树结构**

![image-20221021232129013](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221021232129013.png)

**3. top_tb中config_db修改**

使用config_db将input_if和output_if传递给两个monitor，从而完成端口连接

```verilog
   uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.drv", "vif", input_if);
   uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.i_mon", "vif", input_if);
   uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.o_mon", "vif", output_if);
```

#### 2.3.4 封装成agent

driver与monitor代码相似（本质处理同一种协议）

因此UVM常把二者封装在一起，成为<mark>agent</mark>

<mark>不同的agent就代表了不同的协议</mark>

---

**1. my_agent定义**

```verilog
`ifndef MY_AGENT__SV
`define MY_AGENT__SV

class my_agent extends uvm_agent ;
   my_driver     drv;
   my_monitor    mon;
   
   function new(string name, uvm_component parent);
      super.new(name, parent);
   endfunction 
   
   extern virtual function void build_phase(uvm_phase phase);
   extern virtual function void connect_phase(uvm_phase phase);

   `uvm_component_utils(my_agent)
endclass 


function void my_agent::build_phase(uvm_phase phase);
   super.build_phase(phase);
   if(is_active == UVM_ACTIVE) begin
       drv = my_driver::type_id::create("drv", this);
   end
   mon = my_monitor::type_id::create("mon", this);
endfunction 

function void my_agent::connect_phase(uvm_phase phase);
   super.connect_phase(phase);
endfunction

`endif
```

> `uvm_agent`
>
> + 所有agent派生自此类

> `is_active`
>
> + `is_active`是`uvm_agent`的一个成员变量，类型为`uvm_active_passive_enum`是一个枚举类型
> + 默认为`UVM_PASSIVE`
>
> `uvm_active_passive_enum`：
>
> + `typedef enum bit { UVM_PASSIVE=0, UVM_ACTIVE=1 } uvm_active_passive_enum;`
> + `UVM_PASSIVE`：意味着输入端口，无需驱动任何信号，只做检查信号(即只要monitor不要driver）
> + `UVM_ACTIVE`：意味着输出端口，需驱动任何信号

**2. 修改env中对driver和monitor的实例化（同时配置is_active）**

```verilog
class my_env extends uvm_env;

   my_agent  i_agt;
   my_agent  o_agt;
   
   function new(string name = "my_env", uvm_component parent);
      super.new(name, parent);
   endfunction

   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      i_agt = my_agent::type_id::create("i_agt", this);
      o_agt = my_agent::type_id::create("o_agt", this);
      i_agt.is_active = UVM_ACTIVE;
      o_agt.is_active = UVM_PASSIVE;
   endfunction

   `uvm_component_utils(my_env)
endclass
```

**3. 目前树结构**

![image-20221021234757273](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221021234757273.png)

**4. config_db修改端口连接**

```verilog
module top_tb;
	...
    initial begin
       uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.i_agt.drv", "vif", input_if);
       //原来是uvm_test_top.drv
       uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.i_agt.mon", "vif", input_if);
        //原来是uvm_test_top.i_mon
       uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.o_agt.mon", "vif", output_if);
        //原来是uvm_test_top.*_mon
    end
endmodule
```

**（补充）5.用config_db也可以传is_active**

背景：使用new实例化时，无法传递is_active。此时使用config_db机制可以传递is_active

注意！：UVM中约定俗成的还是在build_phase中完成实例化工作。<mark>因此，强烈建议仅在build_phase中完成实例化</mark>

```verilog
class my_env extends uvm_env;
   my_agent  i_agt;
   my_agent  o_agt;
   ...
   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      uvm_config_db#(uvm_active_passive_enum)::set(this, "i_agt", "is_active", UVM_ACTIVE);
      uvm_config_db#(uvm_active_passive_enum)::set(this, "o_agt", "is_active", UVM_PASSIVE);
      i_agt = my_agent::type_id::create("i_agt", this);
      o_agt = my_agent::type_id::create("o_agt", this);
   endfunction
endclass

class my_agent extends uvm_agent ;
   function new(string name, uvm_component parent);
      super.new(name, parent);
      uvm_config_db#(uvm_active_passive_enum)::get(this, "", "is_active", is_active);
      if (is_active == UVM_ACTIVE) begin
          drv = my_driver::type_id::create("drv", this);
      end
      mon = my_monitor::type_id::create("mon", this);
   endfunction 
endclass 

```

#### 2.3.5 加入reference model

作用：完成与DUT相同功能

输出：被scoreboard接收

---

**1. 定义my_model**

```verilog
`ifndef MY_MODEL__SV
`define MY_MODEL__SV

class my_model extends uvm_component;
   
   uvm_blocking_get_port #(my_transaction)  port;
   uvm_analysis_port #(my_transaction)  ap;

   extern function new(string name, uvm_component parent);
   extern function void build_phase(uvm_phase phase);
   extern virtual  task main_phase(uvm_phase phase);

   `uvm_component_utils(my_model)
endclass 

function my_model::new(string name, uvm_component parent);
   super.new(name, parent);
endfunction 

function void my_model::build_phase(uvm_phase phase);
   super.build_phase(phase);
   port = new("port", this);
   ap = new("ap", this);
endfunction

task my_model::main_phase(uvm_phase phase);
   my_transaction tr;
   my_transaction new_tr;
   super.main_phase(phase);
   while(1) begin
      port.get(tr);
      new_tr = new("new_tr");
      new_tr.my_copy(tr);
      `uvm_info("my_model", "get one transaction, copy and print it:", UVM_LOW)
      new_tr.my_print();
      ap.write(new_tr);
   end
endtask
`endif
```

此reference model就是单纯使用transaction中的my_copy函数，复制一份从i_agt得到的transaction，再传递到后级的scoreboard

reference和其他组件一样，再env中被实例化

**2. 目前的树结构**

![image-20221022000741471](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221022000741471.png)

**3. transaction的传输**

> 目前传输：
>
> i_agt -> ref -> scoreboard
>
> i_agt -> dut（目前还没提到DUT模块在哪）

> <mark>TLM(Transaction Level Modeling)</mark>
>
> UVM一般使用<mark>TLM(Transaction Level Modeling)</mark>实现component之间的transaction通信
>
> TLM有多种实现方式：
>
> + 发送的其中一种方法使用`uvm_analysis_port`
> + 接收的其中一种方法使用`uvm_blocking_get_port`
> + 连接：在此基础上还需要再env上定义一个`uvm_tlm_analysis_fifo`（一个fifo）将二者连在一起

> `uvm_analysis_port`是TLM发送的实现方式的一种
>
> + 定义：`uvm_analysis_port #(my_transaction) ap;`
> + 参数化的类，参数为需要传递数据的类型（本节中为my_transaction）
> + 通过调用内建函数`write`完成发送
> + <mark>非阻塞的</mark>

> `uvm_blocking_get_port`是TLM接收的实现方式的一种
>
> + 定义：`uvm_blocking_get_port #(my_transaction)  port;`
> + 参数化的类，参数为需要传递数据的类型（本节中为my_transaction）
> + 通过调用内建函数`get`完成接收

> `uvm_tlm_analysis_fifo`是TLM连接的实现方式的一种
>
> + 定义：`uvm_tlm_analysis_fifo #(my_transaction) agt_mdl_fifo;`
> + 参数化的类，参数为需要传递数据的类型（本节中为my_transaction）
>
> + 具体连接方法：
>   + 通过在<mark>**connect_phase**</mark>中，分别层次化调用monitor中的`uvm_analysis_port`与model中的`uvm_blocking_get_port`两种port的内建函数`connect`完成发送与接收的连接
>   + 连接时，需要调用`uvm_tlm_analysis_fifo`中用于指示端口类型的成员变量：`analysis_export`与`blocking_get_export`
>
> 为什么需要一个fifo？为啥不直接把monitor中的analysis_port和model中的blocking_get_port连接？
>
> + analysis_port是非阻塞性质的，ap.write函数调用完成后马上返回，不会等待数据被接收。假如当write函数调用时， blocking_get_port正在忙于其他事情，而没有准备好接收新的数据时，此时被write函数写入的my_transaction就需要一个暂存的位置，这就是fifo

> <mark>**connect_phase**</mark>
>
> + 在build_phase之后马上执行
> + <mark>与build_phase的同级执行顺序不同，connect_phase从树叶到叶根</mark>（从小到大）
>   + 即：先执行driver和monitor的connect_phase，再执行agent的connect_phase，最后执行env的connect_phase
>   + <mark>这样做有利于在实例化组件之后，进行层次化连接</mark>（见本节总结）

**4. 使用`uvm_analysis_port`发送**

my_monitor中：

+ 定义一个`uvm_analysis_port`类型的port
+ 在build_phase中实例化
+ 当准备好一个transaction后，在main_phase中写入该port
  + 写入port通过调用它的内建函数`write`实现

如下：

```verilog
`ifndef MY_MONITOR__SV
`define MY_MONITOR__SV
class my_monitor extends uvm_monitor;
   ...
   uvm_analysis_port #(my_transaction)  ap;
   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      if(!uvm_config_db#(virtual my_if)::get(this, "", "vif", vif))
         `uvm_fatal("my_monitor", "virtual interface must be set for vif!!!")
      ap = new("ap", this);
   endfunction
   
   extern task main_phase(uvm_phase phase);
   extern task collect_one_pkt(my_transaction tr);
       
endclass
```

```verilog
task my_monitor::main_phase(uvm_phase phase);
   my_transaction tr;
   while(1) begin
      tr = new("tr");
      collect_one_pkt(tr);
      ap.write(tr);
   end
endtask
```

**5. 使用`uvm_blocking_get_port`接收**

在model中：

+ 定义一个`uvm_blocking_get_port`类型的port
+ 在build_phase中实例化
+ 不断读取i_agt从此port发送来的transaction
  + 从port读取通过调用它的内建函数`get`实现

如下：

```verilog
class my_model extends uvm_component;
   
   uvm_blocking_get_port #(my_transaction)  port;
   uvm_analysis_port #(my_transaction)  ap;

   extern function new(string name, uvm_component parent);
   extern function void build_phase(uvm_phase phase);
   extern virtual  task main_phase(uvm_phase phase);

   `uvm_component_utils(my_model)
endclass 
```

```verilog
function my_model::new(string name, uvm_component parent);
   super.new(name, parent);
endfunction 

function void my_model::build_phase(uvm_phase phase);
   super.build_phase(phase);
   port = new("port", this);
   ap = new("ap", this);
endfunction

task my_model::main_phase(uvm_phase phase);
   my_transaction tr;
   my_transaction new_tr;
   super.main_phase(phase);
   while(1) begin
      port.get(tr);
      new_tr = new("new_tr");
      new_tr.my_copy(tr);
      `uvm_info("my_model", "get one transaction, copy and print it:", UVM_LOW)
      new_tr.my_print();
      ap.write(new_tr);
   end
endtask
```

**6. 使用`uvm_tlm_analysis_fifo`连接**

在env中：

+ 定义一个`uvm_tlm_analysis_fifo`类型的fifo，一个
+ 在build_phase中实例化`uvm_tlm_analysis_fifo`
  + <mark>注意：无需对env中的ap和port例化</mark>，他俩在monitor和model中已经被例化过了，这里只是做调用
+ 在connect_phase中进行连接
  + 发送这么连接到fifo：` i_agt.ap.connect(agt_mdl_fifo.analysis_export);`
  + 接收这么连接到fifo：` model.port.connect(agt_mdl_fifo.blocking_get_export);`
    + `agt_mdl_fifo`：一个`uvm_tlm_analysis_fifo`

如下：

```verilog
class my_env extends uvm_env;

   my_agent  i_agt;
   my_agent  o_agt;
   my_model  mdl;
   
   uvm_tlm_analysis_fifo #(my_transaction) agt_mdl_fifo;
   
   function new(string name = "my_env", uvm_component parent);
      super.new(name, parent);
   endfunction

   virtual function void build_phase(uvm_phase phase);
      super.build_phase(phase);
      i_agt = my_agent::type_id::create("i_agt", this);
      o_agt = my_agent::type_id::create("o_agt", this);
      i_agt.is_active = UVM_ACTIVE;
      o_agt.is_active = UVM_PASSIVE;
      mdl = my_model::type_id::create("mdl", this);
      agt_mdl_fifo = new("agt_mdl_fifo", this);
   endfunction

   extern virtual function void connect_phase(uvm_phase phase);
   
   `uvm_component_utils(my_env)
endclass
```

```verilog
function void my_env::connect_phase(uvm_phase phase);
   super.connect_phase(phase);
   i_agt.ap.connect(agt_mdl_fifo.analysis_export);
   mdl.port.connect(agt_mdl_fifo.blocking_get_export);
endfunction
```

**6. 总结一下**

+ 主要目的：model需要得到从i_agt来的transaction
+ 方式：
  + i_agt发送
  + model接收
  + env建立fifo并连接

+ 实例化位置：
  + i_monitor发送，因此`uvm_analysis_port`在i_monitor中实例化
  + i_agt实例化i_monitor，为了方便引用，因此在这里定义了一个指向i_monitor.ap的指针
  + model中接收，因此`uvm_blocking_get_port`在model中实例化
  + env中实例化一个`uvm_tlm_analysis_fifo`，并实例化各种组件，通过层次化引用到ap和port
+ 重要执行顺序：
  + env在build_phase中实例化agent
  + agent在build_phase中实例化i_monitor
  + agent在connect_phase中引用i_monitor.ap
  + env在conncet_phase中引用i_monitor.ap和model.port，并进行连接

> <mark>为什么是i_monitor发送？</mark>
>
> + 因为driver是给到dut的，基于信号级；model是基于transaction，因此需要i_monitor转换后的

#### 2.3.6 加入scoreboard

用于比较dut与ref的输出

---

**1.scoreboard定义**

```verilog
class my_scoreboard extends uvm_scoreboard;
   my_transaction  expect_queue[$];
   uvm_blocking_get_port #(my_transaction)  exp_port;
   uvm_blocking_get_port #(my_transaction)  act_port;
   `uvm_component_utils(my_scoreboard)

   extern function new(string name, uvm_component parent = null);
   extern virtual function void build_phase(uvm_phase phase);
   extern virtual task main_phase(uvm_phase phase);
endclass 

function my_scoreboard::new(string name, uvm_component parent = null);
   super.new(name, parent);
endfunction 

function void my_scoreboard::build_phase(uvm_phase phase);
   super.build_phase(phase);
   exp_port = new("exp_port", this);
   act_port = new("act_port", this);
endfunction 

task my_scoreboard::main_phase(uvm_phase phase);
   my_transaction  get_expect,  get_actual, tmp_tran;
   bit result;
 
   super.main_phase(phase);
   fork 
      while (1) begin
         exp_port.get(get_expect);
         expect_queue.push_back(get_expect);
      end
      while (1) begin
         act_port.get(get_actual);
         if(expect_queue.size() > 0) begin
            tmp_tran = expect_queue.pop_front();
            result = get_actual.my_compare(tmp_tran);
            if(result) begin 
               `uvm_info("my_scoreboard", "Compare SUCCESSFULLY", UVM_LOW);
            end
            else begin
               `uvm_error("my_scoreboard", "Compare FAILED");
               $display("the expect pkt is");
               tmp_tran.my_print();
               $display("the actual pkt is");
               get_actual.my_print();
            end
         end
         else begin
            `uvm_error("my_scoreboard", "Received from DUT, while Expect Queue is empty");
            $display("the unexpected pkt is");
            get_actual.my_print();
         end 
      end
   join
endtask
```

> 流程：
>
> + 建立两个进程
> + 进程1：接收`exp_port`(ref)一个trans到队列expect_queue中
> + 进程2：
>   + 从`act_port`(dut)接收一个trans为get_actual
>   + expect_queue弹出一个trans到tmp_tran
>   + 调用get_actual.compare函数

> 数据：
>
> scoreboard比较的数据：
>
> + 来自reference model的port，通过端口exp_port获取
> + 来自monitor的o_agt，通过端口act_port获取
>   + 你可能好奇为什么DUT是通过monitor的o_agt获取到scoreboard的？你个笨蛋！DUT是直接和o_agt连接的，把数据打包为事务级（driver相反，把事务级转为端口级）
>
> 端口与对应组件(2.3.5)
>
> + scoreboard：act_port, exp_port
> + monitor：o_agt
> + reference model：port

**2.实例化scoreboard**

在my_env中实例化scoreboard

![image-20221119221905011](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221119221905011.png)

#### 2.3.7 加入<mark>field_automation</mark>机制

用于在transaction中定义变量，从而可以用到uvm的transaction函数

---

**1.my_transaction定义修改**

```verilog
class my_transaction extends uvm_sequence_item;

   rand bit[47:0] dmac;
   rand bit[47:0] smac;
   rand bit[15:0] ether_type;
   rand byte      pload[];
   rand bit[31:0] crc;

   constraint pload_cons{
      pload.size >= 46;
      pload.size <= 1500;
   }

   function bit[31:0] calc_crc();
      return 32'h0;
   endfunction

   function void post_randomize();
      crc = calc_crc;
   endfunction

   `uvm_object_utils_begin(my_transaction)
      `uvm_field_int(dmac, UVM_ALL_ON)
      `uvm_field_int(smac, UVM_ALL_ON)
      `uvm_field_int(ether_type, UVM_ALL_ON)
      `uvm_field_array_int(pload, UVM_ALL_ON)
      `uvm_field_int(crc, UVM_ALL_ON)
   `uvm_object_utils_end

   function new(string name = "my_transaction");
      super.new();
   endfunction

endclass
```

> <mark>file_automation机制</mark>>
>
> 流程：
>
> + ```uvm_object_utils_begin``` 与 ````uvm_object_utils_end```划定file_automation定义的范围
> + 针对不同数据类型调用不同宏进行变量定义
>   + `uvm_field_int`
>   + `uvm_field_array_int`
>
> 经过以上步骤，可以直接调用定义了这些变量的类的方法：
>
> + `copy`
> + `compare`
> + `print`
> + `pack_bytes`
> + `unpack_bytes`

**2.my_model与scoreboard中直接调用file_automation带来的方法**

```verilog
task my_model::main_phase(uvm_phase phase);
   my_transaction tr;
   my_transaction new_tr;
   super.main_phase(phase);
   while(1) begin
      port.get(tr);
      new_tr = new("new_tr");
      new_tr.copy(tr);//直接调用
      `uvm_info("my_model", "get one transaction, copy and print it:", UVM_LOW)
       new_tr.print();//直接调用
      ap.write(new_tr);
   end
endtask
```

```verilog
task my_scoreboard::main_phase(uvm_phase phase);
   my_transaction  get_expect,  get_actual, tmp_tran;
   bit result;
 
   super.main_phase(phase);
   fork 
      while (1) begin
         exp_port.get(get_expect);
         expect_queue.push_back(get_expect);
      end
      while (1) begin
         act_port.get(get_actual);
         if(expect_queue.size() > 0) begin
            tmp_tran = expect_queue.pop_front();
            result = get_actual.compare(tmp_tran);//直接调用
            if(result) begin 
               `uvm_info("my_scoreboard", "Compare SUCCESSFULLY", UVM_LOW);
            end
            else begin
               `uvm_error("my_scoreboard", "Compare FAILED");
               $display("the expect pkt is");
               tmp_tran.print();
               $display("the actual pkt is");
               get_actual.print();
            end
         end
         else begin
            `uvm_error("my_scoreboard", "Received from DUT, while Expect Queue is empty");
            $display("the unexpected pkt is");
            get_actual.print();
         end 
      end
   join
endtask
```



**3.引入file_automation的好处/driver与monitor的定义简化**

好处：引入field_automation机制的另外一大好处是简化了driver和monitor。在2.3.1节及2.3.3节中，my_driver的drv_one_pkt任务和 my_monitor的collect_one_pkt任务代码很长，但是几乎都是一些重复性的代码

使用field_automation机制后，drv_one_pkt任务可以 简化为：

```verilog
task my_driver::drive_one_pkt(my_transaction tr);
   byte unsigned     data_q[];
   int  data_size;
   
   data_size = tr.pack_bytes(data_q) / 8; //直接调用pack_bytes方法，把tr中字段打包为字节流放入data_q中，简化了！
   `uvm_info("my_driver", "begin to drive one pkt", UVM_LOW);
   repeat(3) @(posedge vif.clk);
   for ( int i = 0; i < data_size; i++ ) begin
      @(posedge vif.clk);
      vif.valid <= 1'b1;
      vif.data <= data_q[i]; 
   end

   @(posedge vif.clk);
   vif.valid <= 1'b0;
   `uvm_info("my_driver", "end drive one pkt", UVM_LOW);
endtask
```

> pack流程：
>
> + 直接调用`pack_bytes`

> 对比：
>
> + pack_bytes将tr中所有的字段变成byte流放入data_q中
> + 在2.3.1节中是手工地将所有字段放入data_q中的。 pack_bytes极大地减少了代码量

my_monitor的collect_one_pkt可以简化成：

```verilog
task my_monitor::collect_one_pkt(my_transaction tr);
   byte unsigned data_q[$];
   byte unsigned data_array[];
   logic [7:0] data;
   logic valid = 0;
   int data_size;
   
   while(1) begin
      @(posedge vif.clk);
      if(vif.valid) break;
   end
   
   `uvm_info("my_monitor", "begin to collect one pkt", UVM_LOW);
   while(vif.valid) begin
      data_q.push_back(vif.data);
      @(posedge vif.clk);
   end
   data_size  = data_q.size();   
   data_array = new[data_size];
   for ( int i = 0; i < data_size; i++ ) begin
      data_array[i] = data_q[i]; 
   end
    tr.pload = new[data_size - 18]; //da sa, e_type, crc
    data_size = tr.unpack_bytes(data_array) / 8; //直接调用unpack_bytes方法，把data_q中的bytes流解包为tr各个字段，简化了！
   `uvm_info("my_monitor", "end collect one pkt", UVM_LOW);
endtask
```

> unpack流程，括号内表示解释：
>
> + 定义一个接收data_q的动态数组，用作`unpack_bytes`函数参数（`unpack_bytes`函数的输入参数必须是一个动态数组，所以需要先把收集到的、放在data_q中的数据复制到一个动态数组中，这里使用到的是data_array）
> + 由于tr在定义字段的时候使用到了一个动态数组字段，这里的需要指定大小后才能接收到tr的字段里，*讲道理我感觉可以在main_phase就指定了*（由于在tr中的pload是一个动态数组，所以需要在调用`unpack_bytes`之前指定其大小，这样unpack_bytes函数才能正常工作）
> + 调用`unpack_bytes`

> **打包成的数据流顺序：**
>
> 在把所有的字段变成byte流放入data_q中时，字段按照uvm_field系列宏书写的顺序排列
>
> 在上述代码中是先放入dmac，再依次放入smac、ether_type、pload、crc

### 2.4 UVM的终极大作：<mark>sequence机制</mark>

#### 2.4.1 在验证平台中加入sequencer

功能：<mark>sequence机制</mark>用于产生激励

区别：**前面的例子中激励都是在driver中产生的，但是在一个规范化的UVM验证平台中，driver只负责驱动transaction，不负责生产transaction**

sequence机制两大组成部分：

+ sequence
+ sequencer

---

**1.定义一个sequencer**

```verilog
class my_sequencer extends uvm_sequencer #(my_transaction);
   
   function new(string name, uvm_component parent);
      super.new(name, parent);
   endfunction 
   
   `uvm_component_utils(my_sequencer)
endclass
```

> 流程：
>
> + `uvm_sequencer`是一个参数化的类，参数为自定义的transaction
> + 派生自`uvm_sequencer`
> + 使用``uvm_component_utils`进行注册

> 与其他组件的关系：
>
> + sequencer产生transaction
> + driver接收transaction

**2.有关派生自参数化的类**

> my_driver定义修正
>
> 之前定义的my_driver时都是直接从`uvm_driver`派生的，即`class my_driver extends uvm_driver;`，这是种**不常见**的写法
>
> 因为`uvm_driver`也是一个参数化的类，应该在定义时指明driver要驱动的transaction类型，这么写是**正确**的`class my_driver extends uvm_driver#(my_transaction);`

> 使用参数的好处
>
> 可以直接使用`uvm_driver`中的某些预先定义好的成员变量
>
> 如`uvm_driver`中有成员变量req，它的类型就是传递给uvm_driver的参数，在这里就是my_transaction，可以直接使用req

参数定义后的my_driver中，带来的好处（req直接使用），注：这里依然是在driver中产生激励，下一节从driver中移除

```verilog
task my_driver::main_phase(uvm_phase phase);
   phase.raise_objection(this);
   vif.data <= 8'b0;
   vif.valid <= 1'b0;
   while(!vif.rst_n)
      @(posedge vif.clk);
   for(int i = 0; i < 2; i++) begin 
      req = new("req");
      assert(req.randomize() with {pload.size == 200;});
       drive_one_pkt(req);//直接使用
   end
   repeat(5) @(posedge vif.clk);
   phase.drop_objection(this);
endtask
```

**3.把sequencer放入到agent中**

在完成sequencer的定义后，由于sequencer与driver的关系非常密切，因此要把其加入agent中

```verilog
class my_agent extends uvm_agent ;
   my_sequencer  sqr;
   my_driver     drv;
   my_monitor    mon;
   
   uvm_analysis_port #(my_transaction)  ap;
   
   function new(string name, uvm_component parent);
      super.new(name, parent);
   endfunction 
   
   extern virtual function void build_phase(uvm_phase phase);
   extern virtual function void connect_phase(uvm_phase phase);

   `uvm_component_utils(my_agent)
endclass 


function void my_agent::build_phase(uvm_phase phase);
   super.build_phase(phase);
   if (is_active == UVM_ACTIVE) begin
       sqr = my_sequencer::type_id::create("sqr", this);//创建sequencer
      drv = my_driver::type_id::create("drv", this);
   end
   mon = my_monitor::type_id::create("mon", this);
endfunction 

function void my_agent::connect_phase(uvm_phase phase);
   super.connect_phase(phase);
   ap = mon.ap;
endfunction

```

**4.加入sequencer后的UVM树结构图**

![image-20221121200506347](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221121200506347.png)

#### 2.4.2 sequence机制

<mark>sequence不属于验证平台的任何一部分</mark>，但是它与sequencer之间有密切的联系

![带sequnce的UVM验证平台](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221121200630176.png)

> sequencer与sequence的关系
>
> + 只有在 sequencer的帮助下，sequence产生出的transaction才能最终送给driver
> + sequencer只有在sequence出现的情况下才能体现其价值
>
> 一个奇怪的比喻
>
> + sequence就像是一个弹夹，里面的子弹是transaction
> + 而sequencer是一把枪
> + 弹夹只有放入枪中才有意义，枪只有在放入弹夹后才能发挥威力
>
> sequencer与sequence的不同
>
> + sequencer是一个`uvm_component`
> + sequence是一个`uvm_object`，与transaction一样具有生命周期，比my_transaction生命周期要长一些，其内的transaction全部 发送完毕后，它的生命周期也就结束了
> + sequence使用``uvm_object_utils`宏注册到factory中

**1.一个sequence的定义**

```verilog
class my_sequence extends uvm_sequence #(my_transaction);
   my_transaction m_trans;

   function new(string name= "my_sequence");
      super.new(name);
   endfunction

   virtual task body();
      repeat (10) begin
         `uvm_do(m_trans)
      end
      #1000;
   endtask

   `uvm_object_utils(my_sequence)
endclass

```

> 定义流程：
>
> + 派生自`uvm_sequence`，参数为transaction类型
> + 定义`body`，每一个sequence都有一个`body`任务，当一个sequence启动之后，会自动执行body中的代码

> ``uvm_do`
>
> **这是UVM中最常用的宏之一**，作用：
>
> + 将一个my_transaction的变量m_trans实例化
> + 将m_trans随机化
> + 将m_trans**送给**sequencer
>
> 如果不用``uvm_do`宏，也可以直接使用`start_item`与`finish_item`的方式产生transaction
>
> 什么时候返回：等待driver的`item_done`

**2.sequence与sequencer的交互**

交互：

+ 一个sequence在向sequencer发送transaction前，要先向sequencer发送一个请求
+ sequencer把这个请求放在一个仲裁队列中

sequencer的具体工作：

1. 检测仲裁队列里是否有某个sequence发送transaction的请求
2. 检测driver是否申请transaction

sequencer检测细节：

1. 如果仲裁队列里有发送请求，但是driver没有申请trans：一直等待driver
2. 如果仲裁队列中没有发送请求，但是driver向sequencer申请新的trans：sqr进入等待seq状态
3. 如果仲裁队列中有发送请求，同时driver也在向sequencer申请新的trans：直接同意并移交

**3.driver如何向sequencer申请transaction**

1. 在agent的`connect`函数中，把drv中的变量seq_item_port与sqr中的seq_itemu_export连接，代码如下：

```verilog
function void my_agent::connect_phase(uvm_phase phase);
   super.connect_phase(phase);
   if (is_active == UVM_ACTIVE) begin
      drv.seq_item_port.connect(sqr.seq_item_export);
   end
   ap = mon.ap;
endfunction
```

2. 连接好之后，就可以在driver中通过`get_next_item`或`try_next_item`任务向sequencer申请新的trans，代码如下：

使用`get_next_item`

```verilog
task my_driver::main_phase(uvm_phase phase);
   vif.data <= 8'b0;
   vif.valid <= 1'b0;
   while(!vif.rst_n)
      @(posedge vif.clk);
   while(1) begin
      seq_item_port.get_next_item(req);//向sequencer申请新的transaction
      drive_one_pkt(req);
      seq_item_port.item_done();
   end
endtask
```

使用`try_next_item`

```verilog
task my_driver::main_phase(uvm_phase phase);
   vif.data <= 8'b0;
   vif.valid <= 1'b0;
   while(!vif.rst_n)
      @(posedge vif.clk);
   while(1) begin
      seq_item_port.try_next_item(req);//向sequencer申请新的transaction
      if(req == null)
       	@(posedge vif.clk)
      else begin
        drive_one_pkt(req);
        seq_item_port.item_done();
      end
   end
endtask
```

> my_driver代码解析：
>
> + 使用while(1)，因为需要不断驱动
> + 通过`get_next_item`任务得到一个trans，或者`try_next_item`
> + 通过`item_done`任务告知sequencer

> 为什么会有一个`item_done`？
>
> + 一种握手机制
> + 可以用来返回response（6.7.1）
> + sqr内部保存的同一trans会不断发送，直到调用了`item_done`

> 相比于`get_next_item`，`try_next_item`的行为更加接近真实driver的行为：当有数据时，就驱动数据，否则总线将一直处于空闲状 态

**4.启动seq（在哪里实例化seq）**

启动可以在不同组件的`main_phase`中（sqr/env)，但在实际应用中， 使用最多的还是通过`default_sequence`的方式启动sequence见2.4.3节

在my_env中启动：

```verilog
task my_env::main_phase(uvm_phase phase);
   my_sequence seq;
   phase.raise_objection(this);
   seq = my_sequence::type_id::create("seq");
   seq.start(i_agt.sqr); 
   phase.drop_objection(this);
endtask
```

在my_sqr中启动

```verilog
task my_sequencer::main_phase(uvm_phase phase);
   my_sequence seq;
   phase.raise_objection(this);
   seq = my_sequence::type_id::create("seq");
   seq.start(this); 
   phase.drop_objection(this);
endtask
```

> 流程：
>
> + 声明一个seq变量
> + seq工厂机制实例化
> + 调用seq的`start`方法，入参为sqr指针（如果不指明则不知道匹配的sqr是谁）

> <mark>objection机制与结束仿真</mark>
>
> 在UVM中，objection一般伴随着sequence，通常只在sequence出现的地方才提起和撤销 objection。如前面所说，sequence是弹夹，当弹夹里面的子弹用光之后，可以结束仿真了

> sqr与env中启动的不同：唯一区别是seq.start的参数变为了this

#### 2.4.3 default_sequence的使用

sequence是在my_env的`main_phase`中手工启动的，作为示例使用这种方式足够了，**但在实际应用中， 使用最多的是通过`default_sequence`的方式启动sequence**

---

**1.如何在default_sequence中启动seq**

使用default_sequence的方式非常简单，只需要在某个component（如my_env）的`build_phase`中进行一行设置

在my_env中启动default_sequence的代码：

```verilog
// my_env.sv
virtual function void build_phase(uvm_phase phase);
	  ...
      uvm_config_db#(uvm_object_wrapper)::set(this,
                                              "i_agt.sqr.main_phase",
                                              "default_sequence",
                                              my_sequence::type_id::get());//defualt_sequence!

endfunction

```

> `config_db`的使用：这是除了在top_tb中通过`config_db`设置virtual interface后再一次用到`config_db`
>
> **参数一：**
>
> + 与在top_tb中不同的是，这里set函数的第 一个参数由null变成了this，而第二个代表路径的参数则去除了`uvm_test_top`
> + 事实上，第二个参数是相对于第一个参数的相对路径，由于上述代码是在my_env中，而my_env本身已经是uvm_test_top了，且第一个参数被设置为了this，所以第二个参数中就不需要uvm_test_top
> + 在top_tb中设置virtual interface时，由于top_tb不是一个类，无法使用this指针，所以设置set的第一个参数为null，第二个参数使用绝对路径uvm_test_top.xxx
>
> **参数二：**在第二个路径参数中，出现了`main_phase`。这是UVM在设置default_sequence时的要求。由于除了`main_phase`外，还存在其他任务phase，如`configure_phase`、`reset_phase`等，**所以必须指定是哪个phase**，从而使sequencer知道在哪个phase启动这个sequence
>
> **参数三、参数四、参数类型：**至于set的第三个和第四个参数，以及`uvm_config_db#(uvm_object_wrapper)`中为什么是`uvm_object_wrapper`而不是 `uvm_sequence`或者其他，则纯粹是由于UVM的规定，用户在使用时照做即可
>
> > 为什么这里只用设置一次`config_db`
> >
> > config_db通常都是成对出现的，如：在top_tb中通过set设置virtual interface，而在driver或者monitor中通过get函数得到virtual interface
> >
> > 那么在这里是否需要在sequencer中手工写一些get相关的代码呢？答案是否定的。UVM已经做好了这些，读者无需再把时间花在这上面

也可以在top_tb中启动default_sequence：

```verilog
module top_tb;
    ...
    initial begin
        uvm_config_db# （uvm_object_wrapper)::set(null,
                                                "uvm_test_top.i_agt.sqr.main_phase",
                                                "default_sequence",
                                                 my_sequence::type_id::get());
    end
endmodule
```

> 形参设置：
>
> 第一个参数和第二个参数应该改变一下

也可以在其他的组件内，如my_agent的`build_phase`

```verilog
function void my_agent::build_phase(uvm_phase phase);
    super.build_phase(phase);
    ...
    uvm_config_db# (uvm_object_wrapper)::set(this,
                                            "sqr.main_phase",
                                            "default_sequence",
                                             my_sequence::type_id::get());
endfunction
```

> 形参设置：
>
> 只需要正确地设置set的第二个参数即可（相对于my_env的）

**3.default_sequence如何提起和撤销objection**

在上一节<u>手动启动sequence**前后**，分别提起和撤销objection</u>，此时使用default_sequence自动启动，又如何提起和撤销objection呢？

**方法：**使用继承自uvm_sequence中的变量`starting_phase`，在sequence中使用`starting_phase`进行提起和撤销objection

**sequence定义代码如下：**

```verilog
class my_sequence extends uvm_sequence #(my_transaction);
   my_transaction m_trans;

   function new(string name= "my_sequence");
      super.new(name);
   endfunction

   virtual task body();
      if(starting_phase != null) 
         starting_phase.raise_objection(this);
      repeat (10) begin
         `uvm_do(m_trans)
      end
      #1000;
      if(starting_phase != null) 
         starting_phase.drop_objection(this);
   endtask

   `uvm_object_utils(my_sequence)
endclass
```

**sequencer在启动default_sequence时会自动做如下相关操作：**

```verilog
task my_sequencer::main_phase(uvm_phase phase);
…
  seq.starting_phase = phase;
  seq.start(this);
…
endtask
```

<mark>UVM1.2貌似优化了starting_phase功能，使用方式有所改变</mark>

### 2.5 建造测试用例

#### 2.5.1 加入`base_test`

**1.前情提要**

UVM使用的是一种树形结构，在本书的例子中：

+ 最初这棵树的树根是`my_driver`
+ 由于要放置其他component，树根变成 了`my_env`
+ 但是在一个实际应用的UVM验证平台中，my_env并不是树根，通常来说，树根是一个基于`uvm_test`派生的类

本节先讲述`base_test`，真正的测试用例都是基于`base_test`派生的一个类

---

**1.定义base_test**

```verilog
class base_test extends uvm_test;

   my_env         env;
   
   function new(string name = "base_test", uvm_component parent = null);
      super.new(name,parent);
   endfunction
   
   extern virtual function void build_phase(uvm_phase phase);
   extern virtual function void report_phase(uvm_phase phase);
   `uvm_component_utils(base_test)
endclass


function void base_test::build_phase(uvm_phase phase);
   super.build_phase(phase);
   env  =  my_env::type_id::create("env", this); 
   uvm_config_db#(uvm_object_wrapper)::set(this,
                                           "env.i_agt.sqr.main_phase",
                                           "default_sequence",
                                            my_sequence::type_id::get());
endfunction

function void base_test::report_phase(uvm_phase phase);
   uvm_report_server server;
   int err_num;
   super.report_phase(phase);

   server = get_report_server();
   err_num = server.get_severity_count(UVM_ERROR);

   if (err_num != 0) begin
      $display("TEST CASE FAILED");
   end
   else begin
      $display("TEST CASE PASSED");
   end
endfunction
```

> 定义流程：
>
> + 派生自`uvm_test`	
> + 使用``uvm_component_utils`宏注册到工厂中
> + 在`build_phase`中实例化my_env
> + 在`build_phase`中设置`default_sequence`，**以后都在这里设置default_sequence**

> base_test一般做什么
>
> + base_test中做的事情在根据不同的验证平台及不同的公司而不同，没有统一的答案
> + 此例用到了<mark>report_phase</mark>，用于根据`UVM_ERROR`数量打印不同信息，除此之外一些工具可以根据打印信息判断DUT是否通过了某个测试用例的检查
> + 设置整个验证平台的超时退出时间
> + 通过config_db设置验 证平台中某些参数的值

><mark>report_phase</mark>
>
>report_phase也是UVM内建的一个phase，它在main_phase结束之后执行

**2.加入base_test后的UVM树**

![image-20221122191951186](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221122191951186.png)

**3.top_tb模块修改run_test的env为base_test**

```verilog
module top_tb;

...
    
initial begin
   run_test("base_test");
end

initial begin
   uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.env.i_agt.drv", "vif", input_if);
   uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.env.i_agt.mon", "vif", input_if);
   uvm_config_db#(virtual my_if)::set(null, "uvm_test_top.env.o_agt.mon", "vif", output_if);
end

endmodule
```

#### 2.5.2 UVM中测试用例的启动

测试用例=测试向量=**pattern**

**1.如何启动两个（多个也行）不同的测试用例**

不同测试用例需要修改default_sequence，从而需要多个base_test

**方法1：修改run_test参数**

+ 启动my_case0，需要在top_tb中更改run_test的参数：

```verilog
initial begin 
    run_test("my_case0");
end
```

+ 启动my_case1，需要在top_tb中更改run_test的参数：

```verilog
initial begin 
    run_test("my_case1");
end
```

> 需要不断编译！
>
> 当my_case0运行的时候需要修改代码，重新编译后才能运行；当my_case1运行时也需如此，这相当不方便

**方法2：使用命令行**

**UVM提供对不加参数的run_test的支持**

+ 直接run_test

```verilog
initial begin 
    run_test();
end
```

> 在这种情况下，UVM会利用`UVM_TEST_NAME`从命令行中寻找测试用例的名字，创建它的实例并运行

+ 命令行运行时需要添加参数

```bash
...+UVM_TEST_NAME=my_case0
//或者
...+UVM_TEST_NAME=my_case1
```

**2.测试用例流程启动与验证平台执行流程总结**

![image-20221122194654543](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221122194654543.png)

启动后UVM树的结构如图

![启动后UVM树的结构如图](https://raw.githubusercontent.com/GreensCH/blog-drawbed/main/common/image-20221122194711463.png)

> UVM树与上一节的不同：
>
> 把base_test替换为my_casen（说明有好几个测试用例）

**3.测试用例定义细节**

my_case0与case0_sequence：

```verilog
class case0_sequence extends uvm_sequence #(my_transaction);
   my_transaction m_trans;

   function  new(string name= "case0_sequence");
      super.new(name);
   endfunction 
   
   virtual task body();
      if(starting_phase != null) 
         starting_phase.raise_objection(this);
      repeat (10) begin
         `uvm_do(m_trans)
      end
      #100;
      if(starting_phase != null) 
         starting_phase.drop_objection(this);
   endtask

   `uvm_object_utils(case0_sequence)
endclass


class my_case0 extends base_test;

   function new(string name = "my_case0", uvm_component parent = null);
      super.new(name,parent);
   endfunction 
   extern virtual function void build_phase(uvm_phase phase); 
   `uvm_component_utils(my_case0)
endclass


function void my_case0::build_phase(uvm_phase phase);
   super.build_phase(phase);

   uvm_config_db#(uvm_object_wrapper)::set(this, 
                                           "env.i_agt.sqr.main_phase", 
                                           "default_sequence", 
                                           case0_sequence::type_id::get());//default_sequencer
endfunction
```

my_case1与case1_sequence：

```verilog
class case1_sequence extends uvm_sequence #(my_transaction);
   my_transaction m_trans;

   function  new(string name= "case1_sequence");
      super.new(name);
   endfunction 

   virtual task body();
      if(starting_phase != null) 
         starting_phase.raise_objection(this);
      repeat (10) begin
         `uvm_do_with(m_trans, { m_trans.pload.size() == 60;})
      end
      #100;
      if(starting_phase != null) 
         starting_phase.drop_objection(this);
   endtask

   `uvm_object_utils(case1_sequence)
endclass

class my_case1 extends base_test;
  
   function new(string name = "my_case1", uvm_component parent = null);
      super.new(name,parent);
   endfunction 
   
   extern virtual function void build_phase(uvm_phase phase); 
   `uvm_component_utils(my_case1)
endclass


function void my_case1::build_phase(uvm_phase phase);
   super.build_phase(phase);

   uvm_config_db#(uvm_object_wrapper)::set(this, 
                                           "env.i_agt.sqr.main_phase", 
                                           "default_sequence", 
                                           case1_sequence::type_id::get());//default_sequencer
endfunction
```

> ``uvm_do_with`宏
>
> 是``uvm_do`系列宏中的一个，用于在随机化时提供对某些字段的**约束**
