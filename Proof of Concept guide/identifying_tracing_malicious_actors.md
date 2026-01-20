# Tracing and Isolating High-Risk Entities

Each device that connects to the Ztrust network is assigned a unique IP address that remains fixed until the device is completely removed from the network. This means that even if a device connects to or disconnects from the Ztrust network multiple times, its Ztrust IP address remains unchanged.

The system aggregates connected devices' IP addresses and their connection logs to construct a comprehensive corporate Netmap (access map).
![img.png](img.png)


In the event of an incident, the system leverages assigned static IP addresses and user accounts to swiftly identify suspicious actors.
![img_1.png](img_1.png)
Mapping IP addresses to users and devices:
![img_2.png](img_2.png)
If a device attempts to alter its assigned IP address, it will be immediately disconnected from the Ztrust network and will no longer be able to access any resources.
