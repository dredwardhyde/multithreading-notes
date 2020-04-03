# How 'volatile' works on hardware level?

First of all, let's talk about how modern processors work.
According to Intel's manual for software developers, modern processors have one interesing detail that directly affects visibility of data written to RAM:  


<img src="https://raw.githubusercontent.com/dredwardhyde/multithreading-notes/master/Screenshot%202019-09-07%20at%2010.02.38.png" width="700"/>

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

Well, as you can see, there are a log of things must take place to write single value to RAM, but do we really need it? What if cpu core wants to write value many times and only last one should be promoted to RAM? Store buffers were designed exactly for this situation.
But 
