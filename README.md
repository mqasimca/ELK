# How To Install ELK In LXD Containers

Centralized logging can be very useful when attempting to identify problems with your servers or applications, as it allows you to search through all of your logs in a single place. It is also useful because it allows you to identify issues that span multiple servers by correlating their logs during a specific time frame.

### ELK stack consists of the following

###  Logstash: 

The server component of Logstash that processes incoming logs

###  Elasticsearch

Stores all of the logs

### Kibana

Web interface for searching and visualizing logs, which will be proxied through Nginx

### Filebeat

Installed on client servers that will send their logs to Logstash, Filebeat serves as a log shipping agent that utilizes the lumberjack networking protocol to communicate with Logstash


### Requirement 

 LXD Setup
 
## Installation

Create ELK network (Optional)   

    lxc network create ELK0 ipv6.address=none ipv4.address=10.80.3.1/24 ipv4.nat=true

Download Ubuntu LTS 16.04 image

    lxc image copy images:ubuntu/xenial/amd64 local: --alias=Ubuntu-LTS --copy-aliases --auto-update

ElasticSearch :

	lxc init Ubuntu-LTS ElasticSearch

Attach ELK0 network

	lxc network attach ELK0 ElasticSearch eth0

Assign static IP to  container

	lxc config device set ElasticSearch eth0 ipv4.address 10.80.3.25

Start ElasticSearch container

	lxc start ElasticSearch

Login container

	lxc exec Elasticsearch bash

Update container

	apt-get update

Install necessary packages

	apt-get install python-software-properties software-properties-common -y

Isntall JAVA

	add-apt-repository -y ppa:webupd8team/java -y && apt-get update
	apt-get -y install oracle-java8-installer

Install ElasticSearch

	apt-get install -y wget
	wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
	echo "deb http://packages.elastic.co/elasticsearch/2.x/debian stable main" >> /etc/apt/sources.list.d/elasticsearch-2.x.list
	apt-get update
	apt-get -y install elasticsearch
	echo "network.host: [_eth0_]" >> /etc/elasticsearch/elasticsearch.yml
	systemctl restart elasticsearch
	systemctl enable elasticsearch
	systemctl status elasticsearch

To check ElasticSearch is working

	netstat -atlnp