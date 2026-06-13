When none of the tasks execute, they return to a standstill.
Caused by bad administration of [[MUTEX]] and [[semaphores]].
To avoid this bug:
- Include a default behavior for a timeout (f.e. return a [[MUTEX]]).

A variant of this bug is livelock, when the tasks accidentally synchronize their default behaviors and start going in a loop of taking and returning a [[MUTEX]] (f.e.).
To debug livelock:
- Assign hierarchy to each [[MUTEX]] and include some behavior that uses it.
- Include an arbitrator ([[MUTEX]])
