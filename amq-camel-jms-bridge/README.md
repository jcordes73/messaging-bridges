Building / installing the bridge
=========================
This is a messaging bridge for HornetQ-ActiveMQ and features the following:
* JBoss Fuse / A-MQ 6.1
* JBoss EAP 6.x (at least 6.1.1 required)
* ActiveMQ on JBoss Fuse / A-MQ
* HornetQ on JBoss EAP 6
* ActiveMQ RAR Adapter on JBoss EAP 6
* SSL for JBoss Fuse / A-MQ <--> JBoss EAP communication
* XA
* Karaf feature-file for easy deployment to JBoss Fuse / A-MQ

JBOSS_HOME refers to the installation directory of JBoss EAP
AMQ_HOME refers to the installation directoy of JBoss Fuse
Maven build
----------------
* To trigger the build execute
```bash
mvn clean install
```
* To install the features into the local Maven repository execute
```bash
mvn org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file -Dfile=src/main/resources/amq/features/amq-camel-jms-bridge-1.0.0-features.xml -DgroupId=com.redhat -DartifactId=amq-camel-jms-bridge -Dversion=1.0.0 -Dpackaging=xml -Dclassifier=features
```
Configuration
------------------

* Create keystore/truststore (https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_A-MQ/6.1/html-single/Security_Guide/index.html#SSL-JavaKeystores)
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
Is CN=Jochen Cordes, OU=RHC, O=Red Hat, L=Frankfurt, ST=NRW, C=de correct?  
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
Owner: CN=Jochen Cordes, OU=RHC, O=Red Hat, L=Frankfurt, ST=NRW, C=de  
  Issuer: CN=Jochen Cordes, OU=RHC, O=Red Hat, L=Frankfurt, ST=NRW, C=de  
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
* Configure trust-store and key-store for the Openwire/SSL Protocol (https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_A-MQ/6.1/html-single/Security_Guide/index.html#SSL-SetSecurityContext) in AMQ_HOME/etc/activemq.xml
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
features:addurl mvn:com.redhat/amq-camel-jms-bridge/1.0.0/xml/features
features:refreshurl mvn:com.redhat/amq-camel-jms-bridge/1.0.0/xml/features
features:removeurl mvn:com.redhat/amq-camel-jms-bridge/1.0.0/xml/features
```
* Install the bridge via
```bash
features:install -v amq-camel-jms-bridge
```
Install ActiveMQ RAR in JBoss EAP
----------------------------------------------
* Download JBoss A-MQ from https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=jboss.amq and unzip jboss-a-mq-6.1.0.redhat-379.zip. This will create a folder jboss-a-mq-6.1.0.redhat-379 which I will refer to as AMQ_HOME from now on.
* Install ActiveMQ RAR Adapter
```bash
unzip AMQ_HOME/extras/apache-activemq-5.9.0.redhat-610379-bin.zip apache-activemq-5.9.0.redhat-610379/lib/optional/activemq-rar-5.9.0.redhat-610379.rar -d  JBOSS_HOME/standalone/deployments
```
* Copy broker trust store
```bash
cp AMQ_HOME/etc/broker.ts JBOSS_HOMEstandalone/configuration/
```
* Create a property file JBOSS_HOMEstandalone/configuration/config.properties
```bash
javax.net.ssl.trustStore=standalone/configuration/broker.ts
javax.net.ssl.trustStorePassword=redhat
hornetq.user=test
hornetq.password=test123-
activemq.user=admin
activemq.password=redhat
activemq.serverUrl=failover:(ssl://localhost:61616)?jms.rmIdFromConnectionId=true&maxReconnectAttempts=0
```
* Start JBoss EAP
```bash
JBOSS_HOME/bin/standalone.sh -P standalone/configuration/config.properties -c standalone-full.xml > nohup.out 2>&1 &
```
* Configure ActiveMQ RAR Adapter via JBoss CLI
```bash
if (outcome == success) of :read-resource()
  /subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar":remove()
end-if
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar":add(archive="activemq-rar-5.9.0.redhat-610379.rar",transaction-support="XATransaction")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar"/config-properties="Password":add(value="${activemq.password}")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar"/config-properties="UserName":add(value="${activemq.user}")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar"/config-properties="ServerUrl":add(value="${activemq.serverUrl}")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar"/admin-objects="queueA":add(class-name="org.apache.activemq.command.ActiveMQQueue",jndi-name="java:/queue/queueA",use-java-context="true")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar"/admin-objects="queueA"/config-properties="PhysicalName":add(value="queueA")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar"/admin-objects="topicA":add(class-name="org.apache.activemq.command.ActiveMQTopic",jndi-name="java:/topic/topicA",use-java-context="true")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar"/admin-objects="topicA"/config-properties="PhysicalName":add(value="topicA")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.9.0.redhat-610379.rar"/connection-definitions="ActiveMQConnectionFactoryPool":add(class-name="org.apache.activemq.ra.ActiveMQManagedConnectionFactory",enabled="true",jndi-name="java:/AMQConnectionFactory",max-pool-size="10",min-pool-size="1",no-recovery="false",pool-prefill="false",recovery-password="${activemq.password}",recovery-plugin-class-name="org.jboss.jca.core.recovery.ConfigurableRecoveryPlugin",recovery-plugin-properties={"EnableIsValid" => "false","IsValidOverride" => "true","EnableClose" => "true"},recovery-username="${activemq.user}",same-rm-override="false",use-ccm="true",use-java-context="true")
```
* When the failover-protocol is used some additional changes need to be done, see https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_A-MQ/6.1/html-single/Integrating_with_JBoss_Enterprise_Application_Platform/index.html#DeployRar-Failover-ExRACF

```bash
failover:(tcp://amqhostA:61616,tcp://amqhostB:61616)?jms.rmIdFromConnectionId=true&maxReconnectAttempts=0
```
* Occasionally you will see the following error
```bash
20:16:09,146 WARN  [com.arjuna.ats.jta] (Periodic Recovery) ARJUNA016027: Local XARecoveryModule.xaRecovery got XA exception XAException.XAER_RMERR: javax.transaction.xa.XAException: Failover transport not connected: unconnected
    at org.apache.activemq.TransactionContext.recover(TransactionContext.java:656)
    at org.apache.activemq.ra.LocalAndXATransaction.recover(LocalAndXATransaction.java:135)
    at org.jboss.jca.core.tx.jbossts.XAResourceWrapperImpl.recover(XAResourceWrapperImpl.java:177)
    at com.arjuna.ats.internal.jta.recovery.arjunacore.XARecoveryModule.xaRecoveryFirstPass(XARecoveryModule.java:548) [jbossjts-jacorb-4.17.21.Final-redhat-2.jar:4.17.21.Final-redhat-2]
    at com.arjuna.ats.internal.jta.recovery.arjunacore.XARecoveryModule.periodicWorkFirstPass(XARecoveryModule.java:187) [jbossjts-jacorb-4.17.21.Final-redhat-2.jar:4.17.21.Final-redhat-2]
    at com.arjuna.ats.internal.arjuna.recovery.PeriodicRecovery.doWorkInternal(PeriodicRecovery.java:743) [jbossjts-jacorb-4.17.21.Final-redhat-2.jar:4.17.21.Final-redhat-2]
    at com.arjuna.ats.internal.arjuna.recovery.PeriodicRecovery.run(PeriodicRecovery.java:371) [jbossjts-jacorb-4.17.21.Final-redhat-2.jar:4.17.21.Final-redhat-2]
```
This issue is logged as [ENTMQ-715] ARJUNA016027: Local XARecoveryModule.xaRecovery got XA exception XAException.XAER_RMERR: javax.transaction.x…

* Add queues/topics
	* In **JBOSS_HOME/configuration/standalone-full.xml** 
```xml
<admin-objects>
  <admin-object
    class-name="org.apache.activemq.command.ActiveMQQueue"
    jndi-name="java:/queue/queueA"
    use-java-context="true"
    pool-name="queueA">
    <config-property name="PhysicalName">
      queueA
    </config-property>
  </admin-object>
  <admin-object
    class-name="org.apache.activemq.command.ActiveMQTopic"
    jndi-name="java:/topic/topicA"
    use-java-context="true"
    pool-name="topicA">
    <config-property name="PhysicalName">
      topicA
    </config-property>
  </admin-object>
</admin-objects>
```
 or via JBoss CLI
```bash
/subsystem=resource-adapters/resource-adapter=activemq-rar-5.9.0.redhat-610379.rar/admin-objects=queueA:add(class-name=org.apache.activemq.command.ActiveMQQueue, jndi-name="java:/queue/queueA", use-java-context=true)
/subsystem=resource-adapters/resource-adapter=activemq-rar-5.9.0.redhat-610379.rar/admin-objects=queueA/config-properties=PhysicalName:add(value=queueA)

/subsystem=resource-adapters/resource-adapter=activemq-rar-5.9.0.redhat-610379.rar/admin-objects=topicA:add(class-name=org.apache.activemq.command.ActiveMQTopic, jndi-name="java:/topic/topicA", use-java-context=true)

/subsystem=resource-adapters/resource-adapter=activemq-rar-5.9.0.redhat-610379.rar/admin-objects=topicA/config-properties=PhysicalName:add(value=topicA)
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
* Add trust-store to Karaf (in bin/setenv)
```bash
KARAF_OPTS="-Djavax.net.ssl.trustStore=/NotBackedUp/jcordes/mobiliar/jboss-fuse-6.1.0.redhat-379/etc/broker.ts -Djavax.net.ssl.trustStorePassword=redhat"
```




