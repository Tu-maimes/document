## Kerberos验证手册

[toc]



## 1、Kafka

### 1.1、Kafka 开启Kerberos

- 将创建的kafka服务的keytab文件分发至各个kakfa节点的配置目录中。

```shell
scp kafka.service.keytab root@henghe-100-63:/opt/kafka/config
```

- 修改keytab的所属权限

```shell
chown henghe:hadoop kafka.service.keytab
```

- 创建kafka_server_jaas.conf文件并添加内容以及修改文件权限

```shell
# 节点henghe-100-63
KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
    principal="kafka/henghe-100-63@HENGHE.COM";
};
Client {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
  principal="kafka/henghe-100-63@HENGHE.COM";
};
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
  principal="kafka/henghe-100-63@HENGHE.COM";
};

# 节点henghe-100-64
KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
    principal="kafka/henghe-100-64@HENGHE.COM";
};
Client {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
  principal="kafka/henghe-100-64@HENGHE.COM";
};
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
  principal="kafka/henghe-100-64@HENGHE.COM";
};

# 节点henghe-37
KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
    principal="kafka/henghe-37@HENGHE.COM";
};
Client {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
  principal="kafka/henghe-37@HENGHE.COM";
};
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka/config/kafka.service.keytab"
  principal="kafka/henghe-37@HENGHE.COM";
};
```

- 在部署的kakfa集群中$KAFKA_HOME/bin/kafka-run-class.sh文件。在kafka-run-class.sh文件中找到KAKFA_JVM_PERFORMANCE_OPTS,并增加两个JVM参数：

```shell
-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf
```

- 开启Kerberos服务，需要在配置文件中开启相应的配置项。修改server.properties文件，确保配置项的值如下：

```properties
# 节点henghe-100-63
listeners=SASL_PLAINTEXT://henghe-01:9092
security.inter.broker.protocol = SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol = GSSAPI
sasl.enabled.mechanisms = GSSAPI
sasl.kerberos.service.name = kafka


# 节点henghe-100-64
listeners=SASL_PLAINTEXT://henghe-01:9092
security.inter.broker.protocol = SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol = GSSAPI
sasl.enabled.mechanisms = GSSAPI
sasl.kerberos.service.name = kafka

# 节点henghe-37
listeners=SASL_PLAINTEXT://henghe-37:9092
security.inter.broker.protocol = SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol = GSSAPI
sasl.enabled.mechanisms = GSSAPI
sasl.kerberos.service.name = kafka
```

### 1.2、启动kafka 验证kerbero

- 启动kafka

```shell
$KAFKA_HOME/bin/kafka-server-start.sh -daemon start $KAFKA_HOME/config/server.properties
```

- zookeeper登录成功

![image-20210223113418623](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210223113418.png)

- kafka集群内部通信成功

![image-20210223113547220](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210223113547.png)

- 未认证的Atlas组件

![image-20210223113840025](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210223113840.png)

### 1.3、测试kerbers权限认证

#### 1.3.1 创建主题

```bash
# 创建kafka topic
bin/kafka-topics.sh --create --zookeeper henghe-35:2181 --replication-factor 3 --partitions 5 --topic test-topic
```

#### 1.3.2测试主题测试权限

- 创建console-comsumer.properties

```properties
security.protocol=SASL_PLAINTEXT
sasl.kerberos.service.name=kafka
sasl.mechanism=GSSAPI
group.id=test
```

- 创建console-producer.properties

```properties
security.protocol=SASL_PLAINTEXT
sasl.kerberos.service.name=kafka
sasl.mechanism=GSSAPI
```

```bash
# 启动生产者并发送数据
bin/kafka-console-producer.sh --topic test-topic --broker-list henghe-100-64:9092 --producer.config console-producer.properties
#启动消费者消费发送的数据
bin/kafka-console-consumer.sh --bootstrap-server henghe-100-64:9092 --topic test-topic --from-beginning --consumer.config console-comsumer.properties
```

通过测试相关认证成功并生成与消费数据

![image-20210223131119359](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210223131119.png)

#### 1.3.3 java代码消费kafka数据

- java 代码

```java
public class KerberosTest {
    public static void main(String[] args) throws InterruptedException {
        Properties props = new Properties();
        System.setProperty("java.security.auth.login.config", "D:\\ysstech\\Medusa\\runtime\\src\\main\\resources\\kafka_client_jaas.conf");
        System.setProperty("java.security.krb5.conf", "D:\\ysstech\\Medusa\\runtime\\src\\main\\resources\\krb5.conf");
        props.put("bootstrap.servers", "henghe-37:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test1");
        props.put(ConsumerConfig.CLIENT_ID_CONFIG, "qwe");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put("security.protocol", "SASL_PLAINTEXT");
        props.put("sasl.kerberos.service.name", "kafka");
        props.put("sasl.mechanism", "GSSAPI");
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);
        kafkaConsumer.subscribe(Collections.singletonList("test-topic"));
        while (true) {
            ConsumerRecords<String, String> records = kafkaConsumer.poll(Duration.ofMillis(500));
            for (ConsumerRecord<String, String> record : records) {
                System.out.println(record.value());
            }
        }
    }
}
```

- kafka_client_jaas.conf

```shell
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="D:\\ysstech\\Medusa\\runtime\\src\\main\\resources\\henghe.keytab"
    useTicketCache=false
    principal="henghe@HENGHE.COM"
    serviceName=kafka;
};
```

- keytab授权文件

在kdc中注册的principal

- 运行结果示例

![image-20210223132607094](C:%5CUsers%5Cwangshuai%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20210223132607094.png)



## 2、Medusa

### 2.1 Medusa 开启Kerberos

- 拷贝Mudusa的keytab文件到medusa安装目录config

```bash
scp medusa.service.keytab root@henghe-100-63:/opt/medusa/config
```

- 修改keytab的所属权限

```shell
chown henghe:hadoop medusa.service.keytab
```

- 创建medusa_server_jaas.conf文件并添加内容以及修改文件权限

```shell
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/medusa/config/medusa.service.keytab"
  principal="medusa/henghe-100-63@HENGHE.COM";
};
```

- 在部署的Medusa集群中$MEDUSA_HOME/bin/kafka-run-class.sh文件。在kafka-run-class.sh文件中找到KAKFA_JVM_PERFORMANCE_OPTS,并增加两个JVM参数：

```shell
-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/opt/medusa/config/medusa_server_jaas.conf
```

- 修改connect-distributed.properties添加如下参数

```properties
bootstrap.servers=henghe-100-63:9092
security.protocol=SASL_PLAINTEXT
sasl.kerberos.service.name=kafka
sasl.mechanism=GSSAPI
```

### 2.2 启动Medusa

- 权限认证成功

![image-20210223161813431](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210223161813.png)



### 2.3 medusa任务

> 方案一

- 修改medusa服务的配置覆盖策略并重启

```properties
connector.client.config.override.policy=All
```

- 创建任务

```properties
curl -i -k  -H "Content-type: application/json" -X POST -d '{
    "name":"test",
    "config":{
        "topics":"test-topic",
        "connector.class":"FileStreamSinkConnector",
        "tasks.max":"1",
        "file":"/opt/medusa/user.out",
        "value.converter":"org.apache.kafka.connect.storage.StringConverter",
        "consumer.override.security.protocol":"SASL_PLAINTEXT",
        "consumer.override.sasl.kerberos.service.name":"kafka",
        "consumer.override.sasl.mechanism":"GSSAPI"
    }
}' http://192.168.100.63:8083/connectors



curl -X DELETE  http://192.168.100.63:8083/connectors/test

```

> 方案二

- 修改medusa服务的配置覆盖策略并重启

```properties
consumer.security.protocol=SASL_PLAINTEXT
consumer.sasl.kerberos.service.name=kafka
consumer.sasl.mechanism=GSSAPI
```



- 创建任务

```properties
  curl -i -k  -H "Content-type: application/json" -X POST -d '{
      "name":"test",
      "config":{
          "topics":"test-topic",
          "connector.class":"FileStreamSinkConnector",
          "tasks.max":"1",
          "file":"/opt/medusa/user.out",
          "value.converter":"org.apache.kafka.connect.storage.StringConverter"
      }
  }' http://192.168.100.63:8083/connectors
  
  
  
  curl -X DELETE  http://192.168.100.63:8083/connectors/test
```

  

- 输出结果

![image-20210223174429741](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210223174429.png)

### 2.4 命令扩展

```shell
[henghe@henghe-100-64 kafka]$ bin/kafka-consumer-groups.sh --bootstrap-server henghe-100-64:9092 --list --command-config console-producer.properties
connect-test
test
[2021-02-23 17:41:10,696] WARN [Principal=kafka/henghe-100-64@HENGHE.COM]: TGT renewal thread has been interrupted and will exit. (org.apache.kafka.common.security.kerberos.KerberosLogin)
[henghe@henghe-100-64 kafka]$ bin/kafka-consumer-groups.sh --bootstrap-server henghe-100-64:9092 --delete --group test --command-config console-producer.properties
Deletion of requested consumer groups ('test') was successful.
[2021-02-23 17:41:40,311] WARN [Principal=kafka/henghe-100-64@HENGHE.COM]: TGT renewal thread has been interrupted and will exit. (org.apache.kafka.common.security.kerberos.KerberosLogin)
[henghe@henghe-100-64 kafka]$

```

## 3、Schema

### 3.1 Schema开启kerberos

- 拷贝schema.service.keytab到schema安装目录下的etc中
- 修改schema.service.keytab文件的权限
- 创建schema_server_jaas.conf并添加如下内容

```properties
Client {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/schema/etc/schema.service.keytab"
  principal="schema/henghe-100-64@HENGHE.COM";
};
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/schema/etc/schema.service.keytab"
  principal="schema/henghe-100-64@HENGHE.COM";
};
```

- 在部署的Schema集群中$Schema_HOME/bin/schema-registry-run-class文件。在schema-registry-run-class文件中找到SCHEMA_REGISTRY_LOG4J_OPTS,并增加两个JVM参数：

```shell
-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/opt/schema/etc/schema_server_jaas.conf
```

![image-20210223185231773](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210223185231.png)

- 修改schema-registry.properties配置文件

```properties
kafkastore.security.protocol=SASL_PLAINTEXT
kafkastore.sasl.kerberos.service.name=kafka
kafkastore.sasl.mechanism=GSSAPI
```

### 3.2 启动Schema服务

- 启动成功

![image-20210223190844559](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210223190844.png)

- 添加schema

```properties
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data '{"schema": "{\"type\": \"string\"}"}' http://192.168.100.64:18081/subjects/Kafka-key/versions
```

- 查看schema

```http
curl -X GET http://192.168.100.64:18081/subjects
```

![image-20210223191135921](C:%5CUsers%5Cwangshuai%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20210223191135921.png)

## 4、DataX

### 4.1 kafka

- 拷贝授权文件到dataX安装机器节点
- 创建client_jaas.conf文件

```properties
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/henghe.user.keytab"
  principal="henghe@HENGHE.COM";
};
```

- 创建datax任务

```json
{
    "job":{
        "setting":{
            "speed":{
                "channel":1
            },
            "errorLimit":{
                "record":0,
                "percentage":0.02
            }
        },
        "content":[
            {
                "reader":{
                    "name":"mysqlreader",
                    "parameter":{
                        "username":"root",
                        "password":"root",
                        "column":[
                            "id",
                            "a",
                            "b",
                            "c",
                            "d",
                            "e",
                            "f",
                            "g"
                        ],
                        "where":"id<10000",
                        "connection":[
                            {
                                "table":[
                                    "test"
                                ],
                                "jdbcUrl":[
                                    "jdbc:mysql://192.168.101.42:3306/test"
                                ]
                            }
                        ]
                    }
                },
                "writer":{
                    "name":"kafkaWriter",
                    "parameter":{
                        "kafkaConfig":{
                            "bootstrap.servers":"henghe-100-63:9092,henghe-100-64:9092",
                            "security.protocol":"SASL_PLAINTEXT",
                            "sasl.kerberos.service.name":"kafka",
                            "sasl.mechanism":"GSSAPI",
                            "java.security.krb5.conf":"/etc/krb5.conf",
                            "java.security.auth.login.config":"/opt/client_jaas.conf"
                        },
                        "topics":"data",
                        "column":{
                            "type":"record",
                            "name":"data",
                            "fields":[
                                {
                                    "name":"id",
                                    "type":"int",
                                    "index":0
                                },
                                {
                                    "name":"a",
                                    "type":"double",
                                    "index":1
                                },
                                {
                                    "name":"b",
                                    "type":"float",
                                    "index":2
                                },
                                {
                                    "name":"c",
                                    "type":"decimal",
                                    "scale":2,
                                    "index":3
                                },
                                {
                                    "name":"d",
                                    "type":"date",
                                    "index":4
                                },
                                {
                                    "name":"e",
                                    "type":"string",
                                    "index":5
                                },
                                {
                                    "name":"f",
                                    "type":"time",
                                    "index":6
                                },
                                {
                                    "name":"g",
                                    "type":"datetime",
                                    "index":7
                                }
                            ]
                        },
                        "medusa":{
                            "hostName":"http://henghe-100-63:8083",
                            "name":"data"
                        },
                        "schemaRegistry":{
                            "schema.registry.url":"http://henghe-100-64:18081",
                            "schemas.enable":true,
                            "value.converter":"io.confluent.connect.avro.AvroConverter"
                        }
                    }
                }
            }
        ]
    }
}
```

- 权限认证成功

![image-20210224104752383](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224104752.png)

- 任务运行成功

![image-20210224105305764](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224105305.png)

### 4.2 HDFS

- 拷贝授权文件到dataX安装机器节点
- 创建任务

```json
{
    "job":{
        "setting":{
            "speed":{
                "channel":5
            },
            "errorLimit":{
                "record":0,
                "percentage":0.02
            }
        },
        "content":[
            {
                "reader":{
                    "name":"mysqlreader",
                    "parameter":{
                        "username":"root",
                        "password":"root",
                        "column":[
                            "id",
                            "a",
                            "b",
                            "c",
                            "d",
                            "e",
                            "f",
                            "g"
                        ],
                        "where":"id<=1000000","splitPk":"id",
                        "connection":[
                            {
                                "table":[
                                    "test"
                                ],
                                "jdbcUrl":[
                                    "jdbc:mysql://192.168.101.42:3306/test"
                                ]
                            }
                        ]
                    }
                },
                "writer":{
                    "name":"hdfswriter",
                    "parameter":{
                        "column":[
                            {
                                "name":"id",
                                "type":"int",
                                "index":0
                            },
                            {
                                "name":"a",
                                "type":"double",
                                "index":1
                            },
                            {
                                "name":"b",
                                "type":"decimal",
                                "index":2
                            },
                            {
                                "name":"c",
                                "type":"string",
                                "index":3
                            },
                            {
                                "name":"d",
                                "type":"string",
                                "index":4
                            },
                            {
                                "name":"e",
                                "type":"string",
                                "index":5
                            },
                            {
                                "name":"f",
                                "type":"string",
                                "index":6
                            },
                            {
                                "name":"g",
                                "type":"string",
                                "index":7
                            }
                        ],
                        "defaultFS":"hdfs://henghe-039:8020",
                        "fieldDelimiter":"\t",
                        "fileName":"yss",
                        "fileType":"ORC",
                        "path":"/tmp/ws",
                        "writeMode":"overwrite",
                        "haveKerberos":"true",
                        "kerberosKeytabFilePath":"/opt/henghe.user.keytab",
                        "kerberosPrincipal":"henghe@HENGHE.COM"
                    }
                }
            }
        ]
    }
}

```

- 认证成功

![image-20210224113315969](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224113316.png)

- 成功写入

![image-20210224113431270](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224113431.png)



## 5、Flume

查看官网配置即可，具体没有验证。

- 创建任务

```properties
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = TAILDIR
a1.sources.r1.channels = c1
a1.sources.r1.positionFile = /opt/flume/test/taildir_position.json
a1.sources.r1.filegroups = f1
a1.sources.r1.filegroups.f1 = /opt/flume/example.log
a1.sources.r1.fileHeader = true
a1.sources.ri.maxBatchCount = 1000		


a1.channels = c1
a1.channels.c1.type = memory
a1.channels.c1.capacity = 10000
a1.channels.c1.transactionCapacity = 10000
a1.channels.c1.byteCapacityBufferPercentage = 20
a1.channels.c1.byteCapacity = 800000


a1.channels = c1
a1.sinks = k1
a1.sinks.k1.type = hdfs
a1.sinks.k1.channel = c1


a1.sinks.k1.hdfs.path = hdfs://henghe-039:8020/tmp/ws
a1.sinks.k1.hdfs.fileType = DataStream
a1.sinks.k1.hdfs.writeFormat=TEXT
a1.sinks.k1.hdfs.filePrefix = flumeHdfs
a1.sinks.k1.hdfs.batchSize = 1000
a1.sinks.k1.hdfs.rollSize = 10240
a1.sinks.k1.hdfs.rollCount = 0
a1.sinks.k1.hdfs.rollInterval = 1
a1.sinks.k1.hdfs.useLocalTimeStamp = true
a1.sinks.k1.hdfs.kerberosPrincipal = henghe@HENGHE.COM
a1.sinks.k1.hdfs.kerberosKeytab = /opt/henghe.user.keytab

```

- 权限认证通过

![image-20210224181838222](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224181838.png)

- 查看同步的数据

![image-20210224182157901](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224182157.png)

## 6、sqoop

- 创建sqoop任务

```shell
bin/sqoop import \
--connect jdbc:mysql://192.168.101.42:3306/test \
--username root \
--password root \
--query 'SELECT
	id,
	a,
	b,
	c,
	d,
	e,
	f,
	g
FROM
	test
where $CONDITIONS LIMIT 100;' \
--target-dir /tmp/ws \
--delete-target-dir \
--num-mappers 1 \
--compress \
--compression-codec org.apache.hadoop.io.compress.SnappyCodec \
--fields-terminated-by '\t'
```



- sqoop安装的hadoop节点没有开启kerberos时

![image-20210224151511695](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224151511.png)

- 使用kinit切换用户

```shell
 kinit -kt /opt/henghe.user.keytab henghe@HENGHE.COM
```

![image-20210224152039795](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224152040.png)

![image-20210224152814227](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224152814.png)

- 缓存问题

```shell
# Configuration snippets may be placed in this directory as well
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
 default_realm = HENGHE.COM
 # 在个别情况下需要关闭缓存
 default_ccache_name = KEYRING:persistent:%{uid}

[realms]
HENGHE.COM = {
kdc = henghe-76
admin_server = henghe-76
}

[domain_realm]
.henghe.com = HENGHE.COM
henghe.com = HENGHE.COM
```

## 7、kafka-Manager

- 拷贝keytab文件到kafka-Manager安装集群
- 修改keytab文件所属权限
- 创建jaas.conf文件

```properties
Client {
com.sun.security.auth.module.Krb5LoginModule required
useKeyTab=true
keyTab="/opt/kafka-manager/conf/henghe.user.keytab"
storeKey=true
useTicketCache=false
principal="henghe@HENGHE.COM";
};
KafkaClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/kafka-manager/conf/henghe.user.keytab"
  principal="henghe@HENGHE.COM";
};
```

- 修改consumer.properties

```properties
key.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer
value.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer
security.protocol=SASL_PLAINTEXT
sasl.kerberos.service.name=kafka
sasl.mechanism=GSSAPI
```

- 修改application.conf文件中的配置项指向consumer.properties

```properties
kafka-manager.consumer.properties.file=/opt/kafka-manager/conf/consumer.properties
```

- 启动

```shell
nohup bin/kafka-manager -Dconfig.file=conf/application.conf -Djava.security.auth.login.config=/opt/kafka-manager/conf/jaas.conf &
```

![image-20210224180110024](https://gitee.com/Tu_maimes/csdn_image/raw/master/img/20210224180110.png)

