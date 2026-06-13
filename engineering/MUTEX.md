Mutual Exclusion.

Works like a [[Lock]].
Asssures that a critical section is not tampered. 
Protects shared resources and synchronizes threads.
Once a task gets a hold of the MUTEX, it can work without chance of tampering
If another task wants to access the shared resources, it goes to a blocked state.
Reading and writing a MUTEX is an [[atomic_operations]].