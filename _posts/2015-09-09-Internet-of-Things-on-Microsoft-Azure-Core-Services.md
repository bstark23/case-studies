---
layout: post
title: Internet of Things on Microsoft Azure Core Services
author: Ben Hanson
author-link: http://benjaminstarkhanson.com
date: 2015-09-09 02:26:11
categories: Azure,IoT,Auto Scaling
color: blue
excerpt: A demonstration on how to handle large scale IoT data injestion and processing using Microsoft Azure's Core Services and Queue-Length based auto-scaling.
---
Internet of Things on Microsoft Azure Core Services
===================================================


Let’s say we are creating a system of internet connected devices embedded in the ground all over the world constantly recording data like ground temperature, air temperature, moisture levels – basically whatever sensors you can pack into the device without consuming too much power – and sending it to the cloud to be processed later.

For this example I’m going to use Azure’s most basic core services – Table Storage, Queues, Queue-length based auto-scaling Cloud Services and Scheduler.

Device Assumptions
------------------

* It has some way of permanently storing a unique device id
* It has a mechanism for constantly keeping track of the time in UTC
* It has a connection to the internet


Technical Overview
------------------

On the device side we’re going to need some very simple, constantly looping logic:

1. Read all sensor data and the current timestamp in UTC time
2. Send data to the cloud with the device id and timestamp
3. Wait for some period of time depending on how often you want to sample
4. Repeat

The back-end processing model is going to be basically:

1. For each device
2. Grab all data samples from the last time processing ran
3. Process the batch
4. Store all the various outputs of your job to their respective locations
5. Update the last time processed for the device with the sample’s timestamp

Implementation
--------------

###Data Ingestion

For ingesting the data samples from the device Azure Storage Tables can be a very efficient and cost effective way to store huge sets of data – it’s also been around for a while, is very simple, secure, reliable and can be called from virtually every major programming language.

The only problem is that it’s entirely locked down from the outside world unless you embed the primary or secondary key for the entire storage account into each device (very bad idea – please don’t do that) or – use shared access signatures.

One way to do this in a very cost effective, secure and efficient manner is to setup a very simple API that’s only accessible over SSL with an endpoint for creating a shared access signature for each device.  The device, upon powering on, should call this service and retrieve a write-only shared access signature to the sensor data table.  The server should store off this signature such that it can revoke it at any time should the need arise (e.g. compromised or malfunctioning device).

The device then stores off this signature in encrypted memory and uses it to continuously make write-only calls directly to table storage without the need to go through any intermediary or use virtually any of your compute resources.

###Table Setup Strategy

The single most important thing you can do when starting a project like this is to pick the right strategy for what you use as the table’s PartitionKey and it’s RowKey.  While an extremely efficient and cost effective place to store data like this, querying support is extremely limited (you can only query based on PartitionKey and a RowKey range) and if you don’t know what you’re doing you could run into a ton of headaches and terrible overall performance.

In this example it would make a lot of sense to use a unique Id embedded into each device as the PartitionKey as this makes it easy to query all messages from an individual device – if you want all messages/rows from a given device simply ‘get by partition key’ and you’ll have them relatively quickly, but that performance is dependent on the number of rows on the table so as you gather more and more data, the process for grabbing all of a device’s messages gets progressively longer and longer – which means there’s a scalability limit there somewhere.

This is where your RowKey strategy becomes important.  In this scenario you’re very rarely, if ever, going to actually want to grab an entire table’s partition at once every time you’re processing something but instead just want to process everything since the last entry you processed on a previous run.  A great way to do this is to store either the current time in UTC in ticks or the maximum value of the time value on the sensor – the current time in ticks, depending on whether or not you want the data in ascending or descending order.

There’s a great StackOverflow post on even more interesting scenarios that’s worth a read when considering a strategy for your specific use-case.

For this example we’ll just go with the UTC value in ticks as our RowKey.

Your sensor data table schema is then going to look like this:

* PartitionKey => DeviceId
* RowKey => UTC Timestamp in ticks
* {SensorName} => {Sensor Value} (repeat for all sensor data you’re collecting)

Now comes the question of where do you store off the list of partition keys, the last read row key and whatever other information you may want to store (such as the time it took to process the last batch).

Ideally, as part of your manufacturing process (or whenever you go to deploy a new device) you would already have a separate table with this information in it.  Now comes the question of how to design that table’s structure.  This is what I call an Azure Partition Table (note: this is my own terminology and not that of Microsoft’s).

###Azure Partition Tables

What I call an Azure Partition Table is a table for keeping track of all of the partitions of a separate table along with whatever metadata you want to store about that partition.  This brings up an interesting scenario for how to properly select partition and row keys.

For the row key there is at least one obvious choice – use the partition key of the table you’re tracking (the sensor data table for us) as the row key.  But what about the partition key?

For the partition key, you could just use some constant value and do a ‘get by partition key of {constant value}’ every time you’re trying to get the list of devices to process.  The problem with this approach is that you’re inevitably going to run into the same scenario as before, where the process to grab the information about each partition in our main table takes progressively more time as you add more devices.

Assuming that we’re going to constantly be deploying more and more of these devices all over the globe over time a time-based partition key sounds like a reasonable choice.  This could be just the year the device was deployed all the way down to the granularity of ‘year-month-day’ and so on.  The level of granularity has a direct relation to the performance of the entire system and should be chosen based on how frequently you plan on deploying the devices along with what you think the rate you deploy monthly is going to change.  The reasons for this will become clear in the following section on data processing.

Your partition table schema is then going to look something like this:

* PartitionKey => e.g. ‘year-month’
* RowKey => PartitionKey of the main table (sensor data table)
* LastReadRowKey
* {Whatever other metadata you want to capture about the last processing session)

Now to process all this data.

###Data Processing

For processing all of this data in real-time we’re going to need:

* A Cloud Service that scales according to queue-length
* A ‘master job’ that runs at some set interval and queues off processing for each month since your first device was deployed (a constant value)
* A per-time period job that queues up a job for each device deployed during that time-period.
* The device processing job that actually does whatever processing you want to do with the sensor data for that device since the last time it was processed.

All of this can be done with just a handful lines of code in an extremely simplistic, efficient and scalable manner.

###Cloud Service

The cloud service is going to be a very basic infinite loop that looks something like the following – note: ExtraSmall instances should be completely fine for this, which are ridiculously cheap:

1. Get the next message off the queue (if there is one)
2. Execute the code for the job that’s associated with that message
3. Repeat

Note: for reliability in the following section about the individual jobs, each job should have a property of the last value it processed and be able to start with that value when processing in the event it is set.  Also, during execution, after each element is executed that property value should be updated and the message’s visibility timeout should be updated to extend it long enough to process the next element.  This ensures that no action is done twice and if something happens with the cloud service instance, such as it rebooting or failing to respond, another instance will pick up where the previous one left off after the timeout period expires.

###Master Job

The Master Job is similarly simple:

1. For each time-interval since launch
2. Add a per-time period job to the queue
3. Remove yourself from the queue

###Per-Time Period Job

1. Query by partition key of the time-period specified in the job’s message
2. Add a per-device job to the queue with a properties specifying the deviceId and the last row key that was read
3 .Remove yourself from the queue

###Per-Device Job

1. Query the sensor table by partitionkey=deviceId and rowkey>lastreadrowkey (both values are contained in the message)
2. Perform whatever processing you want on the data, some examples:
	* you may want to update a datastore of the real-time temperature per device to be displayed on a map on a webpage.
	* you may want to calculate the rate the temperature is changing at that time and place
	* you may want to continuously append to a blob for longer, Hadoop-esque big data analytics and/or machine learning (Note I have a pending post on appending to blob storage efficiently – the azure storage team also has announced that they’ll have a method for doing that in the Q3 release of their SDK)
3. et cetera and so on for whatever sensor data you’re collecting.

###Putting it all together

Finally you’re going to want to add the master job to Scheduler and set it to run at whatever rate you wish to process the data from the deployed devices.  You’re then going to want to configure and tune the auto scaling rules on the cloud service such that as new devices are deployed performance for processing all of them stays constant.

##Summary

Here I’ve shown you one way, at a high level in psudo-code, to create a highly scalable cloud service capable of handling ‘Internet of Things’ scale of load in a very simple, cost effective and efficient way on Azure’s Core Services (which are the longest running, most reliable services on azure).  This is just one way to do this particular part of an IoT project and leaves out bits on doing BigData analytics and machine learning.  Going forward I’ll do some follow up posts with actual code, more on the BigData and Machine Learning bits along with how you can leverage many of Azure’s newer services to do a lot of what’s discussed here out of the box.

Happy Coding!

-Ben