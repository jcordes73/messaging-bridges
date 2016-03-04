Building / installing the bridge
=========================
This is a messaging bridge for HornetQ-ActiveMQ running on JBoss EAP and features the following:
* JBoss Fuse / A-MQ 6.2.1
* JBoss EAP 6.4
* Camel subsystem on JBoss EAP
* ActiveMQ on JBoss Fuse / A-MQ
* HornetQ on JBoss EAP 6
* ActiveMQ RAR Adapter on JBoss EAP 6
* SSL for JBoss Fuse / A-MQ <--> JBoss EAP communication
* XA

**JBOSS_HOME** refers to the installation directory of JBoss EAP
**AMQ_HOME** refers to the installation directoy of JBoss Fuse / A-MQ
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
Install Camel subsystem on JBoss EAP
----------------------------------------------
* Download Red Hat JBoss Fuse 6.2.1 on EAP Installer from https://access.redhat.com/jbossnetwork/restricted/softwareDownload.html?softwareId=41301 to JBOSS_HOME
* Run java -jar fuse-eap-installer-6.2.1.redhat-084.jar
* Create an admin user
```
$JBOSS_HOME/bin/add-user.sh 

What type of user do you wish to add? 
 a) Management User (mgmt-users.properties) 
 b) Application User (application-users.properties)
(a): a

Enter the details of the new user to add.
Using realm 'ManagementRealm' as discovered from the existing property files.
Username : admin
The username 'admin' is easy to guess
Are you sure you want to add user 'admin' yes/no? yes
Password requirements are listed below. To modify these restrictions edit the add-user.properties configuration file.
 - The password must not be one of the following restricted values {root, admin, administrator}
 - The password must contain at least 8 characters, 1 alphabetic character(s), 1 digit(s), 1 non-alphanumeric symbol(s)
 - The password must be different from the username
Password : 
Re-enter Password : 
What groups do you want this user to belong to? (Please enter a comma separated list, or leave blank for none)[  ]: admin
About to add user 'admin' for realm 'ManagementRealm'
Is this correct yes/no? yes
Added user 'admin' to file 'JBOSS_HOME/standalone/configuration/mgmt-users.properties'
Added user 'admin' to file 'JBOSS_HOME/domain/configuration/mgmt-users.properties'
Added user 'admin' with groups admin to file 'JBOSS_HOME/standalone/configuration/mgmt-groups.properties'
Added user 'admin' with groups admin to file 'JBOSS_HOME/domain/configuration/mgmt-groups.properties'
Is this new user going to be used for one AS process to connect to another AS process? 
e.g. for a slave host controller connecting to the master or for a Remoting connection for server to server EJB calls.
yes/no? no
```
Install ActiveMQ RAR on JBoss EAP
----------------------------------------------
* Download Red Hat JBoss Fuse 6.2.1 on EAP Installer from https://access.redhat.com/jbossnetwork/restricted/softwareDownload.html?softwareId=41301
* Download JBoss A-MQ from https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=jboss.amq and unzip **jboss-a-mq-6.2.1.redhat-084.zip**. This will create a folder **jboss-a-mq-6.2.1.redhat-084** which I will refer to as **AMQ_HOME** from now on.
* Install ActiveMQ RAR Adapter
```bash
unzip -j AMQ_HOME/extras/apache-activemq-5.11.0.redhat-621084-bin.zip apache-activemq-5.11.0.redhat-621084/lib/optional/activemq-rar-5.11.0.redhat-621084.rar -d  JBOSS_HOME/standalone/deployments
```
* Copy broker trust store
```bash
cp AMQ_HOME/etc/broker.ts JBOSS_HOME/standalone/configuration/
```
* Create a property file **JBOSS_HOME/standalone/configuration/config.properties**
```bash
javax.net.ssl.trustStore=standalone/configuration/broker.ts
javax.net.ssl.trustStorePassword=redhat
hornetq.user=test
hornetq.password=test123
activemq.user=admin
activemq.password=admin
activemq.serverUrl=failover:(ssl://localhost:61616)?jms.rmIdFromConnectionId=true&maxReconnectAttempts=0
```
* When the failover-protocol is used some additional changes need to be done, see https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_A-MQ/6.2/html-single/Integrating_with_JBoss_Enterprise_Application_Platform/index.html#DeployRar-Failover-ExRACF

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
* Start JBoss EAP
```bash
JBOSS_HOME/bin/standalone.sh -P standalone/configuration/config.properties -c standalone-full.xml > nohup.out 2>&1 &
```
* Configure ActiveMQ RAR Adapter and queues via JBoss CLI
```bash
if (outcome == success) of /subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar":read-resource()
  /subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar":remove()
end-if
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar":add(archive="activemq-rar-5.11.0.redhat-621084.rar",transaction-support="XATransaction")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar"/config-properties="Password":add(value="${activemq.password}")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar"/config-properties="UserName":add(value="${activemq.user}")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar"/config-properties="ServerUrl":add(value="${activemq.serverUrl}")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar"/connection-definitions="ActiveMQConnectionFactoryPool":add(class-name="org.apache.activemq.ra.ActiveMQManagedConnectionFactory",enabled="true",jndi-name="java:/AMQConnectionFactory",max-pool-size="10",min-pool-size="1",no-recovery="false",pool-prefill="false",recovery-password="${activemq.password}",recovery-plugin-class-name="org.jboss.jca.core.recovery.ConfigurableRecoveryPlugin",recovery-plugin-properties={"EnableIsValid" => "false","IsValidOverride" => "true","EnableClose" => "true"},recovery-username="${activemq.user}",same-rm-override="false",use-ccm="true",use-java-context="true")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar"/admin-objects="queueA":add(class-name="org.apache.activemq.command.ActiveMQQueue",jndi-name="java:/queue/queueA",use-java-context="true")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar"/admin-objects="queueA"/config-properties="PhysicalName":add(value="queueA")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar"/admin-objects="queueC":add(class-name="org.apache.activemq.command.ActiveMQQueue",jndi-name="java:/queue/queueC",use-java-context="true")
/subsystem="resource-adapters"/resource-adapter="activemq-rar-5.11.0.redhat-621084.rar"/admin-objects="queueC"/config-properties="PhysicalName":add(value="queueC")
/subsystem=messaging/hornetq-server=default/jms-queue=queueB:add(entries=["queueB","java:/queue/queueB"])
```
* Securing HornetQ
```bash
/subsystem=messaging/hornetq-server=default:write-attribute(name=security-enabled,value=true)
/subsystem=messaging/hornetq-server=default:write-attribute(name=cluster-user,value=${hornetq.user})
/subsystem=messaging/hornetq-server=default:write-attribute(name=cluster-password,value=${hornetq.password})
```
The JBoss CLI commands can also be executed in a batch like this
```bash
JBOSS_HOME/bin/jboss-cli.sh -c --file=src/main/resources/jboss/config/setup.cli
```
Maven build
----------------
* To trigger the build execute
```bash
mvn clean install
```
* To deploy the application execute
```bash
mvn jboss-as:deploy
```
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
