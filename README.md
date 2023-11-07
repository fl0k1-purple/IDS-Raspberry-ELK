# ElK stack with Filebeat sensors
### Requisites:
- Linux Host - Either server or desktop, flavour of choice.
- Raspberry Pi - it will be used a sensors, for this we need a USB Ethernet dongle and configure both Ethernet ports as bridge.
- A Network with interesting traffic.

## ElK Stack:
---
Elasticsearch, Logstash and Kibana are part of this stack. Elasticsearch and Logstash handle data transport and communicate with Kibana so it can display all the processed data.

## Elasticsearch (8.5)
---
### Installation:

1. Import the PGP Key.
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

2. Install repository (works wiht APT).
```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```
3. Update and then install the package.
```bash
sudo apt-get update && sudo apt-get install elasticsearch
```
4. After the installation is complete, a __Security autoconfiguration information__ prompt will appear, from that take note of the generated password for the **elastic** user. 
```bash
The generated password for the elastic built-in superuser is : <password>
```
5. Finally you will need to generate a _enrollment token_ for elasticsearch's integration with Kibana using this command:
```bash
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```
Note: All elasticsearch commands are located in ```/usr/share/elasticsearch/bin```, there you will find ways to create new users and reset passwords, among other functionalities.

6. Now you can enable and start the Elasticsearch service:
```bash
systemctl enable elasticsearch.service
systemctl start elasticsearch.service
```

Note: If you want to use anything other than **localhost** as your access to the service, either a domain or a private IP, you will need to edit the configuration file at ```/etc/elasticsearch/elasticseach.yml ``` and then restart the service.

## Kibana (8.5)
---
### Installation:
Now we already have the PGP Key and repository in our system, so all it is left is to install the _Kibana_ package.

1. Install the package normally.
```bash
apt install kibana
```
2. Upon finishing the installation a unique link will prompt to enroll your Kibana instance with Elasticsearch. 
   Click the generated link and the paste the _enrollment token_ previously generated and click the button to connect the instance. Finally you will need to log with the user *elastic* and the password for it (we got it on section 4 of the Elasticsearch installation).
3. Now you can enable and start the Kibana service:
```bash
systemctl enable kibana.service
systemctl start kibana.service
```
Note: If you want to use anything other than **localhost** as your access to the service, either a domain or a private IP, you will need to edit the configuration file at ```/etc/kibana/kibana.yml ``` and then restart the service. 

Both Elasticsearch configuration and Kibana Configuration must have the right IP or domain of each other to work properly.
## Logstash (8.5)
---
If needed you can add Logstash to your machine, to stablish communication when Filebeat has no module available for the service you need, in our case though the Snort module can communicate directly to Elasticsearch.
### Installation:

1. Install the package normally.
```bash
apt install logstash
```
2. Now we need to create the data flow. First create the input with the listening port.
```bash
sudo vim /etc/logstash/conf.d/02-beats-input.conf
```
Paste the following configuration:
```bash 
input {
  beats {
    port => 5044
  }
}
```
3. Then the output to the Elasticsearch service.
```bash
sudo vim /etc/logstash/conf.d/03-elasticsearch-output.conf
```
Paste the following configuration:
```bash 
output {
  if [@metadata][pipeline] {
        elasticsearch {
        hosts => ["https://YOUR_SEVICE_IP_HERE:9200"]
        ssl => true
        ssl_certificate_verification => false
        user => "elastic"
        password => "YOUR_GENERATED_PASSWORD_HERE"

        manage_template => false
        index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        pipeline => "%{[@metadata][pipeline]}"
        }
  } else {
        elasticsearch {
        hosts => ["https://YOUR_SEVICE_IP_HERE:9200"]
        ssl => true
        ssl_certificate_verification => false
        user => "elastic"
        password => "YOUR_GENERATED_PASSWORD_HERE"

        manage_template => false
        index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        }
  }
}
```
Note: Since Elasticsearch is now ssl protected by default, you will need either to provide the certificate path or disable the ssl verification, the example above uses the last option.

4. You can test your configuration with the following command:
```bash
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

5. If you see the ```Configuration Vlidation Result: OK``` you can now enable and start the service.
```
systemctl enable logstash
systemctl start logstash
```

## Filebeat
---
The last step for this service is to configurate every sensor you intend to put in the network. It is recommended if working with a Raspberry Pi to run the official OS with no desktop environment and 64-bit. 
### Installation:
1. Import the PGP Key.
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

2. Install repository (works wiht APT).
```bash
sudo apt-get install apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```
3. Update and then install the package.
```bash
sudo apt-get update && sudo apt-get install filebeat
```
4. Edit the Filebeat ```.yml``` configuration file to direct the logs to both Elasticsearch and Kibana.
Uncomment the kibana section and put the corresponding IP:
```bash
setup.kibana:
	host: "IP:port"
```
Finally uncomment the output.elasticsearch section and add the following options:
```bash
output.elasticsearch:
	hosts: ["ip:port"]
	protocol: "https"

	ssl:
		enabled: true
		ca_trusted_fingerprint: "fingerprint in hex without :"
	username: "elastic"
	password: "pass"
```
Since Elasticsearch is ssl protected you will need to get the certificate's fingerprint and paste it on the Filebeat configuration. You can do that by running the following command:
```bash
openssl x509 -fingerprint -sha256 -noout -in /path/to/elastic-stack-ca.crt.pem
```
Where the path is usually at ```/etc/elasticsearch/certs/httP_ca.crt```.

### Snort Module
---
For the snort module you will need to run the following configuration:
1. Enable the module.
```bash
filebeat modules enable snort
```
You can verify it by running:
```bash
filebeat modules list
```
2. Then edit the module configuration at the path ```/etc/filebeat/modules.d/snort.conf```, then paste the following configuration:
```bash 
module: snort
  log:
    enabled: true
    var.input: file
    var.paths: ["/var/log/snort/snort.alert.fast","/var/log/snort/snort.log.*"]
```
3. Now you need to load the pipelines and the index for Kibana:
```bash
filebeat setup --pipelines
filebeat setup --index-management
```
4. Finally run the full setup:
```bash
filebeat setup
```
5. If everything is OK and no error are displayed you just enable and start the service.
```bash
systemctl enable filebeat.service
systemctl start filebeat.service
```

## Snort

For the purpose of this project we are running Snort as a background process and logging. We accomplish that with this command:
```bash
snort -d -c /etc/snort/snort.conf -l /snort-logs -D -i eth0
```
In the ```snort.conf``` you will need to uncomment and/or add your desired policies for alerting upon activation
