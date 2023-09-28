# Create a app sample CSB[Camel SpringBoot] app
 
Step1: Generate the basic app from archetype
```
mvn archetype:generate \
 -DarchetypeGroupId=org.apache.camel.archetypes \
 -DarchetypeArtifactId=camel-archetype-spring-boot \
 -DarchetypeVersion=3.18.3.redhat-00042 \
 -DgroupId=com.redhat \
 -DartifactId=csb-app \
 -Dversion=1.0.0 \
 -DinteractiveMode=false
```

Step2: Build the app
```
cd csb-app
mvn clean package
```

Step3: Run the app
```
java -jar target/csb-app-1.0.0.jar
```

# Install AMQ Broker on OpenShift 4.x 

Step1: Install AMQ Operator 

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: amq-broker-rhel8
  namespace: openshift-operators
spec:
  channel: 7.11.x
  installPlanApproval: Automatic
  name: amq-broker-rhel8
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: amq-broker-operator.v7.11.1-opr-2  
```
```sh
oc apply mq-subscription.yaml
```

Step2: Create a namespace
```
oc new-project csb-apps
```

Step3: Create a Broker with console
```yaml
apiVersion: broker.amq.io/v1beta1
kind: ActiveMQArtemis
metadata:
  name: mq-broker
  namespace: csb-apps
spec:
  acceptors:
    - name: all
      expose: true
      port: 61616
      protocols: 'openwire,amqp,stomp,artemis,mqtt'
  deploymentPlan:
    image: placeholder
    jolokiaAgentEnabled: false
    journalType: nio
    managementRBACEnabled: true
    messageMigration: false
    persistenceEnabled: false
    requireLogin: false
    size: 1
  console:
    expose: true
  adminPassword: admin
  adminUser: admin
```
```sh
oc apply -f broker.yaml
```


# Change the csb-app to use the AMQ Broker

Step1: Add the AMQ dependency in the pom.xml

```xml
    <!-- Active MQ -->
    <dependency>
      <groupId>org.apache.camel.springboot</groupId>
      <artifactId>camel-activemq-starter</artifactId>
    </dependency>
```

Step2: Create a new java class [JmsConsumer.java]
```java
package com.redhat;

import javax.jms.ConnectionFactory;
import org.apache.activemq.ActiveMQConnectionFactory;
import org.apache.camel.CamelContext;
import org.apache.camel.builder.RouteBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;

@Component
@Configuration
public class JmsConsumer  extends RouteBuilder {
    
    @Autowired
	CamelContext context;
	
	
	@Override
	public void configure() throws Exception {
				
		from("activemq:queue:order?connectionFactory=#cf")
		.log("first ${body}");
				
	}

	@Bean("cf")
	public ConnectionFactory connectionFactory() {
		return new ActiveMQConnectionFactory();
	}

}

```
Step3: Add the Spring properties 
```sh
# AMQ broker configs 
camel.component.activemq.use-pooled-connection=true
camel.component.activemq.use-single-connection=false
camel.springboot.main-run-controller=true
camel.component.activemq.broker-url=tcp://mq-broker-all-0-svc:61616
camel.component.activemq.username=admin
camel.component.activemq.password=admin
```

# Deploy the csb-app on OpenShift

Step1: Log into OCP

Step2: Go to the app folder 

Step3: Use the project that was created for AMQ
```
oc project csb-apps
```

Step4: Build the image and deploy using maven 
```
mvn clean -DskipTests oc:deploy -Popenshift
```

Step5: Monitor the logs
```
oc logs -f dc/csb-app
```

# Test the app on OpenShift

Step1: Go to the AMQ broker GUI console and log in with the admin/admin credentials 

Step2: Navigate to the "order" queue 

Step3: Send a test message 

Step4: Check the csb-app logs to validate the message consumption
