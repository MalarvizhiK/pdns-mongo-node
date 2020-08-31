# pdns-mongo-node
Example to show how to create a private domain name service and accessing mongo db using pdns name from a Nodejs client VSI.   

For more details on pdns, please refer : https://github.ibm.com/sakshan1/docs/blob/master/getting-started/ibmcloud-node-mongo-dns.md

Run the command **terraform apply** and create resources for pdns. 

Once the resources are created, you need to follow this link to update the dns resolver on your server and client VSI: https://cloud.ibm.com/docs/dns-svcs?topic=dns-svcs-updating-dns-resolver

The steps has to be followed in all the VSI where you want to access the DNS name : **malara.testmalar.com**.

Otherwise, you will get the below error:   

**curl: (6) Could not resolve host: malara.testmalar.com.**    

Since, we have used ubuntu 18_04 image for server and client VSI, we need to follow the steps listed under: **VPC with generation 2 compute instances**


** Mongo db installation:  **

https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/   

https://ianlondon.github.io/blog/mongodb-auth/  




