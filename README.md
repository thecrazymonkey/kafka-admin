# Kafka-Admin
Managing topics and acls at scale with Apache Kafka or Confluent Cloud can be a particularly challenging endeavor. 
Kafka Admin was originally created to help address this challenge, particularly with managing ACLs for multi-tenant Kafka clusters. 
Kafka Admin leverages the AdminClient APIs of Apache Kafka to programmatically create topics, as well as add/modify/delete ACLs, from an input configuration file (YAML). 
Kafka Admin leverages the Metadata Service APIs of Confluent Server to programmatically to add/modify/delete rolebindings, from an input configuration file (YAML). 

Once the jar is compiled, using the application is as simple as supplying cluster configuration from a properties file and providing a topic/ACL/Rolebindings YAML configuration. 
If you already have an active cluster with topics, ACLs and RoleBindings configured, you can use Kafka-Admin with the `dump` flag to output the cluster's configuration file (YAML). 

Some of the use cases that have arisen which can leverage Kafka-Admin include:
* Managing RoleBindings, ACLs & Topics at scale with source controlled configurations
* Migrating RoleBindings, ACLs &/or Topics from one cluster to another by combining the `pull` mechanism of Kafka-Admin against the source cluster and applying the resulting output to the target cluster.
* Automating ACLs & Topics with Confluent Cloud (otherwise requires using non-scriptable CLI)

## Using Kafka-Admin  
[Update Your Configuration File](#update-your-configuration-file)

[Build the Application Jar](#build-the-application-jar-file)

[Run the Jar](#run-the-jar)  

## Update your configuration files
You can use the supplied `kafka-admin.properties.example` as a base. Update and rename to `kafka-admin.properties`.
Alternatively, you can use this application to pull the topic & ACL configurations from an existing cluster. 

### Add Cluster Connection Configuration

In your `kafka-admin.properties` file, add your cluster connection configuration as shown here:

```
bootstrap.servers: pkc-l7p2j.us-west-2.aws.confluent.cloud:9092
security.protocol: SASL_SSL
sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username="<API-KEY>" password="<API-SECRET>";
sasl.mechanism: PLAIN
ssl.endpoint.identification.algorithm: https
```

Any cluster connection connection properties may be specified here, just make sure they align to the actual Kafka client properties exactly. 
For example, you can add `ssl.enabled.mechanisms`. 
Any properties that are not specified or are left with a blank value will use the default values. 

### Add/Update Topics

In your `config.yml` file, update the section under "topics" with your desired topics as shown here:

```
topics:
  matt-topic-1:
    name: matt-topic-1
    replication.factor: 1
    partitions: 3
    cleanup.policy: 
    compression.type: "lz4"
    retention.ms: "0"
```

You can choose to leave the additional configurations for a topic blank (and defaults will be used), or you can enter them as well under the desired topic. 
It is important to note that the **additional configuration values must be entered as strings (wrapped in double quotes)** or they will cause an error to the application. 

### Delete Topics

If you are you using delete topics (default is disabled), you will need to make sure all topics that exist on the cluster are in your "topics" section or exist in the "default_topics" section, as shown here:

```
default_topics:
  - my_topic_do_not_delete
  - this_stays_too
  - ^{1,2}_confluent-.*
  - ^connect-cluster-.*
```

All (truly) internal topics are ignored by the delete process, for the rest use a list or regex pattern as shown above.
All topic delete operations must be explicitly enabled using `delete` flag (-d).
If you're not using delete topics, then you do not need to worry about using the "default_topics" section. 

#### Increasing Topic Partitions

For a topic that already exists on the cluster, you can increase the number of partitions that the topic has by updating the configuration for the topic under the "topics" section.
You can only increase partitions -- **there is no ability to remove or reduce partitions.** 

Note that at this time (Jan 2020), no other topic configurations can be modified by updating the config file. 

### Add/Update/Remove ACLs

In your `config.yml` file, update the section under "acls" with your complete list of acls as shown in the examples below:

#### ACLs to Produce to topic(s)

To produce to a topic, the principal of the producer will require the **WRITE** operation on the **topic** resource.

Example 1 -- ACLs needed for Service Account user "12576" to produce to a Topic named "griz-test" from any host:

```
acls:
    .
    .
    .
  project-xyz:
    resource-type: TOPIC
    resource-name: griz-test
    resource-pattern: LITERAL
    principal: User:12576
    host: '*'
    operation: WRITE
    permission: ALLOW
    .
    .
    . 
```

Example 2 -- ACLs needed for user "12576" to produce to **any topic whose name starts with "griz-"** from any host:

```
acls:
    .
    .
    .
  project-xyz:
    resource-type: TOPIC
    resource-name: griz-
    resource-pattern: PREFIXED
    principal: User:12576
    host: '*'
    operation: WRITE
    permission: ALLOW
    .
    .
    . 
```

_Producers may be configured with enable.idempotence=true to ensure that exactly one copy of each message is written to the stream. 
The principal used by idempotent producers must be authorized to perform IdempotentWrite on the cluster._

Example 3 -- ACLs needed for user "12576" to produce idempotently to a Topic named "griz-test" from any host:

```
acls:
    .
    .
    .
  project-xyz-1:
    resource-type: TOPIC
    resource-name: griz-test
    resource-pattern: LITERAL
    principal: User:12576
    host: '*'
    operation: WRITE
    permission: ALLOW
  project-xyz-2:
    resource-type: TOPIC
    resource-name: griz-test
    resource-pattern: LITERAL
    principal: User:12576
    host: '*'
    operation: IDEMPOTENT-WRITE
    permission: ALLOW
    .
    .
    . 
```

_Producers may also be configured with a non-empty transactional.id to enable transactional delivery with reliability semantics that span multiple producer sessions. 
The principal used by transactional producers must additionally be authorized for **Describe** and **Write** operations on the configured transactional.id._

Example 4 -- ACLs needed for user "12576" to produce using a transactional producer with transactional.id=test-txn to a Topic named "griz-test" from any host:

```
acls:
    .
    .
    .
  project-xyz-1:
    resource-type: TOPIC
    resource-name: griz-test
    resource-pattern: LITERAL
    principal: User:12576
    host: '*'
    operation: WRITE
    permission: ALLOW
  project-xyz-2:
    resource-type: TRANSACTIONALID
    resource-name: test-txn
    resource-pattern: LITERAL
    principal: User:12576
    host: '*'
    operation: DESCRIBE
    permission: ALLOW
  project-xyz-3:
    resource-type: TRANSACTIONALID
    resource-name: test-txn
    resource-pattern: LITERAL
    principal: User:12576
    host: '*'
    operation: WRITE
    permission: ALLOW
    .
    .
    . 
```

#### ACLs to Consume from topic(s)

To consume from a topic, the principal of the consumer will require the **READ** operation on the **topic** and **group** resources.

Example 1 -- ACLs needed for Service Account user "12576" to read from a topic named "griz-test" as any consumer group from any host:

```
acls:
    .
    .
    .
  project-xyz-1:
    resource-type: TOPIC
    resource-name: griz-test
    resource-pattern: LITERAL
    principal: User:12576
    host: '*'
    operation: READ
    permission: ALLOW
  project-xyz-2:
      resource-type: GROUP
      resource-name: '*'
      resource-pattern: LITERAL
      principal: User:12576
      host: '*'
      operation: READ
      permission: ALLOW
    .
    .
    . 
```

Example 2 -- ACLs needed for user "12576" to consume from **any topic whose name starts with "griz-"** as any consumer group from any host:

```
acls:
    .
    .
    .
  project-xyz-1:
    resource-type: TOPIC
    resource-name: griz-
    resource-pattern: PREFIXED
    principal: User:12576
    host: '*'
    operation: WRITE
    permission: ALLOW
  project-xyz-2:
      resource-type: GROUP
      resource-name: '*'
      resource-pattern: LITERAL
      principal: User:12576
      host: '*'
      operation: READ
      permission: ALLOW
    .
    .
    . 
```

For a list of all operations & resources, see this [link](https://docs.confluent.io/current/kafka/authorization.html#acl-format "Confluent Docs").

**Note that all fields are required for ACLs (unlike topics) in your config.yml file. 
Any ACLs that are not included in your list but that exist on the Kafka cluster will be removed when the application is run.** 

#### Adding multiple acls with a single acl configuration block:

You can add multiple acls using the same configuration block under acls. 
These can be easily grouped together into a project heading. 

Example -- The following block will create acls that give 'READ' and 'WRITE' to Matt, Brien, Aleks, Vedanta, and Liying for topics 'matt-topic-A','brien-topic', and 'third-topic':

```
acls:
  project-1:
    resource-type: topic
    resource-name: matt-topic-A, brien-topic, third-topic
    resource-pattern: LITERAL
    principal: User:Matt, User:Brien, User:Aleks, User:Vedanta,User:Liying
    operation: READ, WRITE
    permission: ALLOW
    host: '*'
```

### Add/Update/Remove RoleBindings - Confluent Platform Only

In your `config.yml` file, update the section under "rolebindings" with your complete list of rolebindings as shown in the examples below.
Notice that for a combination of principal, role and scope you can add the resource specification as an additional item to already existing specification.
Correspondingly you can just remove resourceType item and it will get deleted without impacting the rest of the rolebinding specification.
To find the Kafka Cluster id you can utilize Zookeeper shell ```zookeeper-shell localhost:2181 get /cluster/id```

```
rolebindings:
  RoleBinding-22:
    principal: User:clientc
    role: DeveloperRead
    resource:
      - resourceType: Connector
        name: datagen-pageviews
        patternType: LITERAL
    scope:
      clusters:
        kafka-cluster: ECBwt-DmSe-WkKNg2dxdXg
        connect-cluster: connect-cluster
  RoleBinding-24:
    principal: User:connect
    role: ResourceOwner
    resource:
      - resourceType: Topic
        name: connect-configs
        patternType: LITERAL
      - resourceType: Topic
        name: connect-statuses
        patternType: LITERAL
      - resourceType: Group
        name: connect-cluster
        patternType: LITERAL
      - resourceType: Group
        name: secret-registry
        patternType: LITERAL
      - resourceType: Topic
        name: connect-offsets
        patternType: LITERAL
      - resourceType: Topic
        name: _confluent-secrets
        patternType: LITERAL
    scope:
      clusters:
        kafka-cluster: ECBwt-DmSe-WkKNg2dxdXg
  RoleBinding-25:
    principal: User:MySystemAdmin
    role: SystemAdmin
    scope:
      clusters:
        kafka-cluster: ECBwt-DmSe-WkKNg2dxdXg
```

### Add/Update/Remove Centralized AclBindings - Confluent Platform Only

In your `config.yml` file, update the section under "aclbindings" with your complete list of aclbindings as shown in the examples below.
Notice that for a combination of a resource pattern (composed of resource type, name and type) and scope (cluster ID)  you 
can add specific Acl rules composed of principal (User: or Group:), permission type, operation and host. 
Correspondingly you can just remove Acl rule item and it will get deleted without impacting the rest of the Centralized Aclbinding specification.
To find the Kafka Cluster id you can utilize Zookeeper shell ```zookeeper-shell localhost:2181 get /cluster/id```

```
aclbindings:
  AclBinding-1:
    resourcePattern:
      resourceType: Topic
      name: test
      patternType: LITERAL
    scope:
      clusters:
        kafka-cluster: ghUubZZ1R5-yc_6uhi-1pw
    aclRules:
      - principal: User:clienta
        permissionType: ALLOW
        host: '*'
        operation: Create
      - principal: User:clientc
        permissionType: ALLOW
        host: '*'
        operation: Describe
  AclBinding-2:
    resourcePattern:
      resourceType: Cluster
      name: kafka-cluster
      patternType: LITERAL
    scope:
      clusters:
        kafka-cluster: ghUubZZ1R5-yc_6uhi-1pw
    aclRules:
      - principal: User:clienta
        permissionType: ALLOW
        host: '*'
        operation: Describe
```

## Build the application jar file

From the project root directly, run the following:

`mvn clean package`

## Run the Jar

### Pull the configured topics, ACLs & RoleBindings from a cluster **and print to stdout**

Supply your connection configuration with the "-properties" or "-p" and "-dump" options to print out current topic/ACLs/RoleBindings configuration to stdout
Note - you need to use "-r" parameter to enable the RBAC related functionality.
Note - you need to use "-cacl" parameter to enable the Centralized ACLs related functionality.

`java -jar <path-to-jar>  -properties <path.properties> -dump -r -cacl`

### Pull the configured topics, ACLs & RoleBindings from a cluster **and write to an output file**

Supply your connection configuration with the "-properties" or "-p", "-dump" and "-output" or "-o" options to print out current topic/ACLs/RoleBindings configuration to a file
Note - you need to use "-r" parameter to enable the RBAC related functionality.
Note - you need to use "-cacl" parameter to enable the Centralized ACLs related functionality.

`java -jar <path-to-jar>  -properties <path.properties> -dump -output <path-output.yml> -r -cacl`

### Generate a Topic, ACL & RoleBindings Plan but **do not** apply any changes

Supply your connection configuration with the "-properties" or "-p" option and your topic/ACL configuration with "-config" or "-c" option to generate your topic & ACL plans
Note - you need to use "-r" parameter to enable the RBAC related functionality.
Note - you need to use "-cacl" parameter to enable the Centralized ACLs related functionality.

`java -jar <path-to-jar> -properties <path.properties> -config <path-config.yml> -r -cacl` 

### Generate a Topic, ACL & RoleBindings Plan, then apply the changes

Supply your configuration as above & adding the "-execute" option to actually execute the plans
Note - you need to use "-r" parameter to enable the RBAC related functionality.
Note - you need to use "-cacl" parameter to enable the Centralized ACLs related functionality.

`java -jar <path-to-jar>  -properties <path.properties> -config <path-config.yml> -execute -r -cacl`

### Generate a Topic & ACL Plan, then apply the changes and allow delete topics operation

Supply your configuration as above & adding the "-execute -delete" options to actually execute the plans including the delete operation

`java -jar <path-to-jar>  -properties <path.properties> -config <path-config.yml> -execute -delete`
