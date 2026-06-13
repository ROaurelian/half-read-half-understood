When shared resources are occupied only by a higher priority task.
Lower priority tasks don't get to run.
Its debugged by:
- Assigning a non blocking delay to higher priority tasks.
- Making higher priority tasks only run by [[Semaphores]] or interrupts. 
- Aging: Gradually increment priority of a starved task in each cycle that doesn't run. At the same priority with the originally higher priority task, our starved task will perform a certain computing and the return to default when it finishes.