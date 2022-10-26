# Guide for Logstash on RHEL(8) Linux Box: 

1. [The first step is to install Logstash on RHEL 8](https://www.elastic.co/guide/en/logstash/8.4/installing-logstash.html#_yum)  
    
### We are importing the RPM package
    sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch 

Esclate to root (sudo command or su)  - This will save you a lot of time later  
    
    vi /etc/yum.repos.d/logstash.repo  

Ex. Config below  

    [logstash-8.x]
    name=Elastic repository for 8.x packages
    baseurl=https://artifacts.elastic.co/packages/8.x/yum
    gpgcheck=1
    gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    autorefresh=1
    type=rpm-md
    
After creating the above config file and saving we are ready to install:  

    sudo yum install logstash


# 2. Cert Creation 
    cd /etc/logstash  


## Creating certs
    openssl req -x509 -days 3650 -nodes -newkey rsa:2048 -keyout logstash-remote.key -out logstash-remote.crt

## Example Config of cert creation  
    Country Name (2 letter code) [XX]:US
    State or Province Name (full name) []:
    Locality Name (eg, city) [Default City]:
    Organization Name (eg, company) [Default Company Ltd]:
    Organizational Unit Name (eg, section) []:
    Common Name (eg, your name or your server's hostname) []: *HOSTNAME OF THE BOX YOU ARE ON (If this is in AWS for example: ec2-18-215-144-147.compute-1.amazonaws.com OR on-prem the hostname/ip )* 
    Email Address []:

### IF YOU DID THIS IN YOUR ROOT DIRECTORY, its ok. just move the above files to the root directory of logstash on rhel 8 that is "/etc/logstash"  
    cp ~/home/logstash-remote.crt /etc/logstash  

    cp ~/home/logstash-remote.key /etc/logstash  

    # Giving Logstash the correct permissions to read the key
    sudo chmod 644 logstash-remote.key  


### CP or MV (I did mv) the example config file to create your new config.

*I prefer to do 'mv' over cp but to each their own*

    mv logstash-sample.conf logstash.conf  
    OR 
    cp logstash-sample.conf logstash.conf  

### *NOTE* - [Logstash requires us to install the syslog dependent plugin for logstash](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-syslog.html#plugins-outputs-syslog-common-options)
    /usr/share/logstash/bin/logstash-plugin install logstash-output-syslog

### Edit your config (vi logstash.conf)
    input {
    tcp
        {
        mode => "server"
        host => "0.0.0.0"
        port => 6515
        ssl_enable => true
        ssl_verify => false
        ssl_cert => "/etc/logstash/logstash-remote.crt"
        ssl_key => "/etc/logstash/logstash-remote.key"
        }
    }
    output {
        syslog{
        host => "127.0.0.1"
        port => 6514
        }
    }

### Once your config file has been created, copy it to `/etc/logstash/conf.d/`

    cp logstash.conf /etc/logstash/conf.d/
 

### Here is an example command to test if logstash can receive syslog over tls  

*Note* - You can use your local mac or pc for this. **I used a Mac**

    touch request.txt
    echo "Hello World" > request.txt 
    cat request.txt | openssl s_client -connect HOSTNAME:PORT


### **OPTIONAL** Install Sumo Logic collector to listen locally on the port to send encrypted TLS syslog to Sumo Logic  
*NOTE* I used wget... But rhel 8 doesn't have it by default. Yes, don't judge this post because I used wget instead of curl.. you old school linux nerds :smirk: . Also this is why debian is better, cause wget is there by defualt :smiling_imp:

# WGET
    yum install wget

    wget -c -O 64.sh https://collectors.sumologic.com/rest/download/linux/64 

    mv 64 64.sh 

    chmod +x 64.sh 

    sudo ./64.sh -q -Vsumo.token_and_url=<>

*Note above, please paste token right after the "=" 

# CURL - This is for the folks who can't use wget :smiling_imp:
    curl https://collectors.sumologic.com/rest/download/linux/64  --output 64.sh

    mv 64 64.sh 

    chmod +x 64.sh 

    sudo ./64.sh -q -Vsumo.token_and_url=<>

*Note above, please paste token right after the "="