Building / installing the bridge
=========================
This is a messaging bridge for HornetQ-ActiveMQ running on JBoss Fuse / A-MQ and features the following:
* JBoss Fuse / A-MQ 6.2.1
* JBoss EAP 6.x (at least 6.1.1 required)
* ActiveMQ on JBoss Fuse / A-MQ
* HornetQ on JBoss EAP 6
* SSL for JBoss Fuse / A-MQ <--> JBoss EAP communication
* XA
* Karaf feature-file for easy deployment to JBoss Fuse / A-MQ

**JBOSS_HOME** refers to the installation directory of JBoss EAP
**AMQ_HOME** refers to the installation directoy of JBoss Fuse
Maven build
----------------
* To trigger the build execute
```bash
mvn clean install
```
* To install the features into the local Maven repository execute
```bash
mvn org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=src/main/resources/amq/features/amq-camel-jms-bridge-2.0.0-features.xml -DgroupId=com.redhat -DartifactId=amq-camel-jms-bridge -Dversion=2.0.0 -Dpackaging=xml -Dclassifier=features
```
Configuration
------------------

* Create keystore/truststore (https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_A-MQ/6.2/html-single/Security_Guide/index.html#SSL-JavaKeystores)
```bash
keytool -keystore broker.ks -genkey -alias broker  
Enter keystore password:  
Re-enter new password:  
What is your first and last name?  
  [Unknown]:  Jochen Cordes  
What is the name of your organizational unit?  
  [Unknown]:  RHC  
What is the name of your organization?  
  [Unknown]:  Red Hat  
What is the name of your City or Locality?  
  [Unknown]:  Frankfurt  
What is the name of your State or Province?  
  [Unknown]:  NRW  
What is the two-letter country code for this unit?  
  [Unknown]:  de  
Is CN=Jochen Cordes, OU=RHC, O=Red Hat, L=Frankfurt, ST=Hessen, C=de correct?  
  [no]:  yes  
Enter key password for <broker>  
  (RETURN if same as keystore password):  
```
* Export certificate
```bash
keytool -keystore broker.ks -export -alias broker -keyalg rsa -file broker.cer
```
* Import certificate into trust-store
```bash
keytool -import -file broker.cer -alias brokerCA -keystore broker.ts  
Enter keystore password:  
Re-enter new password:  
Owner: CN=Jochen Cordes, OU=RHC, O=Red Hat, L=Frankfurt, ST=Hessen, C=de  
  Issuer: CN=Jochen Cordes, OU=RHC, O=Red Hat, L=Frankfurt, ST=Hessen, C=de  
  Serial number: 5433d09c  
  Valid from: Tue Oct 07 13:38:04 CEST 2014 until: Mon Jan 05 12:38:04 CET 2015  
  Certificate fingerprints:  
      MD5:  95:EC:D1:6C:AA:E0:B8:84:E6:18:01:2F:10:97:F2:7D  
      SHA1: 94:CE:D4:B3:22:9B:88:37:45:6A:02:56:4C:44:F2:28:42:92:8E:A1  
      Signature algorithm name: SHA1withDSA  
      Version: 3  
Trust this certificate? [no]:  yes  
  Certificate was added to keystore  
```
* Import default keystore into broker keystore 
```bash
keytool -importkeystore -srckeystore /usr/lib/jvm/jre/lib/security/cacerts -destkeystore broker.ts
```
* Configure trust-store and key-store for the Openwire/SSL Protocol (https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_A-MQ/6.2/html-single/Security_Guide/index.html#SSL-SetSecurityContext) in AMQ_HOME/etc/activemq.xml
```xml
<sslContext>  
  <sslContext keyStore="file:${karaf.home}/etc/broker.ks" keyStorePassword="redhat" trustStore="file:${karaf.home}/etc/broker.ts" trustStorePassword="redhat"/>  
</sslContext>  
      
<transportConnectors>  
  <transportConnector name="ssl" uri="ssl://0.0.0.0:61616?maximumConnections=1000"/>  
</transportConnectors>  
```
* Copy **src/main/resources/amq/config/etc/com.redhat.amq.camel.jms.bridge.cfg** to **AMQ_HOME/etc** and modify it
* On the Karaf shell use the following commands to add, refresh or remove the feature file
```bash
features:addurl mvn:com.redhat/amq-camel-jms-bridge/2.0.0/xml/features
features:refreshurl mvn:com.redhat/amq-camel-jms-bridge/2.0.0/xml/features
features:removeurl mvn:com.redhat/amq-camel-jms-bridge/2.0.0/xml/features
```
* Install the bridge via
```bash
features:install -v amq-camel-jms-bridge
```
Configure JBoss EAP
----------------------------------------------
* Copy broker trust store
```bash
cp AMQ_HOME/etc/broker.ts JBOSS_HOME/standalone/configuration/
```
* Create a property file **JBOSS_HOME/standalone/configuration/config.properties**
```bash
javax.net.ssl.trustStore=standalone/configuration/broker.ts
javax.net.ssl.trustStorePassword=redhat
hornetq.user=test
hornetq.password=test123-
```
* Start JBoss EAP
```bash
JBOSS_HOME/bin/standalone.sh -P standalone/configuration/config.properties -c standalone-full.xml > nohup.out 2>&1 &
```
Securing HornetQ
------------------------
```bash
/subsystem=messaging/hornetq-server=default:write-attribute(name=security-enabled,value=true)

/subsystem=messaging/hornetq-server=default:write-attribute(name=cluster-user,value=${hornetq.user})

/subsystem=messaging/hornetq-server=default:write-attribute(name=cluster-password,value=${hornetq.password})
```

Remote XA for HornetQ (with SSL)
----------------------------------------------
* Add a new connection-factory
```xml
<connection-factory name="RemoteXAConnectionFactory">
  <factory-type>XA_GENERIC</factory-type>
  <connectors>
    <connector-ref connector-name="netty"/>
  </connectors>
  <entries>
    <entry name="java:jboss/exported/jms/RemoteXAConnectionFactory"/>
  </entries>
</connection-factory>
```
* Override netty connector/acceptor
```xml
<connectors>
  ...
  <netty-connector name="netty" socket-binding="messaging">
    <param key="ssl-enabled" value="true"/>
    <param key="key-store-path" value="${jboss.server.config.dir}/broker.ks"/>
    <param key="key-store-password" value="redhat"/>
  </netty-connector>
...
</connectors>

<acceptors>
...
  <netty-acceptor name="netty" socket-binding="messaging">
    <param key="ssl-enabled" value="true"/>
    <param key="key-store-path" value="${jboss.server.config.dir}/broker.ks"/>
    <param key="key-store-password" value="redhat"/>
    <param key="trust-store-path" value="${jboss.server.config.dir}/broker.ts"/>
    <param key="trust-store-password" value="redhat"/>
  </netty-acceptor>
...
</acceptors>
```
* Add trust-store to Karaf (in **bin/setenv**)
```bash
KARAF_OPTS="-Djavax.net.ssl.trustStore=$KARAF_HOME/etc/broker.ts -Djavax.net.ssl.trustStorePassword=redhat"
```
(assuming you have set **KARAF_HOME** in **bin/setenv** as well)

Testing
-------

JBoss A-MQ includes a test client in it's extras folder that can be used to produce and receive messages for testing purposes.

* Produce 100 messages on queue A
```bash
java -Djavax.net.ssl.keyStore=$KARAF_HOME/etc/broker.ks -Djavax.net.ssl.keyStorePassword=redhat -Djavax.net.ssl.trustStore=$KARAF_HOME/etc/broker.ts -Djavax.net.ssl.trustStorePassword=redhat -jar extras/mq-client.jar producer --user admin --password redhat --brokerUrl "ssl://localhost:61616" --destination queue://queueA --count 100  
```
* Read 100 messages from queue C
```bash
java -Djavax.net.ssl.keyStore=$KARAF_HOME/etc/broker.ks -Djavax.net.ssl.keyStorePassword=redhat -Djavax.net.ssl.trustStore=$KARAF_HOME/etc/broker.ts -Djavax.net.ssl.trustStorePassword=redhat -jar extras/mq-client.jar consumer --user admin --password redhat --brokerUrl "ssl://localhost:61616" --destination queue://queueC --count 100  
```
