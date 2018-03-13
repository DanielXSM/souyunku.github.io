---
layout: post
title: Twitter�ķֲ�ʽѩ���㷨 SnowFlake ÿ����������26������������ID (Java��) 
categories: java
description: Twitter�ķֲ�ʽѩ���㷨 SnowFlake ÿ����������26������������ID (Java��) 
keywords: java 
---

�ֲ�ʽϵͳ�У���һЩ��Ҫʹ��ȫ��ΨһID�ĳ���������ʱ��Ϊ�˷�ֹID��ͻ����ʹ��36λ��UUID������UUID��һЩȱ�㣬��������ԱȽϳ�������UUIDһ��������ġ�

��Щʱ������ϣ����ʹ��һ�ּ�һЩ��ID������ϣ��ID�ܹ�����ʱ���������ɡ�

��twitter��SnowFlake����������������Twitter�Ѵ洢ϵͳ��MySQLǨ�Ƶ�Cassandra����ΪCassandraû��˳��ID���ɻ��ƣ����Կ���������һ��ȫ��ΨһID���ɷ���

# ԭ��

Twitter��ѩ���㷨SnowFlake��ʹ��Java����ʵ�֡�

SnowFlake�㷨������ID��һ��64λ�����ͣ��ṹ���£�ÿһ�����á�-�����ŷָ�����

```
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000
```

**1λ��ʶ����**����java������long�����λ�Ƿ���λ��������0��������1��һ�����ɵ�IDΪ����������Ϊ0��

**41λʱ�������**������Ǻ��뼶��ʱ�䣬һ��ʵ���ϲ���洢��ǰ��ʱ���������ʱ����Ĳ�ֵ����ǰʱ��-�̶��Ŀ�ʼʱ�䣩����������ʹ������ID�Ӹ�Сֵ��ʼ��41λ��ʱ�������ʹ��69�꣬(1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69�ꣻ

**10λ�ڵ㲿��**��Twitterʵ����ʹ��ǰ5λ��Ϊ�������ı�ʶ����5λ��Ϊ������ʶ�����Բ���1024���ڵ㣻

**12λ���кŲ���**��֧��ͬһ������ͬһ���ڵ��������4096��ID��

SnowFlake�㷨���ɵ�ID�������ǰ���ʱ������ģ����ڷֲ�ʽϵͳ��ʱ����Ҫע���������ı�ʶ�ͻ�����ʶ����Ψһ���������ܱ�֤ÿ���ڵ����ɵ�ID����Ψһ�ġ��������ǲ�һ������Ҫ����������ʹ��5λ��Ϊ�������ı�ʶ��5λ��Ϊ������ʶ�����Ը�������ҵ�����Ҫ��������ڵ㲿�֣��磺������Ҫ�������ģ���ȫ����ʹ��ȫ��10λ��Ϊ������ʶ�����������Ĳ��࣬Ҳ����ֻʹ��3λ��Ϊ�������ģ�7λ��Ϊ������ʶ��

snowflake���ɵ�ID�����ϰ���ʱ���������򣬲��������ֲ�ʽϵͳ�ڲ������ID��ײ����datacenter��workerId�����֣�������Ч�ʽϸߡ���˵��snowflakeÿ���ܹ�����26���ID��


# Դ��

**����ʵ�⣺100���ID ��ʱ5��**

```java
/**
 * ����: Twitter�ķֲ�ʽ����IDѩ���㷨snowflake (Java��)
 * https://github.com/souyunku/SnowFlake
 *
 * @author yanpenglei
 * @create 2018-03-13 12:37
 **/
public class SnowFlake {

    /**
     * ��ʼ��ʱ���
     */
    private final static long START_STMP = 1480166465631L;

    /**
     * ÿһ����ռ�õ�λ��
     */
    private final static long SEQUENCE_BIT = 12; //���к�ռ�õ�λ��
    private final static long MACHINE_BIT = 5;   //������ʶռ�õ�λ��
    private final static long DATACENTER_BIT = 5;//��������ռ�õ�λ��

    /**
     * ÿһ���ֵ����ֵ
     */
    private final static long MAX_DATACENTER_NUM = -1L ^ (-1L << DATACENTER_BIT);
    private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);

    /**
     * ÿһ���������λ��
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATACENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTMP_LEFT = DATACENTER_LEFT + DATACENTER_BIT;

    private long datacenterId;  //��������
    private long machineId;     //������ʶ
    private long sequence = 0L; //���к�
    private long lastStmp = -1L;//��һ��ʱ���

    public SnowFlake(long datacenterId, long machineId) {
        if (datacenterId > MAX_DATACENTER_NUM || datacenterId < 0) {
            throw new IllegalArgumentException("datacenterId can't be greater than MAX_DATACENTER_NUM or less than 0");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("machineId can't be greater than MAX_MACHINE_NUM or less than 0");
        }
        this.datacenterId = datacenterId;
        this.machineId = machineId;
    }

    /**
     * ������һ��ID
     *
     * @return
     */
    public synchronized long nextId() {
        long currStmp = getNewstmp();
        if (currStmp < lastStmp) {
            throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
        }

        if (currStmp == lastStmp) {
            //��ͬ�����ڣ����к�����
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //ͬһ������������Ѿ��ﵽ���
            if (sequence == 0L) {
                currStmp = getNextMill();
            }
        } else {
            //��ͬ�����ڣ����к���Ϊ0
            sequence = 0L;
        }

        lastStmp = currStmp;

        return (currStmp - START_STMP) << TIMESTMP_LEFT //ʱ�������
                | datacenterId << DATACENTER_LEFT       //�������Ĳ���
                | machineId << MACHINE_LEFT             //������ʶ����
                | sequence;                             //���кŲ���
    }

    private long getNextMill() {
        long mill = getNewstmp();
        while (mill <= lastStmp) {
            mill = getNewstmp();
        }
        return mill;
    }

    private long getNewstmp() {
        return System.currentTimeMillis();
    }

    public static void main(String[] args) {
        SnowFlake snowFlake = new SnowFlake(2, 3);

        long start = System.currentTimeMillis();
        for (int i = 0; i < 1000000; i++) {
            System.out.println(snowFlake.nextId());
        }

        System.out.println(System.currentTimeMillis() - start);


    }
}
```

ѭ�����ɵ�ID�����н�����£�

```
170916032679263329
170916032679263330
170916032679263331
170916032679263332
170916032679263333
170916032679263334
170916032679263335
170916032679263336
170916032679263337
170916032679263338
170916032679263339
170916032679263340
170916032679263341
170916032679263342
```

# ��Դ��ַ

**Github**:[https://github.com/souyunku/SnowFlake](https://github.com/souyunku/SnowFlake)

# �Ƽ��Ķ�

## Spring Cloud ϵ�н̳�

- [Spring Cloud��һ�������ע���뷢�� Eureka ](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964403&idx=1&sn=73ad9f65e2530bf87dadd35a96658fd7)
- [Spring Cloud������Consul ��������ʵ�� ](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964404&idx=1&sn=72676a761715bdd1e4711360dfa2c469)
- [Spring Cloud�����������ṩ�� Eureka + ���������ߣ�rest + Ribbon��](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964405&idx=1&sn=9a85514edf8e8fbe301aea5c0a8fb55e)
- [Spring Cloud���ģ������ṩ�� Eureka + ���������� Feign ](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964406&idx=1&sn=6884383251b4eb9ae8a9c9f3ea4f3c09)
- [Spring Cloud���壩��·�����(Hystrix Dashboard)](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964412&idx=1&sn=5e5e5208aedf7324f1f0ecd7e3780344)
- [Spring Cloud�������������� zuul ��������](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964422&idx=1&sn=4feb1646baa0a2172cdf953f55824002)
- [Spring Cloud���ߣ��������� Zuul Filter ʹ��](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964425&idx=1&sn=f33137f5c312b2d938d17bb1125d411d)
- [Spring Cloud���ˣ��߿��õķֲ�ʽ�������� Spring Cloud Config](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964427&idx=1&sn=9aed5b79885c8e70831b5219ba1571be)
- [Spring Cloud���ţ��߿��õķֲ�ʽ�������� Spring Cloud Config ���� Eureka ����](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964439&idx=1&sn=556c328bd756555ff2c3422277fb7ffc)
- [Spring Cloud��ʮ���߿��õķֲ�ʽ�������� Spring Cloud Config ��ʹ�� Refresh](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964444&idx=1&sn=b8826d3d314d5f64bd5c54bcc8c15dbe)
- [Spring Cloud��ʮһ���߿��õķֲ�ʽ�������� Spring Cloud Bus ��Ϣ���߼��ɣ�RabbitMQ��](http://mp.weixin.qq.com/s/R9bghtpnLPtGBHjBJRR0Og)

## Spring Boot ϵ�н̳�

Դ�� + �̳�

**Github**:[https://github.com/souyunku/spring-boot-examples](https://github.com/souyunku/spring-boot-examples)

![Spring Cloud ϵ�н̳�](http://www.ymq.io/images/2018/spring/SpringBoot.png "Spring Cloud ϵ�н̳�")

## Docker ����

- [Docker Compose 1.18.0 ֮����������](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964477&idx=1&sn=3cfa2332fc3f5fa0cb7613987b48f449)
- [Docker CE ��װ ���� Dockerfile ���� Nginx](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964410&idx=1&sn=a1d189b731662f062ab5c8ae6c1d4662)
- [Docker Container ��������](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964415&idx=1&sn=a1485845724a3595cbeb22abfd19c6ad)
- [Docker Hub �ֿ�ʹ�ã���� Docker Registry](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964418&idx=1&sn=9f7b142968b75eaa7d0934d1045e0ce9)
- [Docker Registry Server �,������� HTTPS ֤�飬��ӵ��Ȩ����֤��TLS ��˽�вֿ�](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964421&idx=1&sn=8ac4f3fb5dc1828b67b5798ffc0a9753)
- [Docker Registry ��ҵ��˽�о���ֿ�Harbor����WEB UI, ����������ϸ�Ĳ���](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964426&idx=1&sn=0c6d7d1ae718c5fea9901e5321aba403)
- [Docker ���� SpringBoot ��Ŀ���� Redis ���������ʼ��� Demo](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964438&idx=1&sn=f709b17011fa2713221f66aa28e5702e)
- [Docker Maven Plugin ���� Docker ���� push �� DockerHub�� ](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964451&idx=1&sn=4865f254dcfa69fe0e98e7dde4a6cd5b)

## �����
- [� Apache RocketMQ ��������](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964513&idx=1&sn=3c926d8f96048ba17dee17541c69893f&chksm=88ede9c9bf9a60dfb530bc3f7d892c826e99fba5a6c533744dcd25ebc6ab8839cbf79e607c5d#rd)  
- [�ְ��ֽ��� MongoDB �İ�װ����ϸʹ�ã�һ��](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964503&idx=1&sn=1335ece54312f9ed91232ce34d222dd5&chksm=88ede9ffbf9a60e9e03efc28fb4fb469b8212d2ea52208c3db4a1b37b4437b8accbff4a2ed86#rd)  
- [�ְ��ֽ��� MongoDB �İ�װ����ϸʹ�ã�����](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964506&idx=1&sn=9f473a0797b6f091629a6e42dc870133&chksm=88ede9f2bf9a60e4d20402824eaf9803261276073a988659d3b45c82bf4f9bc7f589a08e2086#rd)  
- [� MongoDB��Ƭ��sharding�� / ���� / ��Ⱥ����](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964509&idx=1&sn=9476cda64a51956ae7c7a10352ca31b5&chksm=88ede9f5bf9a60e32f03832ced791e6fd95defa18a3ccc854f2a0ad51527a3ffc7d892132fe8#rd)  
- [� SolrCloud ��Ⱥ����](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964464&idx=2&sn=f944f824c58acc709aea4e86498af426)
- [� Solr ��������](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964464&idx=3&sn=32f023390ebdfe1d30bdf8c109316f83&scene=19#wechat_redirect)
- [� RabbitMQ ��Ⱥ����](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964457&idx=3&sn=abb501f40f3314ecf28b5a96ab028f5c)
- [� RabbitMQ ��������](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964457&idx=2&sn=7d19d7067e33e262463cd70186b23946)
- [Mycat ��д���� ���ݿ�ֿ�ֱ� �м�� ��װ���𣬼���ʹ��](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964443&idx=1&sn=ef6243304e80b32b300b36e88cb7dae9)
- [���߲��� CDH 5.12.1 ��ʹ�� CDH ���� Hadoop ������ƽ̨��Ⱥ����](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964430&idx=1&sn=f76c3b3e2fa41fe149fa21dd91061c10)

# Contact

 - ���ߣ�����  
 - ������[http://www.ymq.io](http://www.ymq.io)  
 - ��Ȩ���������У�ת����ע������
 - Wechat����ע���ںţ����ƿ⣬רע�ڿ����������о���֪ʶ����
 
![��ע���ں�-���ƿ�](http://www.ymq.io/images/souyunku.png "���ƿ�")