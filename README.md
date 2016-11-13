# How To Install ELK In LXD Containers

### Requirement 

 LXD Setup
 
### Installation

Create ELK network (Optional)   

    lxc network create testbr0 ipv6.address=none ipv4.address=10.0.3.1/24 ipv4.nat=true
