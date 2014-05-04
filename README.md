# Introduction

Jahmon is a comprehensive cloud monitoring solution for OpenStack based clouds. It employs node-based agents to report metrics to a RESTful API, where alarms can be created and evaluated and notifications are sent. Monitoring enables users to understand the operational effectiveness of the services and underlying infrastructure that make up their cloud and provide actionable details when there is a problem.

* Target Platform: OpenStack based systems and companies that use OpenStack.
* What: Node agent based metrics collection to a RESTful API.
* Benefits: System status and supporting metrics are constantly monitored, readily available, and trackable, making system management tasks more timely and predictable. 
* For Who: System admins or operators

## Features

This section describes the overall features iof Jahmon.

* A highly performant, scalable, reliable and fault-tolerant monitoring solution that scales to service provider metrics levels of metrics throughput. Performance, scalability and high-availability have been designed in from the start. Jahmon can process 100s of thousands of metrics/sec as well as offer data retention periods of greater than a year with no data loss while still processing interactive queries.

* Consolidates and unifies both operational (internal) and Monitoring as a Service (customer facing) capabilities. Service providers usually treat service operational monitoring separately from customer facing Monitoring as a Service (MaaS). This distinction is arbitrary and our solution has been designed to handle both, which allows us to reduce the number of systems that are required for monitoring. Jahmon is a multi-tenant solution.

* Rest API for storing and querying  metrics and historical information. Most monitoring solution use special transports and protocols, such as CollectD or NSCA (Nagios). In our solution, http is the only protocol used. This simplifies the overall design and also allows for a much richer way of describing the data via dimensions.

* Multi-tenant and authenticated. Metrics are submitted and authenticated using Keystone and stored associated with a tenant ID.

* Metrics defined using a set of (key, value) pairs called dimensions.

* Thresholding and alarming on metrics.

* Compound alarms using a simple expressive grammar.

* Monitoring agent that supports a number of built-in system and service checks and also supports Nagios checks and statsd.

* Based on our experiences at HP around monitoring our internal public cloud infrastructure, http://www.hpcloud.com/, as well as running our customer-facing HP Cloud Monitoring, http://www.hpcloud.com/products-services/monitoring.

* Open-source monitoring solution built on open-source technologies.

# Architecture

![Component Diagram](mon-arch-component-diagram.png "Monitoring Component Diagram")

* Monitoring Agent (mon-agent): A modern Python based monitoring agent that consists of several sub-components and supports system metrics, such as cpu utilization and available memory, Nagios plugins, statsd and many built-in checks for services such as MySQL, RabbitMQ, and many others.
 
* Monitoring API (mon-api): A RESTful API for monitoring that is primarily focused on the following concepts and areas:

	* Metrics: Store and query vast amounts of metrics in real-time. 
	* Statistics: Query statistics for metrics.
	* Alarms: Create, update, query and delete alarms and query the alarm history. Jahmon supports a simple expressive grammar for creating compound alarms. The comple state transition for alarms is stored and queryable which allows for subsequent root cause anlaysis (RCA) or advanced analytics.
	* Notification Methods: Create and delete notification methods and associate them with alarms, such as email. Jahmon supports the ability to notify users directly via email when an alarm state transitions occur.
	
* Persister (mon-persister): Consumes metrics and alarm state transitions from the MessageQ and stores them in the Metrics and Alarms database. The Persister uses the Disruptor library, http://lmax-exchange.github.io/disruptor/. We will look into converting the Persister to a Python component in the future.

* Transform and Aggregation Engine: Transform metric names and values, such as delta or time-based derivative calculations, and creates new metrics that are published to the Message Queue. The Transform Engine is not available yet.

* Threshold Engine (mon-thresh): Computes thresholds on metrics and publishes alarms to the MessageQ when exceeded. Based on Apache Storm a free and open distributed real-time computation system.

* Notification Engine (mon-notification): Consumes alarm state transition messages from the MessageQ and sends notifications, such as emails for alarms. The Notification Engine is Python based.

* Message Queue: A third-party component that primarily receives published metrics from the Monitoring API and alarm state transition messages from the Threshold Engine that are consumed by other components, such as the Persister and Notification Engine. The Message Queue is also used to publish and consume other events in the system. Currently, a Kafka based MessageQ is supported. Kafka is a highly performance, distributed, fault-tolerant, and scalable message queue with durability built-in. We will look at other alternatives, such as RabbitMQ and in-fact in our previous implementation RabbitMQ was supported, but due to performance, scale, durability and high-availability limitiations with RabbitMQ we have moved to Kafka.

* Metrics and Alarms Database: A third-party component that primarily stores metrics and the alarm state history. Currently, Vertica is supported. We will look at other open-source alternatives, such as MySQL or a NoSQL database.

* Config Database: A third-party component that stores a lot of the configuration and other information in the system. Currently, MySQL is supported.

* Monitoring Client (python-monclient): A Python command line client and library that communicates and controls the Monitoring API. The Monitoring Client was written using the OpenStack Heat Python client as a framework. The Monitoring Client also has a Python library, "monclient" similar to the other OpenStack clients, that can be used to quickly build additional capabilities. The Monitoring Client library is used by the Monitoring UI, Ceilometer publisher, and other components.

* Monitoring UI: A Horizon dashboard for visualizing the overall health and status of an OpenStack cloud.

* Ceilometer publisher: A multi-publisher plugin for Ceilometer, not shown, that converts and publishes samples to the Monitoring API.

## Post Metric Sequence

This section shows the sequence of operations involved in posting a metric to the Monitoring API.

![Post Metric](mon-arch-post-metric-diagram.png "Post Metric Sequence Diagram")

0. A metric is posted to the Monitoring API.
1. The Monitoring API authenticates and validates the request and publishes the metric to the the Message Queue.
2. The Persister consumes the metric from the Message Queue and stores in the Metrics Store.
3. The Transform Engine consumes the metrics from the Message Queue, performs transform and aggregation operations on metrics, and publishes metrics that it creates back to Message Queue.
4. The Threshold Engine consumes metrics from the Message Queue and evaluates alarms. If a state change occurs in an alarm, an "alarm-state-transitioned-event" is published to the Message Queue.
5. The Notification Engine consumes "alarm-state-transitioned-events" from the Message Queue, evaluates whether they have a Notification Method associated with it, and send the appropriate notification, such as email.
6. The Persister consumes the "alarm-state-transitioned-event" from the Message Queue and store in the Alarm State History Store.


# Development Environment

Jahmon comes with a turn-key development environment based on Vagrant, that can be used for quickly deploying Jahmon on a client system, such as a MAC OS X based system.

Jahmon also comes with a number of Chef cookbooks.

# Technologies

Jahmon uses a number of underlying technologies:

* Vertica (http://www.vertica.com): A commercial SQL analytics database that is highly scalable. It offers built-in automatic high-availability and excels at in-database analytics and compressing and storing massive amounts of data. In Jahmon, we use Vertica primarily for storing and querying metrics and the alarm history. A free version of Vertica that can store up to 1 TB of data with no time-limit is available at, https://my.vertica.com/community/. This should be sufficient to get developers started with using Jahmon and support smaller installations or where data retention periods aren't that long.

* Apache Kafka (http://kafka.apache.org): Apache Kafka is publish-subscribe messaging rethought as a distributed commit log. Kafka is a highly performant, distributed, fault-tolerant, and scalable message queue with durability built-in. 

* Apache Storm (http://storm.incubator.apache.org/): Apache Storm is a free and open source distributed realtime computation system. Storm makes it easy to reliably process unbounded streams of data, doing for realtime processing what Hadoop did for batch processing.

* ZooKeeper: Used by Kafka and Storm, http://zookeeper.apache.org/

* MySQL:

* Vagrant (http://www.vagrantup.com/): Vagrant provides easy to configure, reproducible, and portable work environments built on top of industry-standard technology and controlled by a single consistent workflow to help maximize the productivity and flexibility of you and your team.

* Dropwizard (https://dropwizard.github.io/dropwizard/): Dropwizard pulls together stable, mature libraries from the Java ecosystem into a simple, light-weight package that lets you focus on getting things done. Dropwizard has out-of-the-box support for sophisticated configuration, application metrics, logging, operational tools, and much more, allowing you and your team to ship a production-quality web service in the shortest time possible.

* Disruptor ()http://lmax-exchange.github.io/disruptor/): The Disruptor is a high-speed ring buffer implementation.

# Future Plans

The initial Jahmon code-base has been released by HP as an open-source project and is initially focused on a metrics processing, alarming and notifications pipeline. Although the initial release includes a lot of features, there is still a significant amount of work to do. We are very interested in working with other companies and open-source developers. Here is just a sample of the items that we think make sense to work on next.

* Converting more components to Python: Several of the components in Jahmon have been written using Python, but several have been written in Java and Java libraries or frameworks. We are pursuing converting these to Python where it makes sense.
 
* Support for events. Currently we are focused on a metrics processing engine, but we see events as an important area to address.

* Support for an open-source Metrics and Alarm History database. Currently, Vertica is used for Metrics and Alarm History database. While Vertica is an absolutely amazing analytics database and has a free Community Edition available, we realize that support of only a commerical database in an open-source project is impediment for more wide-spread adoption by the open-source community. We will be adding support for at least one completely open-source database.

* More advanced in-database analytics: Current support for statistics could be greatly extended by adding in-database process of standard of deviation, moving window averages, and many more. We would also like to add the ability to do anomaly detection.   

* Support for additional service checks: The Monitoring Agent support a number of built-in system and service checks, such as MySQL and RabbitMQ.

* OpenStack integration. Currently, we have developed a Ceilometer multi-publisher plugin that published to the Monitoring API. We will be continuing to work with the OpenStack community on additional integration points. Potential areas are:
	* Heat:
	* Keystone: As the Monitoring API uses Keystone for authentication we would like to see the addition of access key authentication, covered in https://blueprints.launchpad.net/keystone/+spec/access-key-authentication.
	* TripleO: When TripleO is used to deploy OpenStack we want the Monitoring Agents and Alarms to be automatically deployed as well.
	
* Transform and Aggregation Engine:

* Support for database migrations:

* Integration with Graphite

* As we are also building Jahmon for HP's CloudSystem we will be developing an integrated and automatically configured solution for the Monitoring Agents and Alarms.



# Contact

* Roland Hochmuth: roland.hochmuth@hp.com

# License

Copyright (c) 2014 Hewlett-Packard Development Company, L.P.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0
    
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
implied.
See the License for the specific language governing permissions and
limitations under the License.



