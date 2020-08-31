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

The above steps needs to be performed in both Server and Client VSI. Restart Server and Client VSI by Stopping and Starting VSI in cloud.ibm.com.  

1. SSH to Server VSI :

ssh root@<floating IP> 
      
2. ping malara.testmalar.com : You should be able to ping using pdns name. You should get response from private ip address of VSI Server.   

root@schematics-demo-vsi-server:~# ping malara.testmalar.com  
PING malara.testmalar.com (10.240.0.5) 56(84) bytes of data.  
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=1 ttl=64 time=0.015 ms  
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=2 ttl=64 time=0.060 ms  
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=3 ttl=64 time=0.037 ms    
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=4 ttl=64 time=0.050 ms  
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=5 ttl=64 time=0.035 ms  
^C      

Similarly, try the pdns name in Client VSI:   

1. SSH to Client VSI :  
  
ssh root@<floating IP>   
      
2. ping malara.testmalar.com : You should be able to ping using pdns name. You should get response from private ip address of VSI Server.      

root@schematics-demo-vsi-server:~# ping malara.testmalar.com  
PING malara.testmalar.com (10.240.0.5) 56(84) bytes of data.  
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=1 ttl=64 time=0.015 ms  
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=2 ttl=64 time=0.060 ms  
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=3 ttl=64 time=0.037 ms    
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=4 ttl=64 time=0.050 ms  
64 bytes from schematics-demo-vsi-server (10.240.0.5): icmp_seq=5 ttl=64 time=0.035 ms  
^C      

Now, lets install Mongodb in Server VSI. 

1. SSH to Server VSI :  
  
ssh root@<floating IP>  

2. Install Mongo db by following the steps below:  

** Mongo db installation:  **

Follow this link https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/   
Section: **Install MongoDB Community Edition**       


1. Import the public key used by the package management system.  

From a terminal, issue the following command to import the MongoDB public GPG Key from https://www.mongodb.org/static/pgp/server-4.4.asc:   

> wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -  

2. The operation should respond with an OK.   

However, if you receive an error indicating that gnupg is not installed, you can:   

    Install gnupg and its required libraries using the following command:   
    
    sudo apt-get install gnupg 
    
 3. Once installed, retry importing the key:   
 
 wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -   
 
 4. Create a list file for MongoDB: 

Create the list file /etc/apt/sources.list.d/mongodb-org-4.4.list for your version of Ubuntu by running the below command.

echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list  

5. Reload local package database.   

Issue the following command to reload the local package database:  

sudo apt-get update  

6. Install the MongoDB packages.   

 sudo apt-get install -y mongodb-org 
 
 Now, Mongo db is installed. 


In order to connect to mongo db from other VSI, you need to provide a user and password and open the port 27017. 

Follow the steps from this link: https://ianlondon.github.io/blog/mongodb-auth/
 



Also, ensure that port 27017 is open in sever VSI in cloud.ibm.com:

Open port 27017 on your EC2 instance  

    Go to your EC2 dashboard: https://console.aws.amazon.com/ec2/   
    Go to Instances and scroll down to see your instance’s Security Groups. Eg, it will be something like launch-wizard-4  
    Go to Netword & Security -> Security Groups -> Inbound tab -> Edit button.  
    Make a new Custom TCP on port 27017, Source: Anywhere, 0.0.0.0/0   
