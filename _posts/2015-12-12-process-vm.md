---
layout: post
img: distributed.JPG
category: projects
title: Process VM for Python
tags: algorithms
summary: A VM that consumes python bytecode written in python.
image: img/posts/process-vm.jpeg
---
Virtual Machines are proxies for real or hypothetical computers. Their uses are varied, sometimes a VM might be needed to simulate hardware that does not yet exist (might never exist) or sometimes to run Windows on a Mac environment. These VMs are better known as system virtual machines and this is the last time we'll talk about them in this post.

The VMs I am interested in are called process VMs whose raison d'etre is to execute a computer program without worrying about the underlying hardware.
Process VMs are what makes languages like Java, Python, Perl, Ruby etc. platform independent. It is the heart of the "write once read anywhere" paradigm.

Process VMs are modeled as state machines which have an execution stack, an instruction pointer, some registers and memory to hold state.

All VMs need a set of opcodes that they consume to change their state, for process VMs these opcodes are called bytecode. For a full description of Python bytecodes see this talk from [Europython](https://www.youtube.com/watch?v=0IzXcjHs-P8). My descriptions of the bytecode are directly taken from this talk.

### Bytecode

Bytecode gets its name from the fact that they are 1 byte long and some of them come with optional 2 bytes (op-arg) which carry extra information. They can be categorized into four types:

  - Stack manipulation opcodes - Operations like Load and Store which manipulate the VM's Stack.
  - Flow control opcodes- Operations like Jump which manipulate the instruction pointer
  - Arithmetic opcodes - Like BINARY_ADD which pops the two top items of the stack, adds them and pushes the result back into the stack
  - Pythonic opcodes - Opcodes specific to the python language such as creation of tuples

Let's take a quick look at the bytecode for some very simple python

```python
a = 128
b = 256
c = a + b
```

  The bytecode of this can be seen by passing this to python's [dis](https://docs.python.org/2/library/dis.html) module
  where it produces the following output.

```python
1           0 LOAD_CONST               0 (128)
            3 STORE_NAME               0 (a)

2           6 LOAD_CONST               1 (256)
            9 STORE_NAME               1 (b)

3          12 LOAD_NAME                0 (a)
           15 LOAD_NAME                1 (b)
           18 BINARY_ADD
           19 STORE_NAME               2 (c)
           22 LOAD_CONST               2 (None)
           25 RETURN_VALUE
```

The leftmost column with the numbers 1,2 and 3 indicate the line numbers of the original python source.
The next column is the byte offset:

- For line 1 the opcode named LOAD\_CONST occurs at byte offset 0 and the opcode named STORE\_NAME occurs at byte offset 3.

As I mentioned before some opcodes have opargs which are two bytes long. Here for example LOAD_CONST contains two bytes which stores an index (here 0) to an array (not yet shown) where the value that is to be loaded to 'a' i.e. 128 is stored. The values in brackets are suggestions from dis as to what they could refer to.

For the sake of simplicity I am hiding a lot of the state that is available to the VM. I will talk about them once I talk about how I modelled my VM.

### States
When python compiles a source file to bytecode it stores states such as the names of variables and the constants assigned to them. The listing below shows how to compile a python sourcefile and get the bytecode.

```python
input_file_name = "test_file.py"
file_ptr = open(input_file_name, 'r')
source = file_ptr.read()
file_ptr.close()
bytecode = compile(source, input_file_name, 'exec')
```

So what states are available for those 3 lines of python code?

- bytecode.co_consts = (128, 256, None)
- bytecode.co_names  = ('a', 'b', 'c')

So when LOAD\_CONST at line 1 had an oparg of 0 it referred to the 0th element in co_consts i.e. 128, and similarly LOAD\_CONST at line 2 with an oparg of 1 was referring to 256.

It should now be obvious that STORE\_NAME in line 1 was referring to index 0 from co_names which had the name of the variable 'a'.

Some other states are:

- bytecode.co_code  = Stores the bytecode
- bytecode.co_filename = Name of file in which this code object was created
- bytecode.co_nlocals = number of local variables

### Execution
It is now a good time to trace through the bytecode and see how it is executed.


### Design of the VM
The VM is modeled as a class with a few member variables to hold that state information shown above, an integer to store the instruction pointer and a stack. This is what the constructor used to look like in it's earliest implementation:

```python
def __init__(self, module):
  self.constants = module.co_consts
  self.names = module.co_names
  self.program = module.co_code
  self.nlocals = module.co_nlocals
  self.varnames = module.co_varnames
  self.argcount = module.co_argcount
  self.ip = 0
  self.stack = []
  self.func_def_args = {}
```

The most important method in this class is the execute() which __main__ method invokes. Here's what the execute method looks like:

```python
def execute(self):
  while True:
    op_code = self.program[self.ip]
    self.ip += 1
    if op_code >= dis.HAVE_ARGUMENT:
      low = self.program[self.ip]
      high = self.program[self.ip + 1]
      op_arg = (high << 8) | low
      self.ip += 2
      self.getattr(self, 'invoke_'+dis.opname[op_code])(op_arg)
    elif op_code == 83: # RETURN_VALUE
      return self.stack.pop()
    else:
      self.getattr(self, 'invoke_'+dis.opname[op_code])()
    if self.ip >= len(self.program):
      break
```

It just keeps running until it reaches the end of the bytecode which is usually a RETURN_VALUE, hence the extra else check. The condition

```python
if op_code >= dis.HAVE_ARGUMENT
```
is used to check if the opcode has op-args in which case we need to extract the bytes and the instruction pointer needs to skip ahead by 2.

The op-arg is obtained by ORing the low byte and the high byte (which is right shifted by 8 bits). Once we know the name of the opcode and we have the op-args we can invoke methods that are written to handle them.

So for example if the opcode is LOAD\_CONST then we invoke the method invoke\_LOAD\_CONST(val) which is defined as below:

```python
def invoke_LOAD_CONST(self, val):
   self.stack.append(self.constants[val])
```
The method simply uses the op-arg to index into co_consts and pick up the constant value which it pushes into the stack.

The other opcode of interest here is STORE\_NAME. This opcode uses it's op-arg to index into co_names and find the name of the variable to which it will assign whatever is on the top of the stack. Here is the implementation for it:

```python
def invoke_STORE_NAME(self, val):
  tos = self.stack.pop()
  var_name = self.names[val]
  self.setattr(self, var_name, tos)

```

The opcode LOAD\_NAME does the exact opposite of the above:

```python
def invoke_LOAD_NAME(self, val):
  var_name = self.names[val]
  self.stack.append(self.getattr(self, var_name))
```


Let's now look at function calls.

### Implementing functions
As before let's look at a simple module with function call and it's corresponding bytecode.

So the code:

```python
1  "Function implementation"
2
3  def some_func(a, b):
4     c = a - b
5     return c
6
7  some_func(3, 6)
```
In bytecode for this will be available in in two blocks, one for the main module which invokes some\_func() and the other for the definition of some\_func().
The bytecode for the function invocation looks like this:

```python
1           0 LOAD_CONST               0 ('Function implementation')
            3 STORE_NAME               0 (__doc__)

3           6 LOAD_CONST               1 (<code object some_func at 0x0000000004343ED0, file "test_file.py", line 3>)
            9 LOAD_CONST               2 ('some_func')
           12 MAKE_FUNCTION            0
           15 STORE_NAME               1 (some_func)

7          18 LOAD_NAME                1 (some_func)
           21 LOAD_CONST               3 (3)
           24 LOAD_CONST               4 (6)
           27 CALL_FUNCTION            2 (2 positional, 0 keyword pair)
           30 POP_TOP
           31 LOAD_CONST               5 (None)
           34 RETURN_VALUE
```

The bytecode at offset 6 which points to the function definition when disassembled looks like:

```python
4           0 LOAD_FAST                0 (a)
            3 LOAD_FAST                1 (b)
            6 BINARY_SUBTRACT
            7 STORE_FAST               2 (c)

5          10 LOAD_FAST                2 (c)
           13 RETURN_VALUE
```
The first new opcode is MAKE\_FUNCTION. When the execute method gets to this opcode, there are two consts in the stack

1. A string called "some_func" which will become the name of the function
2. A code object which can be further disassembled

Using these two items we can now create a function object, however the stack will not always have just two items, for example when a function has default arguments there are more items in the stack.

This difference is understood by looking at the op-arg for MAKE\_FUNCTION, in the above example it is 0 signifying no default args but will be 2 in the example below.

Let's see an example:

```python

1 def some_func(a=9, b=8):
2    c = a - b
3    return c
4
5 some_func(3, 6)
```

The bytecode for the function definition remains unchanged but the bytecode for function invocation changes to:

```python
1           0 LOAD_CONST               0 (9)
            3 LOAD_CONST               1 (8)
            6 LOAD_CONST               2 (<code object some_func at 0x00000000043C3ED0, file "test_file.py", line 1>)
            9 LOAD_CONST               3 ('some_func')
           12 MAKE_FUNCTION            2
           15 STORE_NAME               0 (some_func)

5          18 LOAD_NAME                0 (some_func)
           21 LOAD_CONST               4 (3)
           24 LOAD_CONST               5 (6)
           27 CALL_FUNCTION            2 (2 positional, 0 keyword pair)
           30 POP_TOP
           31 LOAD_CONST               6 (None)
           34 RETURN_VALUE
```
The first two LOAD\_CONSTs point to the default arguments followed by the function implementation and name.

So now I can show what my implementation for the MAKE\_FUNCTION looks like:

```python

def invoke_MAKE_FUNCTION_MODULE(self, val):
  func_name = self.stack.pop()
  func_code = self.stack.pop()
  def_params = []
  # fetch the default params
  while val > 0:
    def_params.append(self.stack.pop())
    val -= 1

  # This function object won't actually get invoked, it exists to store the default args
  func_vm_template = PyByteVM(module=func_code)
  # set the default params inside the function object
  for i, item in enumerate(def_params):
    func_vm_template.func_def_args[func_vm_template.varnames[func_vm_template.argcount-1-i]] = item
  self.stack.append(func_vm_template)
```
The steps involved above are:

1. Store all the default params in a list
2. Create a new object of PyByteVM passing in the func_code to the constructor, as per my design I use this object as a template object with which I shall create a new object during function call time
3. Set the default args in the dictionary called func_def_args in this new object
4. Push this new object into the stack

Why create a new instance of PyByteVM?

Because I want to maintain scope rules. Using the same object as before could potentially affect the state of variables which may have the same names. Besides I also need the instruction pointer to be set to 0 for the first opcode of the function call. I'll be doing this again when I deal with loops albeit with some minor changes.

I mentioned that I used the newly created object as a template object. Of course there is no such thing as a template object in python, this name is of my own devising.

The reason for calling this object a template object is because I will never directly invoke the execute() method on this object. Instead I will use it at function call time to create a new object on which I will invoke the execute() method.

This is done because as I mentioned before the VM is modeled as a state machine. If I called this object, then by the time I returned from this function, I'd have affected it's state in some way. If this function was invoked again (say recursively), that state from before would still be present, thereby creating unwanted behavior.

This is entirely a design choice, I use python's setattr and getattr functions to affect states inside these objects, you could declare a map where you set the states. But in that case each time you called the function's execute() method you'd have to copy the states afresh.

The last thing I need to clear up, what's happening in these two lines?

```python
for i, item in enumerate(def_params):
  func_vm_template.func_def_args[func_vm_template.varnames[func_vm_template.argcount-1-i]] = item
```

func\_def\_args is a map present in PyByteVM where I store the default arguments, varnames is a list containing the names of all the variables. With the variable name as key I set the value that it is supposed to have in the map.

Now let's tackle CALL\_FUNCTION. The low byte of it's op-arg indicates the number of positional parameters while the high byte indicates number of keyword arguments. The implementation is as follows:

```python

def invoke_CALL_FUNCTION_MODULE(self, val):
  # The low byte of argc indicates the number of positional parameters
  num_posn_args = 15 & val
  # the high byte the number of keyword parameters
  num_keyw_args = val >> 8
  posn_args = []
  keyw_args = {}
  # On the stack, the opcode finds the keyword parameters first.
  # For each keyword argument, the value is on top of the key
  for i in range(num_keyw_args):
    value = self.stack.pop()
    key = self.stack.pop()
    keyw_args[key] = value

  # Below the keyword parameters, the positional parameters are on
  # the stack, with the right-most parameter on top
  for i in range(num_posn_args):
    posn_args.append(self.stack.pop())
  posn_args.reverse()
  # Below the parameters, the template function object which will be used to create a new function object
  func_template = self.stack.pop()
  # the actual function object that will be invoked
  func_code = PyByteVM(func_template.module, func_template.globals)
  #  copy the default params from the template
  for key in func_template.func_def_args:
    func_code.set_in_exec_frame(func_code, key, func_template.func_def_args[key])
  # set the keyword pairs
  for key in keyw_args:
    self.set_in_exec_frame(func_code, key, keyw_args[key], func_call=True)
  # set the positional args
  for i, item in enumerate(posn_args):
    self.set_in_exec_frame(func_code,func_code.varnames[i], item, func_call=True)
  # pushes the return value
  self.stack.append(func_code.execute())
```

The steps here are very simple:

1. Get the number of positional and keyword parameters
2. The keyword params are first on the stack, so store them in a map
3. Next put the positional arguments into a list and reverse them since a stack implicitly reverses them
4. Then pop off the template function from the stack and use it to create the object that we will invoke
5. Retrieve the default parameters from the template object and set them in this new object
6. Then set the keyword arguments
7. Followed by positional arguments
8. Call the execute method on this object
9. Push the value returned from execute on to the stack

The reason for setting the default parameters first is so that they can be overwritten by the keyword or positional arguments.

Let's move on to loops.

### Loops
What's so special about loops?

Scope for one, we have to tackle all the scope problems that I dealt with when implementing functions, with the extra trouble that while the variables declared outside functions are hidden inside the function, variables declared in the outer scope are still accessible inside a loop.

For this reason the implementation of the constructor had to be changed. Here's what it looks like now:

```python
def __init__(self, module=None, globals_object=None, parent=None):
  if parent is None:
    self.constants = module.co_consts
    self.names = module.co_names
    self.program = module.co_code
    self.nlocals = module.co_nlocals
    self.ip = 0
    self.varnames = module.co_varnames
    self.argcount = module.co_argcount
    self.globals = globals_object # New item
    self.parent = None # New item
  else: # object created for loops
    self.constants = parent.constants
    self.names = parent.names
    self.program = parent.program
    self.nlocals = parent.nlocals
    self.ip = parent.ip
    self.varnames = parent.varnames
    self.argcount = parent.argcount
    self.globals = parent.globals # New item
    self.parent = parent # New item
  self.module = module
  self.stack = []
  self.context = None
  # stores the execution frame in which a variable is declared
  self.var_contexts = {} # New item
  # stores the default arguments when a function is made
  self.func_def_args = {}
```

The constructor takes two new parameters of which only the parent is of interest here. The globals object is just a class where I the store global variables. It's modeled as a singleton object.

The else block above sets all the same states as the parent (except for all the runtime states set using setattr and getattr) including the instruction pointer.

The parent object changes the PyByteVM to a linked list like structure. Say we had this piece of code:

```python
a = 1
b = 8
c = 8
d = 10
while b > 3: # Let's call this block A
  a += 1
  b -= 1
  while c > 4: # Call this block B
    b -= 2
    c -= 1
    while d > 4: # Call this block C
      b -= 1
      d -= 1
```

So the PyByteVM object for block C would hold the reference to block B which holds the reference to block A which in turn has a NULL object for it's parent.

In block C when we refer to variable 'd' the following steps would occur:

```python
Set context <- block C

loop while context is not NULL:
  Search for 'd' in the state of current context:
    If found:
      In the map var_contexts, set the context with 'd' as key
      return the value for 'd'
    If not found:
      Set context <- context.parent
```

The code for this looks almost the same as the above pseudocode

```python
def get_from_exec_frame(self, context, var_name):
  # first set this to None to indicate current context
  self.context = None
  while context is not None:
    try:
      var_value = getattr(context, var_name)
      self.context = context
      break
    except AttributeError:
      context = context.parent
  self.var_contexts[var_name] = context
  if context is None:
    print('Incorrect byte code')
  return var_value
```

Now let's look at a simple while loop's bytecode.

```python
a = 1
b = 2
while a < 10:
    b = a + b
    a += 2
```

Becomes

```python
1          0 LOAD_CONST               0 (1)
           3 STORE_NAME               0 (a)

2          6 LOAD_CONST               1 (2)
           9 STORE_NAME               1 (b)

3         12 SETUP_LOOP              36 (to 51)
          15 LOAD_NAME                0 (a)
          18 LOAD_CONST               2 (10)
          21 COMPARE_OP               0 (<)
          24 POP_JUMP_IF_FALSE       50

4         27 LOAD_NAME                0 (a)
          30 LOAD_NAME                1 (b)
          33 BINARY_ADD
          34 STORE_NAME               1 (b)

          37 LOAD_NAME                0 (a)
          40 LOAD_CONST               1 (2)
          43 INPLACE_ADD
          44 STORE_NAME               0 (a)
          47 JUMP_ABSOLUTE           15
          50 POP_BLOCK
          51 LOAD_CONST               3 (None)
          54 RETURN_VALUE
```

The opcode of interest is SETUP\_LOOP, the op-arg for this provides the number of byte offsets to jump to clear the loop.
Here is how I implemented this function.

```python
def invoke_SETUP_LOOP(self, val):
  loop_exec_frame = PyByteVM(parent=self)
  self.ip += val
  loop_exec_frame.execute()
```
So a new object of PyByteVM is created using the current instance of PyByteVM and the execute() method of this new object is invoked.
Then the instruction pointer of the current object is incremented by the value of val.

Note that in the new instance of PyByteVM the instruction pointer points to the old value of the previous instance of PyByteVM which brings up LOAD\_NAME at byte offset 15. The opcodes which perpetuate the loop are JUMP\_ABSOLUTE where the op-arg points to the byte offset to jump to and POP\_JUMP\_IF\_FALSE where the op-arg points to the byte offset to jump to when the loop terminates.

### Builtins

I am going to cheat and pass implementations of functions like print and range to their default python implementations.
All I need to do is detect when such a function is invoked. Let's take a very simple example

```python
a = 124
print(a)
```

The opcode of this becomes:

```python
1           0 LOAD_CONST               0 (124)
            3 STORE_NAME               0 (a)

2           6 LOAD_NAME                1 (print)
            9 LOAD_NAME                0 (a)
           12 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
           15 POP_TOP
           16 LOAD_CONST               1 (None)
           19 RETURN_VALUE
```

The two opcodes (LOAD\_NAME) before CALL\_FUNCTION are the positional arguments and the function name.
When LOAD\_NAME is invoked for print, it will not be found in any instance of PyByteVM or in the globals. I use this to set a state which indicates that a builtin function will be invoked at CALL\_FUNCTION time.

### Conclusion

I originally intended to add support for Classes, and I might come back to this later. For now the project is in a stable state and in spite of the few imperfections I feel like Frederick.

![Alt](http://socialcomotion.com/Blog/wp-content/uploads/2015/10/its-alive.jpg)
