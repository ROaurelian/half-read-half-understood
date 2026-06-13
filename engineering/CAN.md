Controller Area Network: Serial communication protocol designed for automotive and industrial applications.
Describes the first two layers of the [[OSI]] model.
Uses two twisted wires (CAN_HIGH and CAN_LOW) that connect in parallel two 120$\Omega$ resistors. 
Resistors avoid signal reflection.
A full twist/inch maintains a characteristic impedance similar to the resistors. 
The voltage on one cable is inverted in respect to its pair.
Devices must be connected <30cm form CAN BUS.
Data is an expression of the difference between the signals.

![[Pasted image 20230714130859.png|500]]
1 = Recessive = 0V dif
0 = Dominant = 2V dif 

Every device connected is always a listener, even while sending data.
Every message has an 11 bit identifier (CAN 2.0A).
Arbitration (priority in message sending) is settled for the lowest identifier currently sending.
An acknowledgement bit is always required in a message, meaning that there must be at least two devices connected to the BUS.
In order to keep synchronization, CAN uses [[bit stuffing]]. 

### Pros
- By design ignores outside electrical noise.
- Doesn't produce noise (differential signal cancels out). 
- Standardized.

### Devices

[[PDM]]
[[ECU]]

### Troubleshooting
In normal conditions, avg voltage measured in CAN_HIGH must be always higher than CAN_LOW (f.e. 2.6V > 2.45V). If different, a device has been connected backwards.
Avg voltage depends on the rate that the msg is sent.
When measuring resistance on a terminal in the BUS, it should be around 60$\Omega$ (two 120$\Omega$ in parallel). Any deviation means that a termination resistor is not connected properly. 
If either CAN_H or CAN_L are shorted, both will be affected. If you detect this error, disconnect the termination resistors, determine which cable is shorted and start probing each device connected to determine which has been shorted.

#### Tips
1. Break the network into subnetworks.
2. Even if you see communication in the network (measured by an oscilloscope or multimeter), that doesn't mean that a module is defective.
3. Either a wire or a module must be the issue.

### Topology
- Ring: Usually no used.
- Star: Star CAN BUS (all wires connect to 2 terminals).
![[Pasted image 20230809112511.png|500]]
- Bus: Can be in parallel (star) or serial. For serial troubleshooting, jump cables to test a defective module.


See also: [[CAN-FD]]