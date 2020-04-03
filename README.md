# How 'volatile' works on JVM level?

First of all, let's talk about how modern processors work.
According to Intel's manual for software developers, modern processors have one interesing detail that directly affects visibility of data written to RAM:  


<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/Screenshot%202019-09-07%20at%2010.02.38.png" width="700"/>
</p>

**So, stores executed in two phases and after execution phase we can not guarantee that other CPU cores will see updated data**  

But why store buffers were implemented in the first place? Well, modern CPUs have many cores, each core has its own (exclusive) L1 cache and access to multiple shared caches - L2 and L3. Caches are good because they greatly speed up access to data, but at the same time we need to synchronized data in all caches so CPU cores won't see stall data. 
That's where MESI protocol is used for.

<details>
  <summary>Protocol MESI explained in details</summary>

- Let's assume:
  - All cpu cores connected via single bus
  - Each core has only single level of cache
  - Each core watches for messages concerned with data addresses which it has cached
  - CPU wanting to write grabs bus cycle and broadcasts new data as it updates its own copy
  - Problem of simultaneous writes is taken care of by bus arbitration - only one CPU can use the bus at any one time
  
- Any cache line can be in one of 4 states (2 bits):
  - Modified - cache line has been modified, is different from main memory - is the only cached copy
  - Exclusive - cache line is the same as main memory and is the only cached copy
  - Shared - same as main memory but copies may exist in other caches
  - Invalid - line data is not valid (as in simple cache)
  
- Operation can be described informally by looking at action in local processor
  – Read Hit
  – Read Miss
  – Write Hit
  – Write Miss

#### MESI Events:

- Local Read Hit:
  - Line must be in one of MES
  - This must be correct local value (if M it must have been modified locally)
  - Simply return value
  - No state change
  
- Local Read Miss:  
  - No other copy in caches
    - Processor makes bus request to memory
    - Value read to local cache, marked E
  - One cache has E copy
    - Processor makes bus request to memory
    - Snooping cache puts copy value on the bus
    - Memory access is abandoned
    - Local processor caches value
    - Both lines set to S
  - Several caches have S copy
    - Processor makes bus request to memory
    - One cache puts copy value on the bus (arbitrated)
    - Memory access is abandoned
    - Local processor caches value
    - Local copy set to S
    - Other copies remain S
  - One cache has M copy
    - Processor makes bus request to memory
    - Snooping cache puts copy value on the bus
    - Memory access is abandoned
    - Local processor caches value
    - Local copy tagged S
    - Source (M) value copied back to memory
    - Source value M -> S
    
 - Local Write Hit:
    - Line must be one of MES:
      - M:
        - Line is exclusive and already ‘dirty’
        - Update local cache value
        - No state change
      - E:
        - Update local cache value
        - State E -> M
      - S:
        - Processor broadcasts an invalidate on bus
        - Snooping processors with S copy change S -> I
        - Local cache value is updated
        - Local state change S -> M

 - Local Write Miss:
    - Detailed action depends on copies in other processors:
      - No other copies:
        * Value read from memory to local cache
        * Value updated
        * Local copy state set to M
      - Other copies, either one in state E or more in state S:
        * Value read from memory to local cache - bus transaction marked RWITM (read with intent to modify)
        * Snooping processors see this and set their copy state to I
        * Local copy updated & state set to M
      - Another copy in state M:
        * Processor issues bus transaction marked RWITM
        * Snooping processor sees this
          * Blocks RWITM request
          * Takes control of bus
          * Writes back its copy to memory
          * Sets its copy state to I
      - Another copy in state M (continued)
        * Original local processor re-issues RWITM request
        * Is now simple no-copy case
          * Value read from memory to local cache 
          * Local copy value updated
          * Local copy state set to M
</details>  

Well, as you can see, there are a lot of things must happen to write single value to RAM, but do we really need it? What if cpu core wants to update variable many times and only last value must be promoted to RAM? Store buffers were designed exactly for this situation.
But we have JLS, JMM and all other stuff that tells us about happens-before, volatile variables behavior. How can we be sure that all visibility rules are strictly followed under all circumstances?

**Let's dig deep into the Hotspot sources to find out how it works!**

But we need to know where to start our journey, right?
I think we should start with some 'hello world' using volatile variables and see how it will be compiled into bytecode:  

<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/1.PNG" width="900"/>
</p>

So, ordinary **putfield** opcode, right?  
No signs of synchronization whatsoever.  
Then we need to find out how this opcode is translated by Hotspot interpreter.  
All opcodes and corresponding generator functions described in **templateTable**:  

<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/2.PNG" width="800"/>
</p>

There is a huge table with all opcodes listed:  

<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/3.PNG" width="900"/>
</p>

Here is our **putfield** opcode!  

<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/4.PNG" width="900"/>
</p>

Actually, static and non-static fields are written using the same method **putfield_or_static**:  

<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/5.PNG" width="700"/>
</p>

Here is method **putfield_or_static** itself. As you can see, in line 3131 we test if target field is volatile and then continue execution through lines **3134** and **3135** if answer is yes, or jump to label **nonVolatile** and go through line **3140**:

<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/6.PNG" width="700"/>
</p>

In line **3135** from previous screenshot we see that function **volatile_barrier** is called:    
<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/7.PNG" width="700"/>
</p>

And then, finally, we found our **membar** function that emits magic assembly code that ensures adherence to Java specifications:  
<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/8.PNG" width="700"/>
</p>

This function emits string like **lock addl [rsp + #0]**. But why it works? Turns out, according to the same Intel specs, locked instructions provide total order:
<p align="center">
  <img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/Screenshot%202019-09-07%20at%2019.55.42.png" width="700"/>
</p>

So, write to **volatile** field compiles to **putfield** opcode, then JVM interpreter translates this opcode to assembly snippet that contains instruction **lock addl [rsp + #0]** which, according to Intel specs, provides total order by draining store buffer to L1 cache, whose content is synchronized with other core's caches, shared caches, and RAM, by MESI protocol.
