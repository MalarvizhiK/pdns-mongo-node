# pdns-mongo-node
Example to show how to create a private domain name service and accessing mongo db using pdns name from a Nodejs client VSI.   

For more details on pdns, please refer : https://github.ibm.com/sakshan1/docs/blob/master/getting-started/ibmcloud-node-mongo-dns.md

Run the command **terraform apply** and create resources for pdns. 

Once the resources are created, you need to follow this link to update the dns resolver on your server and client VSI: https://cloud.ibm.com/docs/dns-svcs?topic=dns-svcs-updating-dns-resolver

The steps has to be followed in all the VSI where you want to access the DNS name : **malara.testmalar.com**.

Otherwise, you will get the below error:   

**curl: (6) Could not resolve host: malara.testmalar.com.**    

Since, we have used ubuntu 18_04 image for server and client VSI, we need to follow the steps listed under: **VPC with generation 2 compute instances** for your operation system. I followed the steps for **Configuring Ubuntu Linux 18.04 LTS Bionic Beaver**

1. Ensure your version of netplan is 0.95 or later:  

dpkg -l |grep netplan  
ii  netplan.io                      0.98-0ubuntu1~18.04.1               amd64        YAML network configuration abstraction   for various backends

If your netplan version is earlier than 0.95, upgrade netplan:  

apt-get update
apt-get upgrade

FYI: Ubuntu 18.04 has a known issue https://askubuntu.com/questions/1120998/error-in-network-definition-unknown-key-dhcp4-overrides   

dhcp4-overrides requires netplan 0.95 or later, which is not yet available in Ubuntu 18.04. See https://bugs.launchpad.net/netplan/+bug/1759014 for the status of this stable update.   
      
apt-get update  
apt-get upgrade   

2. Create the file /etc/netplan/99-custom-dns.yaml with the following content:  
```
network:   
    version: 2   
    ethernets:  
        ens3:  
            nameservers:  
                addresses: [ "161.26.0.7", "161.26.0.8" ]   
```

> **NOTE:**  dhcp4-overrides will not work in ubuntu 18.04, hence removed the line. 
            
 3. Apply the changes to netplan: 

netplan apply               

4. Open the /etc/dhcp/dhclient.conf file and add the line:  
      
supersede domain-name-servers 161.26.0.7, 161.26.0.8;   

5. Run the following command to release the current lease and stop the running DHCP client, then restart the DHCP client. Doing this ensures the statically configured DNS servers take precedence globally.     

    dhclient -v -r; dhclient -v  
  
If you are still unable to resolve with the new configuration, flush all DNS resource record caches the systemd service maintains locally, and try again.         

systemd-resolve --flush-caches   

6. 



** Mongo db installation:  **

https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/   

https://ianlondon.github.io/blog/mongodb-auth/  




