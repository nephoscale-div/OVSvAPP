### vCenter OVSvAPP VM allocation howto

1. Browse vCenter location: *Home / Inventory / Hosts and Clusters*

2. Right-click on host item in sidebar tree, on which you want to allocate VM

3. Click on *New Virtual Machine...* menu item

![Figure_01](img/vcenter-vm-howto/figure_01.png)

4. In appeared dialogue choose *Typical* VM configuration

![Figure_02](img/vcenter-vm-howto/figure_02.png)

5. Set OVSvAPP VM name

![Figure_03](img/vcenter-vm-howto/figure_03.png)

6. Choose VM data storage

![Figure_04](img/vcenter-vm-howto/figure_04.png)

7. Configure guest operating system

![Figure_05](img/vcenter-vm-howto/figure_05.png)

8. Configure VM network:

    8.1. Set 3 NICs

    8.2. Set NIC 1 connected to *trunk port* of *internal DVS*

    8.3. Set NIC 2 connected to *trunk port* of *external DVS*

    8.4. Set NIC 3 connected to *management network*

![Figure_06](img/vcenter-vm-howto/figure_06.png)

9. Configure VM disk

![Figure_07](img/vcenter-vm-howto/figure_07.png)

10. Finish VM allocation and proceed with OS installation.

![Figure_08](img/vcenter-vm-howto/figure_08.png)
