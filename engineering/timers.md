Uses [[clock]] to generate a periodic signal.
For precise generation, use an external hardware clock.
For most applications use the internal clock signal from the [[MCU]].
Used to periodically call a function or peripheral without the [[CPU]].

![[pasted_image_20230815080059.png|600]]

Task A and B provide commands to the queue, which then in turn go to the Timer Service Task (daemon). 
The daemon is in charge of selecting a timer and, when the timer runs out or matches a certain value, run a specific function with a callback.