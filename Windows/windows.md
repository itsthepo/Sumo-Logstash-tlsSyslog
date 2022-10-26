# Guide for Logstash on Windows 2016 Server: 

# 1. [The first step is to download the Logstash .zip on Windows 2016 Server](https://artifacts.elastic.co/downloads/logstash/logstash-8.4.3-windows-x86_64.zip)  



##  We need to extract the files out of our downloads folder. *Note* - I chose the C:\ but please updated to your required folders. Please note that the Microsoft has a limit on the filepath being TOO LONG. Putting in C: helps this.  

    Expand-Archive -LiteralPath $env:userprofile\Downloads\logstash-8.4.3-windows-x86_64.zip -DestinationPath C:\logstash-8.4.32


### After the extraction has finished (it may take awhile), we need to edit the config. By default Logstash provides an "example" *name could change*: "logstash-sample.conf"
    
    cd C:\logstash-8.4.3\config\  
    
    mv logstash-sample.conf logstash.conf 
    OR
    cp logstash-sample.conf logstash.conf

Ex. Config below  

    input {
    tcp
        {
        mode => "server"
        host => "0.0.0.0"
        port => 6515
        ssl_enable => true
        ssl_verify => false
        ssl_cert => "C:\logstash-8.4.3\logstash-remote.crt"
        ssl_key => "C:\logstash-8.4.3\logstash-remote.key"
        }
    }
    output {
        syslog{
        host => "127.0.0.1"
        port => 6514
        }
    }
    
***Don't fret, we will take care of the cert directories in section 2***  

# 2. Cert Creation 
    cd C:\logstash-8.4.3\config\  


## Creating certs 
***Unfortunately, getting openssl on windows 2016 server is challenging, so I recommended heading to a linux/mac to quickly run the command below with a similar [example config](#Example_Config_of_cert_creation)  

    openssl req -x509 -days 3650 -nodes -newkey rsa:2048 -keyout logstash-remote.key -out logstash-remote.crt

## Example Config of cert creation  
    Country Name (2 letter code) [XX]:US
    State or Province Name (full name) []:
    Locality Name (eg, city) [Default City]:
    Organization Name (eg, company) [Default Company Ltd]:
    Organizational Unit Name (eg, section) []:
    Common Name (eg, your name or your server's hostname) []: ec2-18-215-144-147.compute-1.amazonaws.com
    Email Address []:

**Note - the "common name" needs to be the hostname that will be actually receiving the data.**  
For example:  
- If your server is in AWS, you will use the FQDN in the EC2 console
- If your server is on-prem, you can use either the FQDN or the IP of the server

### After you have genered the logstash-remote.crt and logstash-remote.key files, copy them to your windows server and place them in the 'root' directory of logstash ```C:\logstash-8.4.3\```


### *NOTE* - [Logstash requires us to install the syslog dependent plugin for logstash](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-syslog.html#plugins-outputs-syslog-common-options). Run the following command: 
    C:\logstash-8.4.3\bin/logstash-plugin install logstash-output-syslog


# Here is an example command to test if logstash can receive syslog over tls  

*Note* - You can use your local mac or pc for this. **I used a Mac**

    touch request.txt
    echo "Hello World" > request.txt 
    cat request.txt | openssl s_client -connect HOSTNAME:PORT 


# **OPTIONAL** Install Sumo Logic collector to listen locally on the port to send encrypted TLS syslog to Sumo Logic. This walk through assumes that you will install the logstash and Sumo collector on the same box (note the syslog output config above 127.0.0.1). 

*At Sumo we recommended installing collectors using [tokens](https://help.sumologic.com/docs/manage/security/installation-tokens/).*


## Powershell/CLI
**NOTE - When using quiet mode installation on Windows with Microsoft PowerShell, parameters following -console -q must be escaped with quotes, for example:**  

*Using your preferred method of downloading files on windows, the link is [here](https://collectors.sumologic.com/rest/download/win64).*

    SumoCollector.exe -console -q "-Vsumo.token_and_url=<installationToken>" 

*Note above, please paste token right after the "=" 

## GUI
In your web-browser paste in the following [link](https://collectors.sumologic.com/rest/download/win64) 

- Run through the GUI installation process. Our help documentation can be found [here](https://help.sumologic.com/docs/send-data/installed-collectors/windows/#install-a-collector-on-windows)

- Setup a [syslog source](https://help.sumologic.com/docs/send-data/installed-collectors/sources/syslog-source/)
    - This should listen on the port you specified above in the config. The default is 6514


