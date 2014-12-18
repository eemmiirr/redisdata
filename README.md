Redis Data
==========

Redis Data is a Transaction Manager and Object Mapper for redis implemented in Java 6. It simplifies the usage of redis in java and abstracts the way data is mapped. All data deserialization/marshalling are lazy which means they will only happen when the data is accessed.  

Features
--------
- Automated data mapping between redis keys/values and java entities
- Configurable data mappers
- Configurable clients (Currently only jedis supported)
- Configurable AOP/Interception frameworks (Currently supported: AspectJ, Spring AOP)
- Transaction and Pipeline execution without usage of callbacks
- Transparent usage regardless of the type of session (plain, transaction, pipelining. transaction and pipelining combined)
- Data type enforcing
- All exceptions are Unchecked even the Checked ones
- Support for nested transactions (EXPERIMENTAL - Minimum redis connection pool size must be at least the size of the debth of nested transactions or it will end up in a dead lock. Semaphore has to be implemented to solve this problem.)

Minimum requirements
--------------------
- Java >= 1.6
- Spring (dynamic proxy approach) >= 2.5
- aspectj-maven-plugin (compile time weaving approach) >= 1.4
- jedis => 2.3.0

Dependency
----------
```xml
<dependency>
    <groupId>com.github.eemmiirr.redisdata</groupId>
    <artifactId>redisdata-core</artifactId>
    <version>0.7</version>
</dependency>
```

## How it works
### The library consists of 5 main parts:
1. Signalizer (AOP, dynamic proxy etc.)
2. Transaction manager 
3. Session factory
4. Bindings
5. Data mappers (jaxb, jackson json, jdk serialization etc.)
 
### Signalizer
The Signalizer, with the help of a aspect or an interception library, sends signals to the Transaction Manager when to open/close/discard a connection/transaction/pipeline. It basically binds to the execution of the method which should be managed.
 
### Transaction manager
The transaction manager manages the state of the connection and is always coupled with a Signalizer from which it receives the instructions to start, stop or abort a connection/transaction/pipeline. 

### Session factory
The session factory basically creates Sessions. It's always coupled with a transaction manager from which the factory retrieves the connection bound to the current thread. The created session is always bound to a specific managed method by Redis Data, a key data mapper and a value data mapper. This ensures that the data types of the keys and values are always the same.

### Bindings
Bindings basically bind session factories with data mappers and command types.  

### Data mappers
Data mappers are bound to a java type. They tell Redis Data how to serialize/deserialize, marshall/unmarshall the data. 


How to use
----------
#### Define a connection poll and transaction manager
```xml
 <bean id="poolConfig" class="org.apache.commons.pool2.impl.GenericObjectPoolConfig">
    <property name="maxIdle" value="5" />
    <property name="minIdle" value="1" />
    <property name="testOnBorrow" value="true" />
    <property name="testOnReturn" value="true" />
    <property name="testWhileIdle" value="true" />
    <property name="numTestsPerEvictionRun" value="10" />
    <property name="timeBetweenEvictionRunsMillis" value="60000" />
</bean>

<bean id="connectionPool" class="com.github.eemmiirr.redisdata.jedis.DefaultJedisConnectionPool">
    <constructor-arg name="jedisPool">
        <bean class="redis.clients.jedis.JedisPool">
            <constructor-arg name="host" value="localhost"/>
            <constructor-arg name="port" value="6379"/>
            <constructor-arg name="poolConfig" ref="poolConfig"/>
        </bean>
    </constructor-arg>
</bean>

<bean id="transactionManager" class="com.github.eemmiirr.redisdata.jedis.JedisTransactionManager">
    <constructor-arg name="connectionPool" ref="connectionPool"/>
</bean>
```


#### Define the Data Mappers

##### Available Data Mappers

Data mapper                       | Description 
----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------
ByteArrayDataMapper               | DataMapper which actually just forwards the byte array to/from redis. Useful for some byte manipulations if needed.
FSTDataMapper                     | DataMapper which uses the java fast serialization library.
JacksonJsonDataMapper             | DataMapper which uses jackson json library.
JDKSerialisationDataMapper        | DataMapper which uses teh JDK serialization.
StringValueOfDataMapper           | DataMapper which uses the toString method to serialize and the static valueOf method to deserialize objects. Useful for primitive wrapper classes.


```xml
<bean id="objectMapper" class="org.codehaus.jackson.map.ObjectMapper"/>
<bean id="jacksonJsonDataMapper" class="com.github.eemmiirr.redisdata.datamapper.JacksonJsonDataMapper">
    <constructor-arg name="objectMapper" ref="objectMapper"/>
</bean>

<bean id="stringValueOfDataMapper" class="com.github.eemmiirr.redisdata.datamapper.StringValueOfDataMapper" />
```

#### Define the Session Factory
```xml
<bean id="redisSessionFactory" class="com.github.eemmiirr.redisdata.jedis.JedisSessionFactory">
    <constructor-arg name="defaultKeyDataMapper" ref="jacksonJsonDataMapper"/>
    <constructor-arg name="defaultValueDataMapper" ref="jacksonJsonDataMapper"/>
    <constructor-arg name="transactionManager" ref="transactionManager" />
    <constructor-arg name="keyDataMappers">
        <map>
            <entry key="#{T(java.lang.Class).forName('java.lang.Integer')}" value-ref="stringValueOfDataMapper" />
        </map>
    </constructor-arg>
    <constructor-arg name="valueDataMappers">
        <map>
            <entry key="#{T(java.lang.Class).forName('java.lang.Integer')}" value-ref="stringValueOfDataMapper" />
            <entry key="#{T(java.lang.Class).forName('foo.bar.Entity1')}" value-ref="jacksonJsonDataMapper" />
            <entry key="#{T(java.lang.Class).forName('foo.bar.Entity2')}" value-ref="jacksonJsonDataMapper" />
        </map>
    </constructor-arg>
</bean>
```

#### Define the Command Bindings
##### Available Commands

Command                           | Description
----------------------------------|-------------------------------------------------
KeyCommand                        | Redis key commands. 
StringCommand                     | Redis string commands. Extends key commands.
ListCommand                       | Redis list commands. Extends key commands.
SetCommand                        | Redis set commands. Extends key commands.
SortedSetCommand                  | Redis sorted set commands. Extends key commands.
HashCommand                       | Redis hash commands. Extends key commands.

```xml
<bean id="stringCommandEntity1Binding" class="com.github.eemmiirr.redisdata.binding.CommandBindingFactory" factory-method="createStringCommandBinding">
    <constructor-arg name="sessionFactory" ref="redisSessionFactory"/>
    <constructor-arg name="keyClass" value="#{T(java.lang.Class).forName('java.lang.Integer')}"/>
    <constructor-arg name="valueClass" value="#{T(java.lang.Class).forName('foo.bar.Entity1')}"/>
</bean>

<bean id="stringCommandIntegerBinding" class="com.github.eemmiirr.redisdata.binding.CommandBindingFactory" factory-method="createStringCommandBinding">
    <constructor-arg name="sessionFactory" ref="redisSessionFactory"/>
    <constructor-arg name="keyClass" value="#{T(java.lang.Class).forName('java.lang.Integer')}"/>
    <constructor-arg name="valueClass" value="#{T(java.lang.Class).forName('java.lang.Integer')}"/>
</bean>
```

#### Define the Signalizer

##### Available Signalizers

Signalizer                       | Description 
---------------------------------|------------------------------------
RedisDataSignalizerAspect        | Uses AOP to intersect method calls. 

```xml
<bean id="redisDataSignalizerAspect" class="com.github.eemmiirr.redisdata.signalizer.RedisDataSignalizerAspect" factory-method="getInstance">
    <constructor-arg name="transactionManager" ref="transactionManager"/>
</bean>
```

##### Enabling the signalizer
Either using spring
```xml
<aop:aspectj-autoproxy/>
```
or using AspectJ maven plugin
```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>aspectj-maven-plugin</artifactId>
    <version>${aspectj-maven-plugin.version}</version>
    <configuration>
        <verbose>true</verbose>
        <complianceLevel>${java.version}</complianceLevel>
        <source>${java.version}</source>
        <encoding>UTF-8</encoding>
        <showWeaveInfo>true</showWeaveInfo>
        <weaveDependencies>
            <weaveDependency>
                <groupId>com.github.eemmiirr.redisdata</groupId>
                <artifactId>redisdata-core</artifactId>
            </weaveDependency>
        </weaveDependencies>
        <weaveWithAspectsInMainSourceFolder>false</weaveWithAspectsInMainSourceFolder>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>test-compile</goal>
             </goals>
        </execution>
    </executions>
</plugin>
```

#### Annotate the service
Either annotate the class or method with @RedisData. If both annotations are present the annotation on the method has higher priority.

```java
@RedisData
public class RedisService {

    @Autowired
    private StringCommand<Integer, Entity1> stringCommandEntity1Binding;
    
    @Autowired
    private StringCommand<Integer, Integer> stringCommandIntegerBinding;

    public void set() {
        for (int i = 0; i < 100000; i++) {
            stringCommandEntity1Binding.set(i, new Entity1());
        }
    }

    @RedisData(pipelined = true)
    public void setPipelined() {
        for (int i = 0; i < 100000; i++) {
            stringCommandEntity1Binding.set(i, new Entity1());
        }
    }

    @RedisData(transactional = true)
    public void setTransactional() {
        for (int i = 0; i < 100000; i++) {
            stringCommandEntity1Binding.set(i, new Entity1());
        }
    }

    @RedisData(pipelined = true, transactional = true)
    public void setPipelinedTransaction() {
        for (int i = 0; i < 100000; i++) {
            stringCommandEntity1Binding.set(i, new Entity1());
        }
    }
    
    @RedisData(pipelined = true, transactional = true)
    public Response<Integer> setAndIncrPipelinedTransaction() {
        stringCommandIntegerBinding.set(1, 100);
        stringCommandIntegerBinding.incr(1);
        return stringCommandIntegerBinding.get(1);
    }
}
```

##### Benefits of AspectJ approach 
- It enables the library to work in a framework idependent way
- It's faster then a dynamic proxy aproach because it uses compile time or load time weaving
- It enables calls to methods from within the same class. It doesn't have to go from outside trough the proxy


### Exceptions
All exceptions are unchecked.

Exception                         | Description
----------------------------------|-------------------------------------------------
RedisDataException                | Base Redis Data exception.
ClientException                   | Exception which is thrown when the underlying client has some problems. The original exception is wrapped.
DataMapperException               | Exception which occurres when data can not be mapped.
DataMapperNotSupportedException   | Exception which occurres when the data mapper doesn't support the assigned type.
DataNotReadyException             | Exception which occurres when the data is accessed inside the method which is currently in the middle of a transaction/pipeline.
RedisDataSignalizerException      | Thrown by the signalizer.

### Performance
The performance comes pretty close to native use of a client. 
Tested on:
- OS: Ubuntu 14.04 LTS
- Processor: i5 First Gen 4x4GHz
- Memory: 8gb


|Command  | Type       | Client | Session               | Count   | Time    |
----------|------------|--------|-----------------------|---------|---------|
| set     | Native     | Jedis  | Simple                | 1000000 | 16.475s |
| set     | Native     | Jedis  | Pipelined             | 1000000 |  1.648s |
| set     | Native     | Jedis  | Transaction           | 1000000 |  1.847s |
| set     | Native     | Jedis  | Pipelined Transaction | 1000000 |  2.462s |
| set     | Redis-Data | Jedis  | Simple                | 1000000 | 17.446s |
| set     | Redis-Data | Jedis  | Pipelined             | 1000000 |  1.338s |
| set     | Redis-Data | Jedis  | Transaction           | 1000000 |  1.995s |
| set     | Redis-Data | Jedis  | Pipelined Transaction | 1000000 |  2.174s |


| Command | Type       | Client | Session               | Count   | Time    |
|---------|------------|--------|-----------------------|---------|---------|
| get     | Native     | Jedis  | Simple                | 1000000 | 15.932s |
| get     | Native     | Jedis  | Pipelined             | 1000000 |  1.151s |
| get     | Native     | Jedis  | Transaction           | 1000000 |  1.586s |
| get     | Native     | Jedis  | Pipelined Transaction | 1000000 |  1.725s |
| get     | Redis-Data | Jedis  | Simple                | 1000000 | 16.338s |
| get     | Redis-Data | Jedis  | Pipelined             | 1000000 |  0.909s |
| get     | Redis-Data | Jedis  | Transaction           | 1000000 |  1.564s |
| get     | Redis-Data | Jedis  | Pipelined Transaction | 1000000 |  2.066s |


Run locally performance tests: 
```
mvn clean verify -PrunPerfs
```


### Future work:
- Add semaphoring logic to support nested transactions
- Add support for other clients
- Add support for data compression
- Add support for other method interception frameworks/libraries
- Add support for WATCH
- Improve javadoc
- Improve performance
- Improve test coverage
