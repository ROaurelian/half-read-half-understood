Protects shared resources and synchronizes threads.
Consists of a counter and a buffer. 
The counter functions like a flag.
The counter signals certain tasks (consumers) that the buffer is ready to read.
Other tasks (producers) write data on the buffer.
Consumers decrement and producers increment the counter.

