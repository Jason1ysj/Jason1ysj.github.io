# JVM 

[toc]

##  1. Memory Space 

### 1.1 JVM Memory Components
Five components: **Method Area, VM Stack, Native Method Stack, Heap, Program Counter Register**.  
**Direct Memory** is not defined in the JVM Memory Space specification.

- Program Counter Register
	- private to each thread
	- The only memory space does not have JVM specified **OutOfMemoryError**
- Java VM Stack
	- private to each thread, same lifecycle as the thread
	- Possible exceptions: **StackOverflowError**, **OutOfMemoryError**
-  Heap
	- Shared memory space among all threads
	- Major place where GC works
	- Logically continuous, but not physically
- Method Area
	- shared by all threads. 
	- It stores the code information (class, constants, code) that has already been loaded into the JVM.
	- almost "permanent generation"
	- **Runtime Constant Pool** belongs to the **Method Area**. It is dynamic during the runtime. 

### 1.2 Object

A object has two types of data: Instance Data (Heap) and Class Data (Method Area)

Type of Reference:
- Handle Pool (句柄池)
	- Reference in Stack --> Handle Pool --> Instance and Class Data
	- One extra memory access 
	- no need to change reference when data moved
- Direct pointer
	- Reference in Stack --> Instance Data (Heap) --> Class Data (Method)

### 1.3 OutOfMemoryError Debug
- Heap
	- `-Xms, -Xmx`: min and max size of Heap
	- `-XX:+HeapDumpOnOutOfMemoryError`: Dump Heap Memory snapshot when OutOfMemoryError happens
	- use GC reference list to check Memory Leak

- VM Sack & Native Stack
	- `-Xss`
	- One large thread stack: StackOverflowError
	- Two many thread stacks: OutOfMemoryError

- Runtime Constant Pool
	- `-XX:PermSize, -XX:MaxPermSize`

## 2 Garbage Collector

### 2.1 GC Root Tracing
- Start at **GC Roots** and search the path of **Reference Chain**. If an object is unreachable from the GC Roots, then it is ready to collect. 
- Reference types
	- **Strong Reference**
	- **Soft Reference**: live until the second GC cycle
	- **Weak Refernce**: live until the next GC cycle
	- **Phantom Reference**: does not affect the object's lifecycle
- Death of an object
	- at least being **marked twice**
	- After first mark, it would be put into the **F-Queue** if it needs to execute **finalize()**
	- **F-Queue** does not guarantee it would wait for the **finalize()** to finish. 
	- If an object can put it back to the **Reference Chain** in the **finalize()**, then it becomes alive again. 
	- **finalize()** can be executed at most for **only one** time.
	- If an object is going to be recycled for the **second** time, its **finalize()** would not execute.  
- **Method Area** GC
	- **Runtime Constant** is similart to the Heap objects
	- Unused **Class** has strict requirements. Although the class meets the requirements, it is not necessarily being recycled. 

### 2.2 GC Algorithm
- **Mark - Sweep**
	- Two Phases: First **mark** and then **clean**
	- Weakness
		- not efficient
		- free memory not continuous
- **Copying**
	- Only use 50% of the memory space. For each GC cycle, it copies the living objects to another 50%.
	- Strength: efficient, continuous free MS
	- Weakness: waste of MS
	- It is used to GC the **Young Generation**
		- Eden : Survivor : Survivor = 8 : 1 : 1
		- Copy the living objects from the **Eden** and one **Survivor** to another **Survivor**
		- If the **Survivor** does not have enough space, it requires the **Old Generation** to **Handle Promotion**
- **Mark - Compact**
	- If an object has high survival rate, then it is very costly to apply the **Copying Algorithm**
	- Compact the living objects to a continuous space, and then the rest memory space.

### 2.3 GC 
![GCs](https://i.ibb.co/jhwTS4T/GCs.png)

#### 2.3.1 Serial 
![Serial](https://i.ibb.co/ZTx14rB/Serial.png)
- Single thread, Copy Algorithm
- **Stop the World**: it stops all the users' works, and perform the GC. 
- Only apply to the **Young Generation**
- Strengths
	- Simple: fast on small size young generation, good for "Client App"
	- High Efficiency on single CPU, no threads contention

#### 2.3.2 ParNew
- **Parallel**: multiple threads for garbage collec- Not Concurrent, still **Stop the World**
- Only apply to the young generation

#### 2.3.3 Parallel Scavenge
- Copy Algorithm, Parallel GC threads
- instead of optimizing the **pause time** for GC, it optimizes **throughput** of user code execution
- Not Compatible with CMS
- Three params to tune the GC
	- `MaxGCPauseMillis`: the maximum time period for each GC cycle. 
	- `GCTimeRatio`: GC time / total running time, it is related to the **throughput** 
	- `UseAdaptiveSizePolicy`: It monitors the JVM and **automatically** set the memory size for each generation. This adaptive method is called **GC Ergonomics**. (**Major differenc**e compared to **PairNew**)

#### 2.3.4 Serial Old
- Mark-Compact Algorithm, Signle Thread, Old Generation
- For Client Mode, not Server Mode

#### 2.3.5 Parallel Old
- Parallel, Mark-Compact, Old Generation
- Design to be compatible with CMS

#### 2.3.6 Concurrent Mark Sweep (CMS)
![CMS](https://i.ibb.co/vYhtdRm/CMS.png)
- Concurrent, Mark-Sweep, Old Generation
- Designed for **Low Pause**
- Four Steps
	- CMS initial mark
		- Stop the World, Single Thread
		- Only mark the objects **directly** related to the **GC Roots**
		- Short task
	- CMS concurrent mark
		- Concurrent with User program, Parallel
		- To do the **GC Roots Tracing**
		- long task
	- CMS remark
		- Stop the World, Parallel
		- Revise the changes during the User Program running period
	- CMS concurrent sweep
		- Concurrent with User program, Parallel
		- Clean the marked objects
- **Drawbacks**
	- Sensitive to the CPU resources. When CPU resouces is not enough, it may cause relatively large performance down on the user program. 
	- **Floating Garbage**: can not clean the garbage generated at the Mark period. 
	- **Concurrent Mode Failure**: the pre-allocated memory space for user program is not enough, so that the concurrent work would fail. It would tentatively use **Serial Old** to finish the job. 
	- **Memory Fragment**: Needs further operation to compact the free memory. The param `UseCMSCompactAtFullCollection = default is 1` would compact the memory when a Full GC is needed. 

#### 2.3.7 Garbage-First (G1)
![G1](https://i.ibb.co/F7DyKkf/G1.png)
- Designed for Server Mode, to replace CMS
- Parellel and Concurrent, Mark-Compact
- Do not need other GC, it is responsible for all genrerations
- Can **predict** the pause time, major advatange compared to CMS
- Implementation Details
	- Divide the Heap into multiple **Regions** with same size. Have the concept of young and old generation, but they are not continuous in memory space. They are not physically separated. 
	- How it **predicts** the **pause time**: It has plan to avoid Full GC. It traces each Region's **value**, which is calculated by the memory benefit and time cost.  It recycles in the **priority** of their values. 
	- JVM applies **Remembered Set** to save the cross-region object reference information. For very **Write** operation, it generates a **Write Barrier** to interept and check cross-region write. 
- Four steps
	- Initial Marking: same as CMS
	- Concurrent Marking: same as CMS
	- Final Marking: 
		- changes are stored in the **Remembered Set Logs**
	- Live Data Counting and Evacuation
		- Evaluate each Region's recycle cost and benefits
		- It can be **concurrent** or **Stop the World**. Stop the World would be more efficient because only a part of Regions are recycled. 

### 2.4 OopMap (TODO)Multi-thread version of **Serial**

### 2.5 Heap Allocation

#### 2.5.1 Preferentially Allocate in Eden
- First try to allocate in Eden. If not enough space, trigger a Minor GC
- If still lack of space after Minor GC, move objects from Young to Old by **handling promotion**

#### 2.5.2 Large Object Directly in Old Generation
- `-XX:PretenureSizeThreshold` every obejct larger than this value would directly allocated in Old Generatino.

#### 2.5.3 Long-lived Object move to Old Generation
- `Age counter`: it increases by 1 every time it survives in a Minor GC and moves to the Survivor. 
- `-XX:MaxTenuringThreshold`: every object whose age is larger than this value, it would be moved to Old Generation

#### 2.5.4 Dynamic Age Determination
- If same-age objects exceeds **half** of the Survivor's size, then objects with their ages **equal or larger than** that age would be moved to Old Generation

#### 2.5.5 Handle Promotion
- `HandlePromotionFailure = False`
	- If **the largest continuous free space** is smaller than all objects in the Young Generation, then it would trigger a Full GC
- `HandlePromotionFailure = True`
	- Before triggering a Full GC, it would check whether **the largest continuous free space** is larger than the **average size** of objects moving to Old Generation During each Minor GC. If it is larger than the average size, it would try to do a **Minor GC in risks**. 


## 3. Class File

### 3.1 Irrelevance
- The **ByteCode** can run on different **platforms** and can be compiled from different **programing languages** 

### 3.2 Class File Structure
- A class file may not exist on the disk, it can be generated by **Class Loader** at runtime
- **Magic Number**: `0xCAFEBABE` is an identifier of a Class File
- No gap in the Class file, all placed densely. 

#### 3.2.1 Major data in the Class File
- **Constant Pool**: stores **resources**
- The index of `this_class` and `super_class`. Can have at most one Super class
- A set of index of `interfaces`
- `field_info`: consists of multiple **Field Tables**
- `method_info`: consist of mutiple **Method Tables**
- `attribute_info`: the format of attribute_info can be user-defined
	- **Code**: Byte codes after Javac compilation. It includes the `try, catch` exception tables
	- **Exceptions**: describe exceptions after the `throws` in method descriptor
	- **LineNumberTable**: map byte code to line code
	- **LocalVariableTable**: map the local variables to their defined names
	- **SourceFile**: The name of that source file
	- **ConstantValue**: store the initial values of **staic final** variables (CONSTANT), only applies to **Basic Type and String**
	- **InnerClasses**: each inner class is represented as one `inner_classes_info` table

#### Tips
- Can not **overload** `Field`
- In Java, you can not overload a function with only **return type** changed. However, you can do that in the Class File.
- The **finally block** will be executed even **after a return** statement in a method.

### 3.3 Instruction
- **Opcode**'s length is fixed to 1 Byte (0 - 255)
- Not all data type has its own operation. For example, the `Byte, Char, Short, Bollean` does not have dedicated Arithmetic Operations. 

#### 3.3.1 Java Stack
![Java Stack](https://i.ibb.co/WcccS14/jvm-stack.jpg)
- JVM does not have the concept of **Register**. All the operations are done by the **Operand Stack**
- **Dynamic Linking** stores the **References** of outer class's methods. 
	- Because Java programs are dynamically linked, references to methods initially are **symbolic**.

#### 3.3.2 Method Invoke and Return
- `invokevirtual`: the method of an **object instance**
- `invodeinterface`: the method of an interface
- `invokespecial`: special instance methods, including `init`, `private method`, `super method`
- `invokestatic`: static methods
- `invokedynamic`: TODO, user defined invoke logic

#### 3.3.3 Synchronous 
- Controlled by the `Monitor` module. 
- **synchronized block** is achieved by both the *Javac* and *JVM*

## 4. Class Loading

### 4.1 Class Lifecycle
![Class Lifecycle](https://i.ibb.co/zHTq9rK/class-lifecycle.png)
- The order `Loading -> Verifcation -> Preparation -> Initialization -> Unloading` is fixed. But the **Resolution** may take place after **Initialization**
- **Initialization** corresponds to the `<cinit>` of a class. 

### 4.2 Loading Procedures

#### 4.2.1 Loading
- Steps
	- Get Binary Byte Stream by the Class's **Fully Qualified Name**
	- Translate the Stream data and store it to the **Method Area**
	- Construct a `java.lang.Class` object in Memory, which is the **Entry** for the Method Area data. 
- Array Object Loading is **built-in** in JVM. Unarray Object Loading can be done by **user-defined** Loader.

#### 4.2.2 Verification
- Ensure the Security of the code. JVM woudl throw `java.lang.VerifyError` Exception when Class file is illegal.

##### 4.2.2.1 File Format Verification
- Magic Number. Version Number
- Constants Pool, and ...
- After this step, all data are transformed to the Method Area, where the following verfications work on.

##### 4.2.2.2 MetaData Verification
- such as private, public, protected, final
- super class, inheritance, interface, abstract

##### 4.2.2.3 Byte Code Verification
- Most complicated, check the semmatic of the code by go through the **Data-flow** and **Control-flow**. Try to explore the vulnerabilities in the code.
- **StackMapTable** in the Class File can help to improve the efficiency of this process. But still vulnerable to the attack.

##### 4.2.2.4 Symbolic Verification
- Execute when JVM transforms the symbolic references to the direct references. It takes place at **Resolution** process.
- Check whether the local references to the outer classes are legal (compatible, access permission) or not. If not, it throws `java.lang.IncompatibleClassChangeError`

#### 4.2.3 Preparation
- Allocate memory in **Method Area** for the **static** variable and initialize their values. 
- For the static variable **without** `final` tag, the intial value would be 0. 
	- For example, `public static int tValue = 123`. The `tValue` after **Preparation** would be 0 instead of 123. Because this statement is compiled into `<cinit>` by *Javac*. The `<cinit>` would execute at **Initialization**. 
	- For example, `public state int final tValue = 123`. The `tValue` after **Preparation** would be 123. Because the `tValue` has **ConstantValue** attribute in **Field Attribute Table**.

#### 4.2.4 Resolution
- Translate **Symbolic** References to **Direct** References, such as `CONSTANT_Class_info, CONSTANT_Fieldref_info, CONSTANT_Methodref_info`
- The **Resolution** may execute during the **Loading Procedures** or before the usage of that reference
- For each Class Loader, one reference would be Resoluted for at most one time. 

##### 4.2.4.1 Class or Interface Resolution
- The Target is **C**. JVM would let the **Class Loader** to load C. During the loading, it may trigger other loadings of other Classes. If any part fails, then the Resolution fails. 
- Before Resolution succeeds, it needs to do the **Symbolic Verification**. If it does not have corresponding access permission, then it throws `java.lang.IllegalAccessError`

##### 4.2.4.2 Field Resolution
- First Resolute the field's `CONSTANT_Class_info`, which is called **C**
- If C itself has the target Field, then it returns the reference.
- Else if C's **interfaces or parent interfaces** have the target Field, then it returns.
- Else if C's **parent classes** have the target, then it returns
- Else, throws `java.lang.NoSuchFieldError`
- If the class does not have access permission on the Field, then throws `java.lang.IllegalAccessError`

##### 4.2.4.3 Static Method Resolution
- First Resolute the static method's `CONSTANT_Class_info`, which is called **C**
- If C is an **interface**, then throw `java.lang.IncompatibleClassChangeError`
- Else if the static method exists in C or C's parent, then returns
- Else if the static method exists in C's **interfaces**, then C is an abstract class. Throw `java.lang.AbstractMethodError`.
- ELse, resolution fails, throw `java.lang.NoSuchMethodError`
- If the class does not have access permission on the Field, then throws `java.lang.IllegalAccessError`

##### 4.2.4.4 Interface Method Resolution
- First Resolute the static method's `CONSTANT_Class_info`, which is called **C**
- If C is a **Class**, then throw `java.lang.IncompatibleClassChangeError`
- Else if the method exists in C or C's parent, return the reference.
- Else, resolution fails, throw `java.lang.NoSuchMethodError`
- All Interface Methods are **public**, no need to check the permission

#### 4.2.5 Initialization
- Execute the `<cinit>` method. (user code)
- `<cinit>` is not mandantory for Class. The parent `<cinit>` would always execute before the child by JVM. 
- `<cinit>` is **atomic**. JVM runs it with locking and synchronously. 

### 4.3 Class Loader
- The two classes loaded by two **different Class Loaders** from the **same Class File** are **different**. For example, `equals(), isInstance()`

#### 4.3.1 Parents Delegation Model
![Class Loader](https://i.ibb.co/4j4mPkz/class-loader.png)
- Bootstrap ClassLoader
- Extension ClassLoader
- Application ClassLoader: load user code

#### 4.3.2 Break Parents Delegation Model
TODO: OSGi, HotSwap, Hot Deployment


## 5. Method Execution

### 5.1 Local Variable Table
- Storage for **Method Params** and **Method Local Variables**.
- The storage unit is called **Slot**, usually 32 bits. 
- JVM uses Local Variable Table to pass the method parameters. (Hidden param `this` for non-static method)
- The Slot can be reused by different blocks of the method, which may results in anormal behaviors of GC. 
- To improve the performace, the **Operand Stack** may overlap with **Local Variable Table**


### 5.2 Method Call

#### 5.2.1 Resolution
- Only translate **Non-Virtual Method**'s **symbolic** reference to **direct** reference
	- include **Static, Private, cinit, Parent, Final Methods**
	- Call by instruction `invokestatic, invokespecial`

#### 5.2.2 Dispatch

##### 5.2.2.1 Method Overload Resolution (Static Dispatch)
- Designed for **Overload Methods**
- It is determined during the **Compile Time**
- Choose corresponding overload method based on its **Static Type**, not the **Actual Type**
```java
// 
class Human {}
class Man extends Human {}
// Human is Static Type, Man is Actual Type
Human man = new Man();
```
- If it cannot find the exact matching method, it would choose the most propriate one. 
```java
// For example, priority: char > int > long
public static void main(String args[]) {
	sayHello('a')
}
public static void sayHello(char arg) {}
public static void sayHello(int arg) {}
public static void sayHello(long arg) {}
```

##### 5.2.2.2 Dynamic Dispatch
- Designed for **Overwrite Methods**, Polymorphisim
- Use `invokevirtual` instruction and it is based on the **Actual Type**
- To improve performance
	- `invokevirtual` has a **Virtual Method Table (vtable)**
	- `invokeinterface` has a **Interface Method Table (itable)**
	- These tables has the Direct references (entry) to the methods. 

#### 5.2.3 Dynamically Typed Language
- New instruction in JDK7 `invokedynamic`
- **Dynamically Typed Language** do the **Type check** during the **runtime** not the **compile-time**
	- The varible does not have a Type, but the **value** of that variable has a Type
	- The four other `invoke methods` must have `CONSTANT_Methodref_info` or `CONSTANT_InterfaceMethodref_info` in the compiled bytecode. However, **Dynamically Typed Language** does not know the reveiver's Type until the runtime. 

##### 5.2.3.1 java.lang.invoke.MethodHandle
	- Similar to the function pointer in C++, it allows a method call **without a Static Type**
```java
public class MethodHandleTest {
	static class ClassA {
		public void println(String s) {
			System.out.println(s);
		}
	}
	public static void main(String[] args) throws Throwable {
		Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();
		getPrintlnMH(obj).invokeExact("icyfenix");
	}
	private static MethodHandle getPrintlnMH(Object reveiver) throws Throwable {
		// first param: return type, following params: method params
		MethodType mt = MethodType.methodType(void.class, String.class);
		// bindTo() passes `this` as a param
		return lookup().findVirtual(reveiver.getClass(), "println", mt).bindTo(reveiver);
	}
}
``` 
- **MethodHandle**'s result is similar to the **Reflection**, but they are different at execution level. 
	- MethodHandle is at the Byte Code level, and Reflection is at the Java Code level.
	- MethodHandle is more light-weight and works for all languages on JVM, not only the Java language

##### 5.2.3.2 invokedynamic (TODO)
- It cannot be generated by **Javac**. But can run on **JVM**. 
- It does not requires the `CONSTANT_Methodref_info` which includes the `CONSTANT_Class_info`
- The first param is `CONSTANT_InvokeDynamic_info`
	- It has three components: **Bootstrap Method, MethodType, and Name**
- According to the `CONSTANT_InvokeDynamic_info`, JVM can find the Method and get a **CallSite** object. 

### 5.3 ByteCode Execution
- Two Modes to execute the ByteCode: **Interpreter** and **Compiler**
- Two types of instuction set: one is based on **Operand Stack** and another is based on **Register**
	- Operand Stack based is more portable on different hardwares
	- Register based is faster

#### 5.3.1 Proxy
- **Static Proxy**, each proxy is dedicated to one Class, needs manual code modification for different Class Proxy. 
- **Dynamic Proxy**, apply `java.lang.reflect.Proxy` to dynamically proxy Class. 
```java
public class DynamicProxyTest {
	interface IHello {
		void sayHello();
	}
	static class Hello implements IHello {
		@Override
		public void sayHello() {
			System.out.println("hello world");
		}	
	}

	static class DynamicProxy implements InvocationHandler {
		Object originalObj;
		Object bind(Object originalObj) {
			this.originalObj = originalObj;
			return Proxy.newProxyInstance(orignialObj.getClass().getClassLoader(), originalObj.getClass().getInterfaces(), this);
		}
		@Override
		public Object invoke(Object proxy, Method method, Object[] arg) throws Throwable {
			System.out.println("Welcome");
			return Method.invoke(originalObj, args);
		}
	}

	public static void main(String[] args) {
		IHello hello = (IHello) new DynamicProxy().bind(new Hello())
	} 
}
```

## 6. JIT Complier
**Interpreter** and **Compiler**, tradeoff between storage and performance

### 6.1 Hotspot
- Two types of Hotspot
	- **methods** called for multiple times
	- **loops** executed for multiple times
- HotCode Detector
	- **Sample Based Hot Spot Detection**: simple and fast. but interfered by blocking and I/O
	- **Counter Based Hot Spot Detection**: based on the sum of two counters
		- **Invocation Counter**: for method
		- **Back Edge Counter**: for loop

### 6.2 Compiling
- **OSR**: On Stack Replacement
![OSR](https://i.ibb.co/TTyL5XH/thread-state.png)



## 7. Memory & Thread

### 7.1 Cache Coherence
![Cache Coherence](https://i.ibb.co/9YzFHGQ/cache-coherence.png)

### 7.2 Memory Model
- Add a layer of virtualization over the hardware memory to deal with the cross-platform performance consistency. **Java Memory Model, JMM**.
- All variables are stored in **Main Memory** ( == physical memory)
- Each thread has its **Working Memory** which can not be accessed by the other threads.
![Memory Model](https://i.ibb.co/VpL0hfV/memory-model.png)

#### 7.2.1 Inter-Memory Operation
- 8 operations, all **atomic**
	- lock, unlock: a variable in main memory locked by one thread
	- read: read from **main memory** to **working memory**
	- load: load the value obtained from the **read** operation to the local copy
	- use, assign: between **working memory** and **execution engine**
	- store: store the value from **working memory** to **main memory**
	- write: write the value from **store** operation to that variable in main memory
- (read->load) and (store->write) must happen together 
	- the order is **reserved**, but they may not be **consecutive**

#### 7.2.2 Special Rules for Volatile
- **Volatile** Two properties:
	- It is visible to all threads
	- Once a thread changes the value, the change is propagated to every thread at once. (does not need to load back to Main Memory)
- However, Volatile is **not** guaranteed to be safe under multi-threading
	- Because the operations are not all atomic. **Race Condition** exists. 
	- Needs `synchronized` block or `java.util.concurrent` to ensure atomity
- Can **prohibit instruction reordering** optimization
	- Once the **assignment** of the Volatile variable completes, all the instruction before the assignment must finished too.
	- Because it adds a **Memory Barrier** after the assignment. 
		- **Write Memory Barrier**: Store execute the following instruction, it must store the value back to Main Memory
		- **Load Memory Barrier**
- **Volatile** is a bit faster than `synchronized` block and `java.util.concurrent`
- Two Rules for Operations on **Volatile**
	- load -> use: must be **together and consecutive**
	- assign -> store: must be **together and consecutive**

#### 7.2.3 Atomity, Visiblity, Ordering
- **Atomity**: `synchronized` block and locks
- **Visibility**: The changes of a variable are propagated to every thread at once. 
	- Volatile, synchronous, final
	- before **unlock**, the value must be written back to Main Memory
- **Ordering**: the ordering across multiple threads can be achieved by **Volatile** and **synchronized**

### 7.3 Thread

#### 7.3.1 Thread Scheduling
- **Cooperative Threads-Scheduling**: Thread execution time is determined by itself. Other threads maybe blocked by Errors or I/O
- **Preemptive Threads-Scheduling**: System determine the execution time of each thread. 
	- User can set priority to each thread. Higher priority means higher probability to execute

#### 7.3.2 Thread States
![Thread States](https://i.ibb.co/TTyL5XH/thread-state.png)

#### 7.3.3 Thread Terminate
- **Thread.stop()** (Deprecated)
	- force the thread to stop at once. (Not Thread-Safe)
- **Thread.interrupt()**
	- Set the interrupt bit to 1. If the method is blocked for some reason or manually detecting the interrupt state, it would throw an **InterruptException**. And user can define their behavior when it is thrown. 
- **Set Terminate Flag from Outside**
	- The thread periodicly polls on a terminate flag, which can be set to true by the parent or other thread. 


### 7.4 Multi-Thread 

#### 7.4.1 Multi-Thread Safe
- `synchronoized` block is implemented by `monitorenter` and `monitorexit` instructions
	- If no reference is passed to the `synchronized` block, then the instance or the class would be locked. (based on virtual or class method)
	- `synchronized` block is **Reentrant** to one thread. A thread **cannot** be blocked by itself.  
	- advanced similar class `java.util.concurrent.ReentrantLock` (TODO)

#### 7.4.2 Thread Communication
- wait() and notify()/notifyAll()
	- before calling these methods, it must hold the lock of the callee object. For example, by putting them into a `synchronized` block.
	- wait() and notify() would automatically release the lock when they are executed. Hence, prevent some deadlock scenarios
	- Example code: `waitNotify()` **succeeds** and `waitNotify2()` **fails**
		- That is because the `this` object in `waitNotify2()` is the enclosing `Task` object. However, the `this` object in `waitNotify()` is the `WaitNotifyTest` object.  
		- TODO: the lambda function has special mechanism for the `this` object. 
	```java
	public class WaitNotifyTest {
	    static Object obj;
	    static class Task implements Runnable {
	        @Override
	        public void run() {
	            if (obj == null) {
	                synchronized (this) {
	                    try {
	                        System.out.println("1. wait");
	                        this.wait();
	                    } catch (InterruptedException e) {
	                        e.printStackTrace();
	                    }
	                }
	            }
	            System.out.println("2. consume");
	        }
	    }

	    public void waitNotify2() throws InterruptedException{
	        new Thread(new Task()).start();

	        Thread.sleep(1000);
	        obj = new Object();
	        synchronized (this) {
	            this.notifyAll();
	            System.out.println("3. produce");
	        }

	    }

	    public void waitNotify() throws InterruptedException{
	        new Thread(() -> {
	            if (obj == null) {
	                synchronized (this) {
	                    try {
	                        System.out.println("1. wait");
	                        this.wait();
	                    } catch (InterruptedException e) {
	                        e.printStackTrace();
	                    }
	                }
	            }
	            System.out.println("2. consume");
	        }).start();

	        Thread.sleep(1000);
	        obj = new Object();
	        synchronized (this) {
	            this.notifyAll();
	            System.out.println("3. produce");
	        }

	    }

	    public static void main(String[] args) throws InterruptedException {
	//        new WaitNotifyTest().waitNotify();
	        new WaitNotifyTest().waitNotify2();
	}
	```
- Two Thread **private** variables: **ThreadLocal** and its **Local** variables. 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTMyOTg1MjI0MSwxMTkxNjU0NjA0LDIwOD
E0NTgwOSwtMTYzMzQyNzI5MCwtMjE0OTUzOTIwLDExOTcwMDQz
MDQsMTg0MTg1MjQwNCwtMTMyODI3ODkzNywxNTQ1Mjc3NzIsMT
EyMjQzNjYzOSwtMjA5OTM2NzgxOSwtMTcwNDgwNzMwNyw3NTMw
Mzk4ODMsODgzMDk2ODA1LC0xMzQyNDk2MDI5LC0xODg0OTU3MT
UxLC0xODQ5MTM2ODIsMTY5ODg4NzM4MiwxMzM3NDE1MzIzLC0x
MTY1NTUyOTMyXX0=
-->