# Spring Cloud Config 사용하기


## Spring Cloud Config 설정 내용 gitlab 올리기
Spring Boot Application에서 사용하는 application.yml에서 profile 구분에 따른 설정 내용울 파일로 구분해서 gitlab에서 형상을 관리할 수 있도록 한다.    
아래의 yml이 각 프로파일에 따라서 구성될 수 있다. 해당 프로파일은 local, dev, staging, prod로 구성되고 각 프로파일 마다 접속정보나 설정이 달라 질 수 있다.   
Spring Cloud Config Server에서 형상 서버에서 해당 설정 파일을 읽어 들일 수 있다. 아래 설정에 대한 부분들이 각 파일들로 구성해서 gitlab서버에 올린다.   
```yml
# local environment (개발 DB사용)
spring:
  cloud:
    config:
      enabled: true
  datasource:
    hikari:
      master:
        driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
        jdbc-url: jdbc:log4jdbc:mysql://localhost:3306/test?useSSL=false&characterEncoding=UTF-8&serverTimezone=Asia/Seoul&autoReconnect=true&autoReconnectForPolls=true&allowMultiQueries=true&allowPublicKeyRetrieval=true
        username: xxxxxxx
        password: xxxxxx
        read-only: false
        connection-timeout: 30000
        maximum-pool-size: 5
        test-on-borrow: true
        connection-test-query: SELECT 1
      slave:
        driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
        jdbc-url: jdbc:log4jdbc:mysql://localhost:3306/test?useSSL=false&characterEncoding=UTF-8&serverTimezone=Asia/Seoul&autoReconnect=true&autoReconnectForPolls=true&allowMultiQueries=true&allowPublicKeyRetrieval=true
        username: xxxxxxx
        password: xxxxxx
        read-only: true
        connection-timeout: 30000
        maximum-pool-size: 5
        test-on-borrow: true
        connection-test-query: SELECT 1
      auth:
        driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
        jdbc-url: jdbc:log4jdbc:mysql://localhost:3306/test2?useSSL=false&characterEncoding=UTF-8&serverTimezone=Asia/Seoul&autoReconnect=true&autoReconnectForPolls=true&allowMultiQueries=true&allowPublicKeyRetrieval=true
        username: xxxxxxx
        password: xxxxxx
        read-only: false
        connection-timeout: 30000
        maximum-pool-size: 5
        test-on-borrow: true
        connection-test-query: SELECT 1
  redis:
    master:
      host: xxx.xxx.xxx.xxxx
      port: 6380
      password: xxxxxxxxxxx
      timeout: 5000
      database: 0
      pool:
        max_active: 8
        max_idle: 8
        min_idle: 0
        max_wait: -1
    slaves:
      - host: xxx.xxx.xxx.xxxx
        port: 6380
        password: xxxxxxxxxxx
        timeout: 5000
        database: 0
        pool:
          max_active: 8
          max_idle: 8
          min_idle: 0
          max_wait: -1
  data:
    mongodb:
      authentication-database: admin
      host: xxx.xxx.xxx.xxx
      port: 27017
      slave-port: 27018
      username: xxxxx
      password: xxxxxxx
      database: dbtest
      repositories:
        type: auto
      replica-set: rs0
      field-naming-strategy: org.springframework.data.mapping.model.SnakeCaseFieldNamingStrategy #동작안함, RedisConfig에서 설정함.

      cluster-server-selection-timeout: 30000
      cluster-local-threshold: 15000

      socket-connect-timeout: 10000 #
      socket-read-timeout: 10000 #

      connection-pool-max-connection-idle-time: 1 #
      connection-pool-max-connection-life-time: 1 #
      connection-pool-min-size: 5 #
      connection-pool-max-size: 10 #
      connection-pool-maintenance-frequency: 10000 #
      connection-pool-maintenance-initial-delay: 11000 #
      connection-pool-max-wait-time: 120000 #최대 대기 시간

  sso:
    auth-url: https://xxx.xxx.xxx/api/v1/auth/authorize
    client-id: xxxxx

# log4jdbc, Mybatis Console Log
logging:
  level:
    com:
      zaxxer:
        hikari: INFO
    javax:
      sql:
        DataSource: OFF
    jdbc:
      audit: OFF
      resultset: OFF
      resultsettable: INFO  #SQL 결과 데이터 Table을 로그로 남긴다.
      sqlonly: OFF     #SQL만 로그로 남긴다.
      sqltiming: INFO    #SQL과 소요시간을 표기한다.
      connection : OFF  # 커넥션 확인가능
    org:
      hibernate:
        SQL: DEBUG
        type:
          descriptor:
            sql:
              BasicBinder: TRACE
      springframework:
        data:
          mongodb:
            core:
              MongoTemplate: INFO
    io:
      lettuce:
        core:
          protocol: ERROR

oci:
  api-tenancy: ocid1.tenancy~~~~~
  api-user: ocid1.user~~~~
  region: ap-seoul-1
  finger-print: 36:a3:8d:6e:54:1a:dc:~~~~~
  pass-phrase: N
  key-file: cert/oci_api_key_dev.pem
  endpoint: https://~~~~~~~
  minute-queue-name: minute-queue-name
  log-queue-name : log-queue-name

#custom
custom:
  cloud:
    cdn:
      accountId: ~~~~~~
      apiKey: ~~~~~~
      email: test@gmail.com
      imgUploadUrl: https://api.cloudflare.com/client/v4/accounts/{account_id}/images/v1
      videoUploadUrl: https://api.cloudflare.com/client/v4/accounts/{account_id}/stream
      VideoDirectUploadUrl: https://api.cloudflare.com/client/v4/accounts/{account_id}/stream/direct_upload
      ImageDirectUploadUrl: https://api.cloudflare.com/client/v4/accounts/{account_id}/images/v2/direct_upload
  push:
    server:
      baseUrl: http://push-url
  auth:
    server:
      baseUrl: https://auth-url
```

## Spring Cloud Config Server 구축
Spring Cloud Config Server 구축은 build.gradle에 Spring Cloud Config 관련 의존성을 추가하고 Application Main 클래스에 annotation으로 Spring Cloud Config를 사용하겠다고 선언을 하면 된다. 또한 Config 설정 내용을 어디서 가져올 것인지에 대한 부분을 application.yml에 설정을 한다.    
### build.gradle
```gradle
ext {
    set('springCloudVersion', "2021.0.4")
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-config-server'
    implementation 'org.springframework.cloud:spring-cloud-starter-config'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'com.github.ulisesbocchio:jasypt-spring-boot-starter:3.0.5'
    implementation 'net.logstash.logback:logstash-logback-encoder:7.1.1'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
```

### Application 에 annotation 추가
@EnableConfigServer annotation을 설정하면 해당 어플리케이션은 Spring Cloud Config Server로 설정된다.
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class FantooConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(FantooConfigServerApplication.class, args);
    }

}
```

### application.yml에 Config server 설정
application.yml에서는 설정 내용을 어디에서 가져올 것인지에 대한 부분을 설정한다. 보통은 git 서버를 통해서 형상을 관리하고 해당 서버에서 가져올 수 있도록 설정한다.    
gitlab server 접속시 access token을 발급해서 해당 토큰을 등록해서 가져올 수 있다. 또한 ssh로 접속해서도 가져올 수 있다. 아래 샘플은 access token을 발급 받아서 접속해서는 가져오는 샘플이다.    
```yaml
server:
  port: 8888
spring:
  application:
    name: config
  security:
    user:
      name: config-server-user
      password: xkqUf5W00ab6adfasfds470235994dddadfafasfdcda730ccb0d1a86df9dfdde33glpatxxxxxx
  cloud:
    config:
      server:
        git:
          default-label: master
          uri: http://xxx.xxx.xxx.xxx:9000/server/config.git
          username: config-server
          password: xxxxxxxxxxxxxxxxx
---
spring:
  config:
    activate:
      on-profile: dev
logging:
  config: classpath:logback-spring-dev.xml
  level:
    root: DEBUG

---
spring:
  config:
    activate:
      on-profile: prod
logging:
  config: classpath:logback-spring-prod.xml
  level:
    root: DEBUG
```
security에서의 user name과 password의 경우 클라이언트에서 config server에 접속할 때 해당 아이디와 패스워드를 가지고 접속할 수 있도록 구성한 것이다.   
cloud.config.server.git에서 default-label: master는 브렌치명이고 uri는 git의 주소이다. username과 password의 경우 access token을 생성할 때 name과 토큰 값이다.   

각 프로파일별 접속 정보를 달리 해서 해당 프로파일만 가져오게 할 수 있다. 위의 샘플은 서버는 형상에 접속해서 환경 정보를 가져오고 클라이언트 접속시 필요한 프로파일 구분에 따라서 해당 정보를 내려주는 방식이다.    

## Spring Cloud Config Client 설정

Spring Cloud Config Client에서는 sping cloud config에 관계된 의존성을 추가한다.  또한 어플리케이션의 여러 설정 부분에서 Spring Cloud Config 정보를 읽어오기전에 무엇인가 구성해서 에러가 발생되는 경우가 있다. 그럴 경우 Spring Cloud Config를 먼저 읽어 들이고 Bean 구성이 될 수 있도록 bootstrap으로 구성해야 되므로 해당 의존성을 추가하고 bootstrap에서 Spring Cloud Config를 설정할 수 있도록 한다. 

### build.gradle
```yml
ext {
	set('springCloudVersion', "2021.0.8")
}
~~~

	implementation 'org.springframework.cloud:spring-cloud-starter-config'
	implementation 'org.springframework.cloud:spring-cloud-starter-bootstrap'

```

### application.yml
프로파일에 따라 읽어들여야 할 설정 파일들을 모두 제거한다. 
```yml
spring:
  main:
    allow-bean-definition-overriding: true
  servlet:
    multipart:
      max-file-size: 1GB
      max-request-size: 1GB

  mvc: #Swagger
    pathmatch:
      matching-strategy: ANT_PATH_MATCHER # STS > 소문자 시 (ant_path_matcher), problems > error 표시됨, 단순히 대문자로 변경했음 > 이슈 해결됨
    hiddenmethod:
      filter:
        enabled: true

management:
  endpoints:
    web:
      exposure:
        include: metrics, prometheus, health, refresh
    refresh:
      enabled: true
server:
  port: 9000
  servlet:
    session:
      timeout: 50m
  http2:
    enabled: true
  shutdown: graceful
```

### bootstrap.yml
아래 bootstrap.yml에서 설정한 부분에서 cloud.config.name은 git 형상에서 abc-admin-api-local.yml 로 등록된 파일명에서 abc-admin-api를 가리킨다.   
spring.config.activate.on-profile이 local이면 abc-admin-api-local.yml 파일을 읽고 dev이면 abc-admin-api-dev.yml 내용으로 구성한다. 
```yml
---
spring:
  config:
    activate:
      on-profile: local
  cloud:
    config:
      uri: http://xxx.xxx.xxx.xxx
      name: abc-admin-api
      username: Config 서버의 security에서 선언한 user.name
      password: Config 서버의 security에서 선언한 user.password
spring:
  config:
    activate:
      on-profile: dev
  cloud:
    config:
      uri: http://xxx.xxx.xxx.xxx
      name: abc-admin-api
      username: Config 서버의 security에서 선언한 user.name
      password: Config 서버의 security에서 선언한 user.password
---
spring:
  config:
    activate:
      on-profile: staging
  cloud:
    config:
      uri: http://xxx.xxx.xxx.xxx
      name: abc-admin-api
      username: Config 서버의 security에서 선언한 user.name
      password: Config 서버의 security에서 선언한 user.password
---
spring:
  config:
    activate:
      on-profile: prod
  cloud:
    config:
      uri: http://xxx.xxx.xxx.xxx
      name: abc-admin-api
      username: Config 서버의 security에서 선언한 user.name
      password: Config 서버의 security에서 선언한 user.password
```


