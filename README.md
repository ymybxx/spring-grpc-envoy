# spring-grpc-envoy

本例子实现了springboot内部grpc调用，同时外部通过envoy将rest请求转换为grpc请求

如需了解浏览器如何直接调用grpc可以详细https://github.com/grpc/grpc-web

grpc-web也是需要通过envoy转发的

### 前置准备
1.安装protoc

[release](https://github.com/protocolbuffers/protobuf/releases)

配置环境变量
export PATH=$PATH:/Users/inequality/Downloads/protoc-3.11.3-osx-x86_64/bin


2.下载googleapis

git clone https://github.com/googleapis/googleapis.git

配置环境变量
export GOOGLEAPIS_DIR="/Users/inequality/lib/googleapis"


3.下载protobuf

https://github.com/protocolbuffers/protobuf.git

配置环境变量
export PBPATH="/Users/inequality/lib/protobuf"

4.安装docker

5.IDEA安装插件：protobuf-support

### 项目结构
grpc-lib: 公共包，存放proto文件 和通过protobuf 生成的java文件

grpc-server: 服务端

grpc-client: spring客户端，可以内部直接调用

## grpc-lib

#### proto文件，及其编译方法
greeter.pb
```
syntax = "proto3";

option java_package = "com.yyx.grpc.lib";
import "google/api/annotations.proto";

// The greeter service definition.
service Greeter {
    // Sends a greeting
    rpc SayHello ( HelloRequest) returns (  HelloReply) {
        option (google.api.http) = {        // #add
                   post: "/say"            // #add
                      body: "*"              // #add

             };
    }

}
// The request message containing the user's name.
message HelloRequest {
    string name = 1;
}
// The response message containing the greetings
message HelloReply {
    string message = 1;
}

重点是和一般proto不同的是option (google.api.http)来定义对外的rest接口

```

在grpc-lib/pom 中配置maven编译插件

```
<build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>${os.plugin.version}</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>${protobuf.plugin.version}</version>
                <extensions>true</extensions>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:${protoc.version}:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
                    <!--默认值-->
                    <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
                    <!--默认值-->
                    <!--<outputDirectory>${project.build.directory}/generated-sources/protobuf/java</outputDirectory>-->
                    <outputDirectory>${project.basedir}/src/main/java</outputDirectory>
                    <!--设置是否在生成java文件之前清空outputDirectory的文件，默认值为true，设置为false时也会覆盖同名文件-->
                    <clearOutputDirectory>false</clearOutputDirectory>
                    <additionalProtoPathElements>
                        <additionalProtoPathElement>${GOOGLEAPIS_DIR}</additionalProtoPathElement>
                        <additionalProtoPathElement>${PBPATH}/src</additionalProtoPathElement>
                    </additionalProtoPathElements>
                    <!--更多配置信息可以查看https://www.xolstice.org/protobuf-maven-plugin/compile-mojo.html-->
                </configuration>
                <executions>
                    <execution>
                        <!--在执行mvn compile的时候会执行以下操作-->
                        <phase>compile</phase>
                        <goals>
                            <!--生成OuterClass类-->
                            <goal>compile</goal>
                            <!--生成Grpc类-->
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

因为用到了googleapi来配置restful接口，所以需要additionalProtoPathElements
用到了一开始配置的GOOGLEAPIS_DIR PBPATH和PBPATH

配置完插件后可以在grpc-lib下执行mvn compile 编译生成java

#### grpc-server

application.yml配置grpc端口
```
spring:
  application:
    name: local-grpc-server
grpc:
  server:
    port: 9898
```
编写对应service

#### grpc-client（内部方式）

client中可以不写controller,直接内部调用

application.yml
```
server:
  port: 9090
spring:
  application:
    name: local-grpc-client
grpc:
  client:
    local-grpc-server:
      host:
      - ${LOCAL-GRPC-HOST:127.0.0.1}
      port:
      - 9898
      enableKeepAlive: true
      keepAliveWithoutCalls: true
```

#### 通过envoy proxy 实现rest请求转grpc

1.生成pb文件

在grpc-lib中的proto文件夹下执行

protoc -I$GOOGLEAPIS_DIR -I. --include_imports --include_source_info   --descriptor_set_out=greeter.pb  greeter.proto

2.创建envoy.yaml文件
```
static_resources:
  listeners:
  - name: listener1
    address:
      socket_address: { address: 0.0.0.0, port_value: 51051 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: grpc_json
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: grpc, timeout: { seconds: 60 } }
          http_filters:
          - name: envoy.grpc_json_transcoder
            config:
              proto_descriptor: "/data/envoy/proto.pb"     #
              services: ["Greeter"]             #
              print_options:
                add_whitespace: true
                always_print_primitive_fields: true
                always_print_enums_as_ints: false
                preserve_proto_field_names: false
          - name: envoy.router

  clusters:
  - name: grpc
    connect_timeout: 1.25s
    type: logical_dns
    lb_policy: round_robin
    dns_lookup_family: V4_ONLY
    http2_protocol_options: {}
    load_assignment:
      cluster_name: grpc
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                  address: host.docker.internal
                  port_value: 9898
```

主要是利用了 evnoy 的
http_filters:
         - name: envoy.grpc_json_transcoder

这里监听51051端口转发给服务端grpc的9898端口

3.启动grpc-server

4.运行docker容器
docker run -it --rm --name envoy -p 51051:51051  \
    -v "$(pwd)/greeter.pb:/data/envoy/proto.pb:ro" \
      -v "$(pwd)/envoy.yaml:/etc/envoy/envoy.yaml:ro" \
      envoyproxy/envoy
      

5.通过curl 或者 postman请求即可

curl -X POST \
  http://localhost:51051/say \
  -H 'Content-Type: application/json' \
  -d '{
	"name":"banana"
}'

返回

{
    "message": "Hello banana"
}





