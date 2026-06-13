At memory allocation, every task gets assigned a spot in the heap ([[stack]], [[TCB]])
### Definitions
Period: When will the instruction will execute again.
Execution time.
Deadline: Max task finish time.
Release time: when is the task ready to perform.
Preemption: When task takes over the process because of higher priority. At the end of execution time, processor returns to a lower priority previous task.

### States
![[Pasted image 20230813200618.png|500]]|

### Communication
Uses the [[Queue]] to for inter-task communication.

### Bugs 
[[Deadlock]]
[[Starvation]]
[[Priority Inversion]]