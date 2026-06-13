## Bounded priority inversion
![[pasted_image_20230815105856.png]]

## Unbounded priority inversion
![[pasted_image_20230815110059.png]]

## Debugging
- Priority ceiling: When taking a [[MUTEX]], a task is dynamically assigned the highest priority. At the returning, the task returns to default priority. 
- Priority inheritance: When taking a [[MUTEX]], the task retains the same priority. If another task attempts to retrieve the [[MUTEX]], the owner task is assigned the same priority as the demander. 