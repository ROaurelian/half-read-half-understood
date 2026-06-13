A Real Time Operating System provides part of the same functionality as a [[GPOS]] but assures timing deadlines.
Generally used in embedded systems for concurrent multitasking.

### Definitions
1. [[Task]]: Set of instructions
2. Thread: Unit of CPU utilization with its own stack

### Types
- Hard RTOS: Guarantees that the task will be executed on the deadline.
- Soft RTOS: Most of the time will meet the deadline with some uncertainty.

### Memory allocation
In [[FreeRTOS]] there are 5 ways to manage [[heap]] memory:
1. Declare all memory as static (heap_1)
2. Obsolete (heap_2)
3. Unites fragmented parts of the heap into a block (heap_4)
4. Auto free (heap_3) #notsure
5. Unites non-fragmented parts of memory into a block (heap_5)