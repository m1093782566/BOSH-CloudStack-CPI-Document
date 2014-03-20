#Setting Up your IaaS for BOSH

##Choosing Suitable Hardware

###CloudStack

CloudStack describes hardware in terms of service offering, which includes CPU speed, number of CPUs, RAM size, tags on the root disk, and other choices. For more information on how to define your service offerings in CloudStack, see Section 4 of the [CloudStack Administration Guide](http://download.cloud.com/releases/2.2.0/CloudStack2.2.4AdminGuide.pdf).

##Restricting Inbound Network Traffic
Configuring Security Groups is required to deploy BOSH on AWS and OpenStack. Configuring multiple networks is recommended for vSphere. CloudStack supports both Security Groups in basic zone and networks in advanced zone. You do this to restrict incoming network traffic to the BOSH VM or VMs.

###CloudStack
In basic zone, you should configure Security Groups like AWS and OpenStack, and configure networks like vSphere when in advanced zone. BOSH CloudStack CPI support both zone types and add an option to switch between them.

##Validating Your IaaS Instance
There are at least two aspects of your IaaS instance that you can validate: hardware, and functionality (for example, communication between VMs).
There is also the question of how far you can or want to go towards automating validation.
If you are using OpenStack, you can run the validation procedure which appears in the Cloud Foundry documentation but applies to the present context as well.
If you are using AWS, vSphere or CloudStack, you can consider adapting the parts of the OpenStack procedure that make sense for your IaaS.
You can use the appropriate APIs for performing and automating validation:
* VMware vSphere API
* Amazon EC2 API
* OpenStack APIs
* [CloudStack APIs](https://cloudstack.apache.org/docs/api/)

