---
title: Angr Ultime Guide Part 1 Introduction 
categories: REVERSE 
image: null
published: true
---

![angr](/assets/images/angr.jpeg)
<br>
Angr the most powerful tool for symbolic execution and binary analyse


Created by ASU University researchers, Angr is a powerful tool for symbolic execution and binary analys.
It is used to automate  reverse engineering challenge like crackmes solving  by making symbolic execution and also to found bug like buffer overflow through static and dynamic binary analyse (also symbolic execution)

If you are CTF Player especially PWN/REV challenges,  it will be very important for you to know Angr. It will help you automate crackmes solving and win time durant CTF competition.
Angr is not only made for CTF players. It is made also for real live vulnerabilities researchers (Reverse engineers, bug hunters, and so on).
So if you plan to make reverse engineering or vulnerabilities researche (binary side)  your carrer, it will be very important for you to master Angr skills.
In real live, Angr was used to found a lot of bug and solve insane challenges


### In this Guide, i will try to explain you Angr framework as most as possible.  To do that, i will use the official documentation and also others article about Angr that i found on the net. It will not be therorical guide,  we will solve together challenge by using angr . Which will help us to understand the inner working of this powerfull tool. Be aware that , mastering this tools can take some time !!
[Getting start: Angr installation && Project](#Get-start)

[Understand symbolic execution](#Symoloc-execution)

[Symbolic Execution: State, Block, Bitvector](#Important-concepts)



# Getting start {#Get-start}
- Angr installation: <br>
  Angr is a python module so you can install it by using pip <br>
  ``python3 -m venv myvirtual_env; source ./myvirtual_env/bin/active; pip install angr``

  The above command create a virtual envirnment named myvirtual_env and install angr on that virtual envirnment. I strickly recommand you  to also install Angr on a python virtual envirnment
  <br> If everything goes well, you can start using Angr on your python script by importing it.
``pytohn import angr ``

- Angr Project:  <br>
  With Angr, the most part of things that you will do, you will perform it on some project. Considere an Angr project as a program/binary that you try to reverse engineering or analyse. Thing about project on Ghidra or other reverse engineering tools.
<br> As angr is a python3 module. to create a project, you must import it and use it's classe ``Project``. Bellow is an example of angr project with /bin/ls binary

```bash
$ipython3
Python 3.11.2 (main, Apr 28 2025, 14:11:48) [GCC 12.2.0]
Type 'copyright', 'credits' or 'license' for more information
IPython 9.1.0 -- An enhanced Interactive Python. Type '?' for help.
Tip: Use the IPython.lib.demo.Demo class to load any Python script as an interactive demo.

In [1]: import angr

In [2]: proj = angr.Project("/bin/ls", auto_load_libs=False, main_opts={"base_addr": 0})

In [3]: 
```

The mandatory param to create a project is a binary file path. Don't worry about others param that i passed to Project constructor.  In other chapter, we will explore all of theme.

After you create a project, all things you will do with Angr will be onside those project (except a little thing that you will do with claripy without having Angr Project


- Supported platforms <br>
  Angr is made to be cross plateform project. <br>
With angr, you can work on ELF, PE, Mach-0 binary. You can also work with Android APK binary (JVM)
So as we created a project with and ELF on the last example, you can do that on windows by using a PE binary

- Congrate: you create your first project with Angr tool. Keep reading and you will get a lot of powerful

# Understand symbolic execution {#Symoloc-execution}
<br> Symbolic execution is the technical used to execute a program by considere it object as symbolic object not concret objet. By symbolic object, i reffer to an object which don't have a concret value. It is a symbole . This technic is most used to solve constraints,  to explore all path of a program and found wich value can make program reach some part. In real execution, when you have  int variable counter, in some state of program runtime, counter will have a concret value ( 0 for example).  In symbolic execution, that is not the same, counter can have multiple values(symbolic values) .  That is the powerfull of symboli execution.
<br>
For example, execution of this snipped program: 
```C
if (a > 20){
  do_good_thing();
  }else{
  do_bad_thing()
  }
```

In concret execution case, when the execution reach the condition part, only one of those functions (do_good_thing and do_bad_thing) will be called.
<br>
But in symbolic execution context, the two funcitons can be called. Because `a` variable is a symbolic variable ( it value is a symbolical value) and can satisfy the two condition (a > 20 and a <=20). 
<br>The internal work of this process is to considere in first place that the value of `a` is not greater than 20 and call do_good_thing and also considere it value as not greater than 20 and call do_bad_thing. So the workflow of this symbolic execution create two states
The first one where  variable `a` is  greater than 20 and  the second on where variable `a` is not greater than 20.
<br>
Suppose that the `a` variable is a user input variable, this workflow can help us to determinate which values user must enter to call do_good_thing and which one she/he will also enter to call do_bad_thing function just by inspecting the value of `a` variable on the two created states of the program at the end of the condition.
<br>

# Symbolic Execution: State, Block, Bitvector {#Important-concepts}
  To master your symbolic execution skills with Angr framework, there is some concepts thar are mandatory to understands. This part is made to explain those concepts

# State
  - State
  What is a state!? <br>
  A `state` is a program and all it compoments (memory, registers, varibables, stack, heap) at some point of it execution. Considere it as a screenshot of a program at some point of it execution.
  The concept of state is very important in symbolic execution especialy with Angr. For example, in crackme solving with Angr, your purpose is to found an user input that will make program reach some state ( the state that print congratulation valide license key for example). If you reach that state, you get the valide licence to enter. 
  <br>
  Angr framework has multiple state:

  - full_init_state <br>
    It is a state that can be considere as a program on it runtime initial part (when the program is about to run _init_ or something like that). <br> Here is how to create the full_init_state in Angr: <br>
    
```python

$ipython3
Python 3.11.2 (main, Apr 28 2025, 14:11:48) [GCC 12.2.0]
Type 'copyright', 'credits' or 'license' for more information
IPython 9.1.0 -- An enhanced Interactive Python. Type '?' for help.
Tip: Use the IPython.lib.demo.Demo class to load any Python script as an interactive demo.

In [1]: import angr

In [2]: proj = angr.Project("/bin/ls", auto_load_libs=False, main_opts={"base_addr": 0})

In [3]: help(proj)


In [4]: full_init_state = proj.factory.full_init_state()

In [5]: 

```
- Entry state: <br>
The `entry state` is a state of the program when it is about to run the main function. <br>
Bellow is how to create it with Angr project <br>

```python
$ipython3
Python 3.11.2 (main, Apr 28 2025, 14:11:48) [GCC 12.2.0]
Type 'copyright', 'credits' or 'license' for more information
IPython 9.1.0 -- An enhanced Interactive Python. Type '?' for help.
Tip: Use the IPython.lib.demo.Demo class to load any Python script as an interactive demo.

In [1]: import angr

In [2]: proj = angr.Project("/bin/ls", auto_load_libs=False, main_opts={"base_addr": 0})

In [3]: help(proj)


In [4]: full_init_state = proj.factory.full_init_state()

In [5]: entry_state = proj.factory.entry_state()

In [6]: 

```


- The call state <br>
 a `call state` is a state that represente a program when it is about to call some specific function ( do_good_thing for example)
 <br>Here is how to create it: 
 ```python
 call_state = proj.factory.call_state(addr_of_function, arg1, arg2,ar3, **kwars)
 #addr_of_function is the address of tunction to call: do_good_thing for example
 #argx are function argument
 #**kwargs is a dictionary that hold other argument like function call convention to use and others things
 ```


 - Blank state: <br>
 The `blank state` it a state of a program at some address with the most data unintilize
 Bellow is how to create it
 ```python
 blank_state = proj.factory.blank_state(addr, **kwars)
 #addr is the address where the execution start:
 #**kwargs is a dictionary that hold other argument like function call convention to use and others things
 ```




# Block
- What is a `block` ?
<br>
A `block` is a suite of instructions ( i talk about asm instruction) which end with  some of those thee instuctions (call,jmp.., ret).
When the program reach the first instruction of a block, all of that `block` instructions will be executed before the program jump else where.
When you use tool like IDA to view the graph representation of a binary, all node is a block.
It is very important to understand the concept of `Block` because it define the workflow of a program.

Example of a block: malloc function prelogue <br>

```asm
        0010120d 55              PUSH       RBP
        0010120e 48 89 e5        MOV        RBP,RSP
        00101211 48 83 ec 20     SUB        RSP,0x20
        00101215 64 48 8b        MOV        RAX,qword ptr FS:[0x28]
                 04 25 28 
                 00 00 00
        0010121e 48 89 45 f8     MOV        qword ptr [RBP + local_10],RAX
        00101222 31 c0           XOR        EAX,EAX
        00101224 bf 20 00        MOV        EDI,0x20
                 00 00
        00101229 e8 d2 fe        CALL       <EXTERNAL>::malloc                               void * malloc(size_t __size)
                 ff ff

```


# Bitvector

A `Bitvector` is some kind of variable that is a suite of bits in memory. Thing about the int type in C langage. Angr is python module and the int type in python is not the same as the int of C langage. A bitvector is use to map C like int value. It is not C int type. I take the example of C int type to explain how the bitvector is mapped into the memory. 
`Bitvector` is very useful especially in symbolic execution.
Angr come with a submodule call `claripy` which is the constraints solver engine that help us to create a `bitvector` type

<br>
With claripy, we have to kind of bitvector:  `BitVectorValue` which has a predefine value and `BitVectorSymbol` which hold a symbolic value useful for symbolic execution
<br> Bellow is how to create theme:
```python
import claripy
# create bitvectorValue with 0x414141 as it value and 32 as it length
bitvectorvalue = claripy.BVV(0x414141, 32)

# Create bitvectorSymbolic x of 64 lenght
bitvectorsym = claripy.BVS("bitvectorSym", 0x40)

```

In symbolic execution, we use bitvector to presente data like flag, or license key. It is also used in constraints solving



# End
With all of those knowledges, you are well armed to deep into Angr usage and get a lot of powerful in your reverse engineering or vulnerabilities research task. In the next Chapter, we will deep more and see some usage case of Angr  <br> Thank you for your reading and see you next time.<br> 

# Don't forget to reach me if you found mistake @Top0n3.
