#### An Ambari Service to deploy HDP IoT Demo
Ambari service for easily installing and managing [HDP IoT Demo](https://github.com/hortonworks-gallery/iot-truck-streaming) which shows real time monitoring/alerts and predictions of driving violations generated by fleet of trucks. 

Videos on the demo itself available here:
  - [Arun's Hadoop Summit 2015 keynote](https://youtu.be/FHMMcMYhmNI?t=1h25m13s)
  - [Shaun/George's Hadoop Summit 2014 Keynote](http://library.fora.tv/program_landing_frameview?id=20333&type=clip)
  - [Nauman's Demo at Phoenix Data conference 2014](http://www.youtube.com/watch?v=ErDmSIQ4gX0)
  - [![Phoenix Data conference 2014](http://img.youtube.com/vi/ErDmSIQ4gX0/0.jpg)](http://www.youtube.com/watch?v=ErDmSIQ4gX0)

Pre-reqs: 
  - The service currently requires that it is installed on the Ambari server node and that Kafka and Zookeeper and also running on the same node.
  - HBase and Storm must be available on the cluster and started. 
  - Falcon must be stopped before installing this service.

Limitations:
  - This is not an officially supported service and *is not meant to be deployed in production systems*. It is only meant for testing demo/purposes
  - It does not support Ambari/HDP upgrade process and will cause upgrade problems if not removed prior to upgrade


Previous versions:
  - For 2.3 version of the steps see [here](https://github.com/hortonworks-gallery/iotdemo-service/blob/master/README-23.md)
  - For 2.2 version of the steps see [here](https://github.com/hortonworks-gallery/iotdemo-service/blob/master/README-22.md)
  
##### Setup steps

- Download HDP 2.4 sandbox VM image (Hortonworks_sanbox_with_hdp_2_4_vmware.ova) from [Hortonworks website](http://hortonworks.com/products/hortonworks-sandbox/)
- Import Hortonworks_sanbox_with_hdp_2_4_vmware.ova into VMWare and set the VM memory size to at least 8GB (preferably 10GB) RAM and at least 4cpus allocated.
- Now start the VM
- After it boots up, find the IP address of the VM and add an entry into your machines hosts file e.g.
```
192.168.191.241 sandbox.hortonworks.com sandbox    
```
- Connect to the VM via SSH (password hadoop) and restart Ambari server
```
ssh root@sandbox.hortonworks.com 
```

- **Make sure Storm, HBase, Kafka, Hive are up and Falcon is down** and all these services are out of maintenance mode. You can SSH into Ambari server node and run the below as a shortcut.
  - Before proceeding, you may want to wait a few minutes to ensure they stay up reliably or the demo setup may fail. If they do not, you may need to increase the memory/cpus allocated to the VM.

```
#Ambari password
export PASSWORD=admin
#Ambari host
export AMBARI_HOST=localhost

#detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`

#make sure kafka, storm, falcon are out of maintenance mode
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Falcon from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/FALCON
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Kafka from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/KAFKA
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Storm from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/STORM
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Hbase from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/HBASE
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Hive from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/HIVE


#Start Kafka, Storm, HBase, Hive
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start KAFKA via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/KAFKA
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start STORM via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/STORM
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start HBASE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/HBASE
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start HIVE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/HIVE


#stop Falcon
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop FALCON via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/FALCON

```

- Wait for all the operations to start/stop services complete

- Deploy the IoTDemo service as well as [Apache Zeppelin service](https://github.com/hortonworks-gallery/ambari-zeppelin-service) to visualize/analyze violations events generated via prebuilt notebook
```
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
sudo git clone https://github.com/hortonworks-gallery/iotdemo-service.git   /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/IOTDEMO   

#(Optional) Zeppelin already comes installed on HDP 2.4 sandbox
#sudo git clone https://github.com/hortonworks-gallery/ambari-zeppelin-service.git   /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/ZEPPELIN   

#on sandbox
sudo service ambari restart

#on non sandbox
sudo service ambari-server restart
```
- Then you can click on 'Add Service' from the 'Actions' dropdown menu in the bottom left of the Ambari dashboard:

On bottom left -> Actions -> Add service ->  'IoT Demo' (also check 'Zeppelin' if not already installed) -> Next -> Next -> Configure service -> Next -> Deploy
![Image](../master/screenshots/select-service.png?raw=true)

Things to remember while configuring the service
  
  - The service currently requires that it is installed on the Ambari server node and that Kafka and Zookeeper and also running on the same node.
    - If kafka is on a different node, the demo still could work (not tested) if you manually create the topics ahead of time 
    ```
    /usr/hdp/current/kafka-broker/bin/kafka-topics.sh --create --zookeeper $ZK_HOST:2181 --replication-factor 1 --partitions 2 --topic truck_events
    ```

  - **Under "Advanced user-config": update the Ambari admin user/password/port**
    - These will be used to check the required services are up
    - Needed on HDP 2.4 sandbox because the default admin password has been changed.  

  - Under "Advanced demo-config". 
    - enter public name/IP of IoTDemo node: This is used to setup the Ambari view. Set this to the public host/IP of IoTDemo node (which must must be reachable from your local machine). If installing on sandbox (or local VM), change this to the IP address of VM. If installing on cloud, set this to public name/IP of IoTDemo node. Alternatively, if you already have a local hosts file entry for the internal hostname of the IoTDemo node (e.g. sandbox.hortonworks.com), you can leave this empty - it will default to internal hostname
      - you should use the same value for publicname property in Zeppelin config as well
      
  - Under "under "Advanced demo-env"
    - **You do NOT have to enter your github credentials anymore**. The code will be picked up from https://github.com/hortonworks-gallery/iot-truck-streaming
      
  - The IoT demo configs are available under "Advanced demo-env", but do not require updating as all required configs will be auto-populated:
    - Ambari host
    - Name node host/port
    - Nimbus host
    - Hive metastore host/port
    - Supervisor host
    - HBase master host
    - Kafka host/port (also where ActiveMQ will be installed)
  
![Image](../master/screenshots/config1.png?raw=true)

![Image](../master/screenshots/config2.png?raw=true)
      

- On successful deployment you will see the IOTDEMO service as part of Ambari stack and will be able to start/stop the service from here:
![Image](../master/screenshots/started-service.png?raw=true)

- You can see the parameters you configured under 'Configs' tab
![Image](../master/screenshots/started-config.png?raw=true)

- One benefit to wrapping the component in Ambari service is that you can now monitor/manage this service remotely via REST API
```
export SERVICE=IOTDEMO
export PASSWORD=admin
export AMBARI_HOST=localhost

#detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`

#get service status
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X GET http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#start service
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#stop service
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
```


#### Intall the view

- If iotdemo service was *not* deployed on same node as Ambari server:
  - Transfer the compiled view jar from /root/iotdemo-view/target/*.jar on the iotdemo node to /var/lib/ambari-server/resources/views on ambari server node
  - Follow steps below to stop IotDemo service, restart ambari, start IoTDemo service:

- Otherwise, if iotdemo service was deployed on same node as Ambari server:
  - Stop the IoTDemo service from Ambari
  - Restart ambari
```
#sandbox
service ambari restart
#non sandbox
service ambari-server restart
```  
  - Start IotDemo service

#### Access webapp

- Open the webapp via Ambari view or at http://sandbox.hortonworks.com:8081/storm-demo-web-app/
  - Login
  - Generate <50 events
  - Navigate to the monitoring and prediction webapps

![Image](../master/screenshots/iot-predictionapp.png?raw=true)

- Check Storm view for metrics
![Image](../master/screenshots/iot-stormview.png?raw=true)

- Alternatively check Storm UI for metrics
![Image](../master/screenshots/storm.png?raw=true)


#### Visualize events using Zeppelin

- If installed, open Zeppelin via the Ambari view or at http://sandbox.hortonworks.com:9995

- Open the "IoT Data Analysis" notebook and execute the cells one by one to:
  - answer business users questions (e.g. does fatigue cause violations? etc)
  ![Image](../master/screenshots/zeppelin-iot-notebook.png?raw=true)
  
  - build a regression model to predict violations
  ![Image](../master/screenshots/zeppelin-iot-model.png?raw=true)


- If you setup the 'spark' queue earlier, verify that Zeppelin submitted application to this queue. See [here](https://github.com/hortonworks-gallery/ambari-zeppelin-service/blob/master/README.md#zeppelin-yarn-integration) for screenshots

- If Ranger is installed, you can also use it to secure Spark by setting authorization policies and getting audit reports. See sample steps/screenshots to (setup Ranger's YARN plugin)[https://github.com/abajwa-hw/security-workshops/blob/master/Setup-ranger-23.md#setup-yarn-plugin-for-ranger] and (setup YARN queue and Ranger policy on an Ambari installed HDP 2.3 cluster)[https://github.com/abajwa-hw/security-workshops/blob/master/Setup-ranger-23.md#yarn-audit-exercises-in-ranger].


#### Troubleshooting

- Components going down? Increase VM memory/cores and restart

- Storm Nimus is not coming up? 
  - or getting error `java.lang.RuntimeException: Could not find leader nimbus from seed hosts [sandbox.hortonworks.com]. Did you specify a valid list of nimbus hosts for config nimbus.seeds`

  - Solution: Stop storm and run below script to clean old data before starting Storm back up
```
setup/bin/cleanupstormdirs.sh

/usr/hdp/current/zookeeper-client/bin/zkCli.sh
rmr /storm	
quit
```

- Other issues? Try resetting demo and restarting
```
setup/bin/cleanup.sh
```

#### Remove the service

- To remove the IOTDEMO service: 
  - Stop the service via Ambari
  - Delete the service
  
```
export SERVICE=IOTDEMO
export PASSWORD=admin
export AMBARI_HOST=localhost

output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`
    
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X DELETE http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#if above errors out, run below first to fully stop the service
#curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

```
    
  - Remove artifacts 
  
```
rm -rf /root/sedev
rm -rf /root/iot*
rm -rf /root/scala
rm -rf /root/maven
rm -rf /var/log/iotdemo.log
rm -rf /var/lib/ambari-server/resources/views/iot*
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
rm -rf /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/IOTDEMO

service ambari-server restart
```

