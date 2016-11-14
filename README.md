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

**ElasticSearch :**

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

Start / Enable Elasticsearch service

	systemctl restart elasticsearch
	systemctl enable elasticsearch
	

To check ElasticSearch is working

	systemctl status elasticsearch
	netstat -atlnp

**Kibana :**

	lxc init Ubuntu-LTS Kibana

Attch ELK0 network to Kibana network

	lxc network attach ELK0 Kibana eth0

Assign static IP to Kibana container

	lxc config device set Kibana eth0 ipv4.address 10.80.3.35

Start Kibana container

	lxc start Kibana

Login Kibana container

	lxc exec Kibana bash

Install necessary packages

	apt-get update
	apt-get install python-software-properties software-properties-common wget -y

Install Kibana

	wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
	echo "deb http://packages.elastic.co/kibana/4.4/debian stable main" > /etc/apt/sources.list.d/kibana-4.4.x.list
	apt-get update
	apt-get -y install kibana --allow-unauthenticated

Start / Enable Kibana service

	systemctl restart kibana
	systemctl enable kibana

To check Kibana is working

	systemctl status kibana
	netstat -atlnp

**Nginx :**

	lxc init Ubuntu-LTS Nginx

Attach ELK0 network to Nginx continer

	lxc network attach ELK0 Nginx eth0

Assign static IP to Nginx container

	lxc config device set Nginx eth0 ipv4.address 10.80.3.5

Start Nginx container

	lxc start Nginx

Login Nginx container

	lxc exec Nginx bash

Install Nginx in Nginx container

	apt-get update
	apt-get install -y nginx apache2-utils

Set the password for Kibana Admin

	htpasswd -c /etc/nginx/htpasswd.users kibanaadmin

Remove the default Nginx configuration

	rm -rf /etc/nginx/sites-available/default

Copy paste the following

	vi /etc/nginx/sites-available/default
	server {

	server {
    listen 80;

    server_name example.com;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;

    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;        
    		}
	}

Start / Enable Nginx service

	systemctl restart nginx
	systemctl enable nginx

To check Nginx service

	systemctl status nginx

**Logstash :**

	lxc init Ubuntu-LTS LogStash
	lxc network attach ELK0 LogStash eth0
	lxc config device set LogStash eth0 ipv4.address 10.80.3.50

Start LogStash container

	lxc start LogStash

Login LogStash container

	lxc exec LogStash bash

Install necessary packages

	apt-get install python-software-properties software-properties-common wget -y
	add-apt-repository -y ppa:webupd8team/java -y && apt-get update
	apt-get -y install oracle-java8-installer

Installing Logstash

	echo 'deb http://packages.elastic.co/logstash/2.2/debian stable main' > /etc/apt/sources.list.d/logstash-2.2.x.list
	apt-get update
	apt-get install logstash --allow-unauthenticated

Configure Logstash

	vi /etc/logstash/conf.d/02-beats-input.conf
	
Copy paste the following

	input {
 	beats {
    port => 5044
  	}
	}

Now create a configuration file called 10-syslog-filter.conf, where we will add a filter for syslog

	vi /etc/logstash/conf.d/10-syslog-filter.conf

Copy paste the following

	filter {
  	if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    syslog_pri { }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  	}
	}

Lastly, create a configuration file called 30-elasticsearch-output.conf

	vi /etc/logstash/conf.d/30-elasticsearch-output.conf

Copy paste the following

	output {
  	elasticsearch {
    hosts => ["10.80.3.25:9200"]
    sniffing => true
    manage_template => false
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
    document_type => "%{[@metadata][type]}"
  	}
	}

Test your Logstash configuration

	service logstash configtest

Start / Enable LogStash

	systemctl start logstash
	systemctl enable logstash

To check LogStash service

	systemctl status logstash
	netstat -atlnp