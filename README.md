## Protocol MESI explained:

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

## x86_64 memory barriers:
   - LFENCE (Load Fence / combines LoadLoad & LoadStore barriers):
     - Prevents reordering of reads with subsequent reads and writes
     - Stalls the execution of all younger loads until the older ones (and the fence itself)  
       have finished and committed. This will affect performance by serializing the loads,  
       but would not otherwise protect you against any operation in other cores
       
   - SFENCE (Store Fence / StoreStore barrier):
     - **lock xchg**
     - Prevents reordering of writes
     - Flushes the store buffer as they won't allow pending speculative stores to remain  
       (that's why they're fencing). Once they changes are in L1 they're already observable  
       by anyone, they don't have to be flushed anywhere further away
       
   - MFENCE (Full Fence / LoadStore & StoreStore & LoadLoad & StoreLoad barriers):
     - **lock addl 0,(sp)**
     - Full memory fence for all operations on all memory types, whether non-temporal or not
