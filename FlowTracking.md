# Flow Tracker #

Flow tracker is a tool that detects [timing attack](http://en.wikipedia.org/wiki/Timing_attack) vulnerabilities in C/C++ programs that implement cryptographic routines. If the program contains a vulnerability, then Flow Tracker finds one or more (static) traces of instructions that uncover it. A vulnerable trace describes a sequence of data and control dependences from a secret information - e.g., a crypto key, the seed for a random number generator, etc - to a predicate of a branch. This is a problem because the branch normally influences the execution time of the program. So, if an adversary can measure this time for a given input which he controls, it may discover some information about the secret that we want to protect.


# Timming Attack Problem #

In cryptography, a timing attack is a side channel attack in which the attacker attempts to compromise a cryptosystem by analyzing the time taken to execute cryptographic algorithms. Every logical operation in a computer takes time to execute, and the time can differ based on the input; with precise measurements of the time for each operation, an attacker can work backwards to the input.

Information about the key or other sensitive information can leak from a system through measurement of the time it takes to respond to certain queries. How much such information can help an attacker depends on many variables: crypto system design, the CPU running the system, the algorithms used, assorted implementation details, timing attack countermeasures, the accuracy of the timing measurements, etc

# LLVM Install #

Flow Tracker has been implemented as an [LLVM](http://www.llvm.org#) 3.3 Compiler pass. To run it, we use opt, a tool that comes in the llvm distribution. Thus, to use our tool, you need to install llvm 3.3. To download LLVM 3.3 use the links below:

- [LLVM 3.3](http://www.llvm.org/releases/3.3/llvm-3.3.src.tar.gz#)

- [Clang](http://www.llvm.org/releases/3.3/cfe-3.3.src.tar.gz#)

Extract llvm-3.3.src.tar.gz in your home directory. You can rename that folder to llvm33 for instance.
Now, extract cfe-3.3.src.tar.gz in /YourHome/llvm33/tool/

Notice that if your system does not have an installation of g++, you will need to install it. (For Debian Linux, type as a root the following command... apt-get install g++)

Once you download llvm, you must type ./configure at /YourHome/llvm33/. After the configuration script finished its execution, you must type make -j 4

After compilation, include the path to the llvm binarie in your /YourHome/.bashrc. You can do this using the following line:
```
export PATH=$PATH:/YourHome/llvm33/Release+Asserts/bin
```

# Flow Tracker  Install #

Flow Tracker is available at [this](http://www.dcc.ufmg.br/~brunors/flowtracker/flowtracker.tar.gz#) link.

Extract the flowtracker.tar.gz to /YourHome/llvm33/lib/Transforms/

Make the following symlinks in your LLVM installation

```
ln -s include/llvm/IR/Function.h Function.h
ln -s include/llvm/IR/Instruction.h Instruction.h
ln -s include/llvm/IR/BasicBlock.h BasicBlock.h
ln -s include/llvm/IR/Module.h Module.h
ln -s include/llvm/IR/Instructions.h Instructions.h
ln -s include/llvm/IR/User.h User.h
ln -s include/llvm/IR/Metadata.h Metadata.h
ln -s include/llvm/IR/InstrTypes.h InstrTypes.h
ln -s include/llvm/IR/Operator.h Operator.h
ln -s include/llvm/IR/Constants.h Constants.h
ln -s include/llvm/IR/Constant.h Constant.h
ln -s lib/Transforms/PADriver/PADriver.h PADriver.h
ln -s lib/Transforms/PADriver/PointerAnalysis.h
```

Now, you must compile each of the passes that constitute Flow Tracker:

```
cd /YourHome/llvm33/lib/Transforms/PADriver
make
```

```
cd /YourHome/llvm33/lib/Transforms/AliasSets
make
```

```
cd /YourHome/llvm33/lib/Transforms/DepGraph
make
```


```
cd /YourHome/llvm33/lib/Transforms/hammock
make
```

```
cd /YourHome/llvm33/lib/Transforms/bSSA2
make
```

You have also to compile the xmlparser using the command below inside /YourHome/llvm33/lib/Transforms/bSSA2:

```
g++ -shared -o parserXML.so -fPIC parserXML.cpp tinyxml2.cpp
```

The command above will create a file named parserXML.so. Move this file to /YourHome/llvm33/Release+Asserts/lib.

# Flow Tracker Usage #

To demonstrate how to use Flow Tracker, we will find a vulnerability in this [program](http://www.dcc.ufmg.br/~brunors/flowtracker/monty.c#). This file is an implementation of the [montgomery reduction](LINK.md). It must run in constant time, otherwise it may disclosure parts of the crypto key that it uses internally. So, we want to know if the program runs always in the same time, independent on the cryptographic key or on the input that it uses. To do that, type the following command:

```
clang -emit-llvm -c -g monty.c -o monty.bc
```
```
opt -instnamer -mem2reg monty.bc > monty.rbc
```
```
opt -load PADriver.so -load AliasSets.so -load DepGraph.so -load hammock.so -load bSSA2.so -bssa2 -xmlfile in.xml monty.rbc
```

Moreover, you can join several .c files in a single .bc using clang + llvm-link just like example bellow:

```
clang -emit-llvm -c -g main.c -o main.bc
```

```
clang -emit-llvm -c -g foo.c -o foo.bc
```

```
llvm-link foo.bc main.bc -o out.bc
```


Please, see that you must inform a xml file with the functions and respective parameters which you consider sensitive or secrete information. Below you can see an example of such an xml file. In this case, Flow Tracker will locate the function named fp\_rdcn\_var and it will consider the first and the second parameters as sensitive information. If the user specify the parameter 0, FlowTracker will consider the function's return value as a secret.

```
<functions>
<sources>
<function>
<name>fp_rdcn_var</name>
<parameter>1</parameter>
<parameter>2</parameter>
</function>
</sources>
</functions>
```

If the target program contains a vulnerable trace, then Flow Tracker will create two files for each of those traces.

> - The first one is a .dot file which requires a dot visualization tool like [xdot](http://www.graphviz.org/category/tags/xdot#) to visualize it. This file shows the vulnerable subgraph of the entire dependence graph of the program analyzed. The nodes are operands and operators of the LLVM Intermediate Representation. The label used at each node informs the line and the filename where that node was observed by Flow Tracker. In case you find this file hard to understand, we also output an ASCII report, which we describe next.

> - The second output of Flow Tracker is an ASCII file which informs the start line in your code where the problem happens and the intermediate lines that propagate the secret information. It also shows the line that contains the function or branch that uses this information. For instance, Let's see the content of the respective ASCII file that Flow Tracker produces to our [(monty.c)](http://www.dcc.ufmg.br/~brunors/flowtracker/monty.c#) example]:

```
Tracking vulnerability for 'file monty.c line 298 to file monty.c line 181 '
monty.c 181
monty.c 181
monty.c 181
monty.c 246
monty.c 246
monty.c 246
monty.c 199
monty.c 199
monty.c 199
monty.c 199
monty.c 199
monty.c 199
```

As we can see, the monty.c has a vulnerable trace that starts at line 298 and finishes at line 181. The following lines shows in backward way the entire vulnerable trace. We do not show the last line in the trace, only in the header description. Line 199 is the line that comes immediately before the start line 298 in our trace.

# Virtual Machine #

If you do not want to install llvm and Flow Tracker, and you have a lot of network bandwidth available, you can download our Virtual Machine (Virtual Box) with Linux Ubuntu 14 64 bits and Flow Tracker.  You must install [Virtual Box](https://www.virtualbox.org/#) in your system and download our [virtual machine](https://www.dropbox.com/s/r3tvbawhalia55f/FlowTracker.zip?dl=0#).

The virtual machine user name is "user" and password is "user". At home directory you will see a folder named monty which contains a tiny example of the montgomery reduction. You may use it in order to test Flow Tracker. To do it, just type:

```
clang -emit-llvm -c -g monty.c -o monty.bc
```
```
opt -instnamer -mem2reg monty.bc > monty.rbc
```
```
opt -load PADriver.so -load AliasSets.so -load DepGraph.so -load hammock.so -load bSSA2.so -bssa2 -xmlfile in.xml monty.rbc
```


# Example's Gallery #

In this page we show some examples of programs, and the results that
out flow tracker analysis produces for them. For all examples, our secret is the variable pw. We use the following keys in the
table:

  * **C-Src**: the C program that we have analyzed. This is an actual program, i.e., the very input to our analyzer.
  * **Full Dep Graph**: the complete Dependence Graph with control edges and data edges. Each vertex is labeled with one operand or operator.
  * **Tainted Sub Graph**:the Dependence Graph which IS produced by our tool. This DFG has the respective line numbers from source file and is useful for developer to identify the vulnerability in the code. Note that the SUB-DFG is one example, however the tool can to produce more than one SUB-DFG if there exists several vulnerable paths. In red color is the shortest path which connects the secret to a branch instruction.
  * **TXT-Lines**: ASCII format of the path in red color presented at "Taited Sub Graph".

| **C-Src** | **Full Dep Graph** | **Tainted Sub Graph**| **TXT-Lines** | **Comments** |
|:----------|:-------------------|:---------------------|:--------------|:-------------|
| [![](http://homepages.dcc.ufmg.br/~fernando/images/c.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex1/ex1.c) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex1/fullGraph.png) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex1/subgraphLines0.png) | [![](http://cuda.dcc.ufmg.br/flowtracker/examples/images.png)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex1/subgraphASCIILines0.txt)|**ex1.c**: This is an example of naive string matching algorithm. The function compVar does not execute at constant time because the return instruction inside the loop could ends the function before the last iteration for some inputs.  |
| [![](http://homepages.dcc.ufmg.br/~fernando/images/c.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex2/ex2.c) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex2/fullGraph.png) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex2/subgraphLines0.png) | [![](http://cuda.dcc.ufmg.br/flowtracker/examples/images.png)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex2/subgraphASCIILines0.txt)|**ex2.c**: Again, an example of naive string matching algorithm. The function compVar does not execute at constant time because the attribution of variable isEqual could not be done for some input. |
| [![](http://homepages.dcc.ufmg.br/~fernando/images/c.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex3/ex3.c) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex3/fullGraph.png) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/safe.jpg) | ![![](http://cuda.dcc.ufmg.br/flowtracker/examples/images.png)](http://cuda.dcc.ufmg.br/flowtracker/safe.jpg)|**ex3.c**: This is a good example of a function for string comparisson which runs in constant time and can be used in cryptosystem's implementation. Of course, our tool did not found any vulnerable tainted subgraph in this example.   |
| [![](http://homepages.dcc.ufmg.br/~fernando/images/c.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex4/ex4.c) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex4/fullGraph.png) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/safe.jpg) | ![![](http://cuda.dcc.ufmg.br/flowtracker/examples/images.png)](http://cuda.dcc.ufmg.br/flowtracker/safe.jpg)|**ex4.c**: Again, this example runs in constant time. Of course we have a branch instruction, but it is not problematic because the loop runs for constant iterations and the cost of each iteration is the same. Therefore our tool did not found any tainted subgraph in this example.  |
| [![](http://homepages.dcc.ufmg.br/~fernando/images/c.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex5/ex5.c) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex5/fullGraph.png) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/safe.jpg) | ![![](http://cuda.dcc.ufmg.br/flowtracker/examples/images.png)](http://cuda.dcc.ufmg.br/flowtracker/safe.jpg)|**ex5.c**: Another example which the code runs in constant time. In this case the branch instruction for the while loop is influenced by information stored in the secret variable. However, again this loop runs all iteration and therefore the code is not dangerous. Of course, our too did not found any problem with this code.   |
| [![](http://homepages.dcc.ufmg.br/~fernando/images/c.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex6/ex6.c) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex6/fullGraph.png) | ![![](http://homepages.dcc.ufmg.br/~fernando/images/flowchart.jpg)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex6/subgraphLines0.png) | [![](http://cuda.dcc.ufmg.br/flowtracker/examples/images.png)](http://cuda.dcc.ufmg.br/flowtracker/examples/ex6/subgraphASCIILines0.txt)|**ex6.c**: At this example, the number of instructions in both side of branch instruction (true and false) is the same. However the execution time could be non constant because architectural aspects like cache miss, speculative execution and the respective pipeline flush for instance. So, Flow Tracker informs a vulnerability at this code.    |



# Contact #

If you have questions, you can submit it to brunors@dcc.ufmg.br or brunors172@gmail.com