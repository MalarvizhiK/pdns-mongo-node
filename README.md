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
 
1. SSH to server VSI:   

2. Run the below commands:   

root@schematics-demo-vsi-server:~# **sudo service mongod restart**    
root@schematics-demo-vsi-server:~# **sudo service status mongod **   
```
You should get the below:
status: unrecognized service
root@schematics-demo-vsi-server:~# sudo systemctl status mongod   
● mongod.service - MongoDB Database Server    
   Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)   
   Active: active (running) since Mon 2020-08-31 05:23:00 UTC; 53s ago  
     Docs: https://docs.mongodb.org/manual  
 Main PID: 13420 (mongod)   
   CGroup: /system.slice/mongod.service  
           └─13420 /usr/bin/mongod --config /etc/mongod.conf   

Aug 31 05:23:00 schematics-demo-vsi-server systemd[1]: Started MongoDB Database Server.
```

3. Connect to mongo db client. Enter the mongo shell by typing mongo.  

root@schematics-demo-vsi-server:~# mongo 

a) Set up your user   

First ssh into your server and enter the mongo shell by typing mongo. For this example, I will set up a user named xxx and give that user read & write access to the user_db database.      
```
use user_db     
 
db.createUser({
    user: 'dbuser',
    pwd: 'dbpassword',
    roles: [{ role: 'readWrite', db:'user_db'}]
})
```
   
b) Enable auth and open MongoDB access up to all IPs   
   
Edit your MongoDB config file. On Ubuntu:   
   
sudo vim /etc/mongod.conf   
   
Look for the net line and comment out the bindIp line under it, which is currently limiting MongoDB connections to localhost:   
   
**Warning:** do not comment out the bindIp line without enabling authorization. Otherwise you will be opening up the whole internet to have full admin access to all mongo databases on your MongoDB server!   

```
# network interfaces   

net:   
  port: 27017  
#  bindIp: 127.0.0.1  <- comment out this line
```

Scroll down to the #security: section and add the following line. Make sure to un-comment the security: line.   

```
security:  
  authorization: 'enabled'  
```

c) Last step: restart mongo daemon (mongod)  
  
> sudo service mongod restart  

If you get the below error:   
```
root@schematics-demo-vsi-server:~# sudo systemctl status mongod
● mongod.service - MongoDB Database Server
   Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)
   Active: failed (Result: exit-code) since Mon 2020-08-31 05:25:34 UTC; 7s ago
     Docs: https://docs.mongodb.org/manual
  Process: 13789 ExecStart=/usr/bin/mongod --config /etc/mongod.conf (code=exited, status=14)
 Main PID: 13789 (code=exited, status=14)

Aug 31 05:25:34 schematics-demo-vsi-server systemd[1]: Started MongoDB Database Server.
```
Then run the 2 commands and restart mongodb service: 

```
root@schematics-demo-vsi-server:~# chown mongodb:mongodb /tmp/mongodb-27017.sock
root@schematics-demo-vsi-server:~# chown -R mongodb:mongodb /var/lib/mongodb
root@schematics-demo-vsi-server:~# sudo service mongod stop
root@schematics-demo-vsi-server:~# sudo service mongod start
root@schematics-demo-vsi-server:~# sudo systemctl status mongod
● mongod.service - MongoDB Database Server
   Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-08-31 05:34:35 UTC; 5s ago
     Docs: https://docs.mongodb.org/manual
 Main PID: 14667 (mongod)
   CGroup: /system.slice/mongod.service
           └─14667 /usr/bin/mongod --config /etc/mongod.conf
```

Now, try to connect to mongodb using the created user, pass and using pdns name. It should be successful.   

```
root@schematics-demo-vsi-server:~# mongo -u malar -p malar malara.testmalar.com/user_db   
MongoDB shell version v4.4.0   
connecting to: mongodb://malara.testmalar.com:27017/user_db?compressors=disabled&gssapiServiceName=mongodb   
Implicit session: session { "id" : UUID("ce35f185-85ef-4d8b-9a7c-03fc995e4e64") }   
MongoDB server version: 4.4.0   
> exit   
```

Also, ensure that port 27017 is open in sever VSI in cloud.ibm.com:

Open port 27017 on your Server VSI. 

    Go to your VPC Gen2 : https://cloud.ibm.com/vpc-ext
    Go to VPC. Select your VPC.  Click on the Security Group. 
    Add a security group rule.   
    Make a new Custom TCP on port 27017, Source: Anywhere, 0.0.0.0/0   
    
 Now, we are in the final step. Lets connect to Client VSI and access the Mongo db:

   
