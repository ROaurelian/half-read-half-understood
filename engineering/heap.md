Used for allocating data that need to persist beyond the scope of a block.
Uses `new` keyword to auto allocate memory with assistance by the [[OS]].
The `new` [[operator]] also calls the constructor + the `malloc()` function.

### Declaration

`int* a = new int //4 bytes`

### Pros
- Bigger allocation in [[RAM]] than the [[stack]].
- Longer lifetime.

### Cons
- Consumes resources for "bookkeeping" and allocating.
- Must be manually deallocated with the `delete` operator. 
