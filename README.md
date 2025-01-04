# openshift-installation-offline-for-RAN

# **Descriptions**
This repository is to specify how to offline install openshift due to some network restriction scenarios.


# **Prerequistes**
## **Hardware**
a. HPE/Dell/Supermicro/Nvidia server as target server which need install openshift
b. HPE/Dell server as image registry server
c. Jumper server which is used to execute the commands to configure and install openshift for target server
d. DNS server (optional)
e. NTP server (optional)
note: if possible, DNS,NTP and image registry can be in same one hardware server

## **software tool**
a. docker image registry.
b. DNS server
c. Chronyd NTP server

