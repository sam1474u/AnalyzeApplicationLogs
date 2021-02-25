# AnalyzeApplicationLogs



### 1 - Introduction
Observability (O11y)  is key for cloud-native applications. As mentioned by Clay Magouyrk, EVP Oracle Cloud Infrastructure  in his <a href="https://www.oracle.com/events/live/multicloud-observability-and-management/">webcast</a> on October 6th 2020, the OCI platform has just added new powerful features and tools in this area - as Oracle OCI Observability Platform.

The term observability is usually comprised of three main areas: metrics, logging and tracing. In this post I will focus on (central) logging - and will describe as an example how easy it is to add your Kubernetes container logs (or any other custom log file) to OCI Logging.

<a href="https://docs.cloud.oracle.com/en-us/iaas/Content/Logging/Concepts/loggingoverview.htm">OCI Logging</a> is a new central component for analyzing and searching log file entries for tenancies in Oracle Cloud Infrastructure (OCI). It uses Fluentd under the hoods to push log files to a central log store where it it indexed for easier and faster searching. There are many services which have predefined agents and can be easily enabled to publish logs - like Functions. The ability to add custom logs from any compute instance makes it even more flexible to use this with other components like Oracle Kubernetes Engine (OKE) as well.

![image](https://user-images.githubusercontent.com/42166489/109104169-87d6f500-7751-11eb-8a3d-779a76e74d6a.png)


### 2 - Setting up the OKE Kubernetes Cluster
**prerequisite**: Setup <a href="https://github.com/sam1474u/Deploy-Helidon-Based-Application-in-Kubernetes-Cluster">Helidon App</a>

For publishing the custom logs from the OKE worker nodes (or any other compute instance), it is required to have the Oracle Unified Monitoring Agent installed. This agent is by default included in the newer Oracle Linux images.

If you create a Node Pool for OKE based on one of these newer images - for example "Oracle-Linux-7.8-2020.08.26-0" or newer, then you will not need to install the agent manually.

See instructions in the <a href="OCI Logging Documentation">OCI Logging Documentation</a> on how to install and verify the agent. You can check if the agent is installed and running by executing this via ssh on one of the worker nodes:  

    systemctl status unified-monitoring-agent
    
### 3 - Setting up OCI Logging for Custom Logs

I assume you have an existing Kubernetes cluster available on OKE - otherwise just go ahead and deploy a cluster using the quickstart option.

First create a new Dynamic Group (or a User Group), this is to identify the hosts (compute instances) where logs should be collected:

You can for example create a Dynamic Group with this rule to identify all instances in one compartment (of your Kubernetes cluster). This will include the worker node instances:

All {instance.compartment.id = '<your-compartment-ocid>'} 
![image](https://user-images.githubusercontent.com/42166489/109101841-89062300-774d-11eb-9b97-c733e10d9655.png)
    
Create a policy with dynaic group to use log content

![image](https://user-images.githubusercontent.com/42166489/109102249-70e2d380-774e-11eb-99c6-ec0889e278ba.png)



Then create a new log group:
Under "Logging - Log Groups" create a new group named "OKE-Custom-Log-Group".

The next step is to define a new Custom Log and Agent Configuration:

In step 1 the custom log is created: 
<br/>
![image](https://user-images.githubusercontent.com/42166489/109101633-19903380-774d-11eb-9842-4cf0c6df98b2.png)
![image](https://user-images.githubusercontent.com/42166489/109101637-1bf28d80-774d-11eb-8f20-966efe6ee40c.png)
![image](https://user-images.githubusercontent.com/42166489/109101646-1f861480-774d-11eb-8776-d9d982c2eac9.png)



In step 2, you create the Agent Configuration:

![image](https://user-images.githubusercontent.com/42166489/109101649-214fd800-774d-11eb-9601-2b85d238bdb8.png)




Select your created Dynamic Group, and enter "Log Path" using "/var/log/containers/*.log"  (without the quotes). 
![image](https://user-images.githubusercontent.com/42166489/109101656-23b23200-774d-11eb-8730-3905ff4abc9b.png)

![image](https://user-images.githubusercontent.com/42166489/109101660-26148c00-774d-11eb-9e8f-66cb6bfe1013.png)


You do not need to enter any advanced parser option for now.

Then submit the form and wait until the Agent Configuration shows status "Active".



### 4 - Deploying a sample cloud-native application 
I have been using an app similar to the Helidon MP quickstart example - or you can try the Helidon MP sample described on Todd Sharp's blog "Building And Deploying A Helidon Microservice With Hibernate"

In the resource java class, I have added this logging code using java.util.logging - which is called everytime a GET is called on the resource:

    import java.util.logging.Logger;

    final Logger logger = Logger.getLogger(HelloResource.class.getName());

    logger.info("Executing method getDefaultMessage() for GET in HelloResource");

    The logger configuration for Helidon MP is defined in my logging.properties:

**Send messages to the console**

    handlers=java.util.logging.ConsoleHandler

**Global default logging level. Can be overriden by specific handlers and loggers**

    .level=INFO

**Helidon Web Server has a custom log formatter that extends SimpleFormatter.**

**It replaces "!thread!" with the current thread name**

    java.util.logging.ConsoleHandler.level=INFO

    java.util.logging.ConsoleHandler.formatter=io.helidon.webserver.WebServerLogFormatter

    java.util.logging.SimpleFormatter.format=%1$tY.%1$tm.%1$td %1$tH:%1$tM:%1$tS %4$s %3$s !thread!: %5$s%6$s%n 

Just go ahead and deploy this example or any other app exposing a simple REST endpoint and emitting some log entries  to OKE using kubectl or terraform - and then execute some requests.

### 5 - Monitoring and Searching Log Entries
The value of the central OCI Observability framework is that you do not need to access each cluster using "kubectl logs" - but that you have all log entries of all your components available in a Single Pane of Glass" via the OCI Console.

In OCI Console - under "Logging - Search" you can use two modes to search for log entries - basic and advanced mode.

If you navigate to your Log Group, you can see on overview of all events during the specified time frame. To narrows down the number of events, select only the Log Group you have created:



With the Advanced Mode option, you can filter or search using more complex expressions.

This is an example for the standard expression searching within one Log Group:

    search "ocid1.compartment.oc1..yourid1/ocid1.loggroup.oc1.eu-frankfurt-1.yourid2" | sort by datetime desc

And here is an example for searching using a specific search string in the expression:

    search "ocid1.compartment.oc1..yourid1/ocid1.loggroup.oc1.eu-frankfurt-1.yourid2" | where logContent = '*getDefaultMessage()*' | sort by datetime desc



This is the detail view of the log entry:

![image](https://user-images.githubusercontent.com/42166489/109101679-3462a800-774d-11eb-9077-0df51ac3cbbd.png)


You can optimize the parsing result structure by using the appropriate Fluentd parser options.

In the overview page of the Log Group you can see all log events within a certain timeframe:



In the next post I will describe how to setup and use OCI Logging Analytics - which gives you even more ways to analyze logging information across OCI tenancies, to do trend analysis and enable proactive monitoring.

Happy analyzing of your logs!


