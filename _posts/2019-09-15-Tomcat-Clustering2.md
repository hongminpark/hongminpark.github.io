---
layout: post
title: "Tomcat Clustering - 2편"
author: "Hongmin Park"
tags: tomcat
---
[Tomcat Clustering - 1편](https://hongminpark.github.io/2019-09-10/Tomcat-Clustering)에서는 간단히 클러스터링을 구현하는 방법을 알아보았다. 이번 편에서는 좀 더 자세히 각 설정 값들이 의미하는 바를 알아볼 것이다.
모든 설명은 [Tomcat 8.0 Documentation](https://tomcat.apache.org/tomcat-8.0-doc/cluster-howto.html)을
기준으로 ~~(거의 번역에 가깝게..)~~ 작성하였다.
<br><br>
Tomcat 클러스터링은 아래와 같이 `server.xml`에 설정해야하는데, 각 태그들이 의미하는 바를 자세히 알아보겠다.
```xml
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
         channelSendOptions="8">

  <Manager className="org.apache.catalina.ha.session.DeltaManager"
           expireSessionsOnShutdown="false"
           notifyListenersOnReplication="true"/>

  <Channel className="org.apache.catalina.tribes.group.GroupChannel">
    <Membership className="org.apache.catalina.tribes.membership.McastService"
                address="228.0.0.4"
                port="45564"
                frequency="500"
                dropTime="3000"/>
    <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
              address="auto"
              port="4000"
              autoBind="100"
              selectorTimeout="5000"
              maxThreads="6"/>

    <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
      <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
    </Sender>
    <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
    <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/>
  </Channel>

  <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
         filter=""/>
  <Valve className="org.apache.catalina.ha.session.JvmRouteBinderValve"/>

  <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
            tempDir="/tmp/war-temp/"
            deployDir="/tmp/war-deploy/"
            watchDir="/tmp/war-listen/"
            watchEnabled="false"/>

  <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
</Cluster>
```
<br>
## SimpleTcpCluster
```xml
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
         channelSendOptions="6">
```
[Cluster](https://tomcat.apache.org/tomcat-8.0-doc/config/cluster.html)는 톰캣에서 클러스터링을 관리해주는 클래스 이름이다. 기본적으로는 all-to-all 구조로 세션을 백업하는 
`DeltaManager`에 의해 세션이 관리된다. all-to-all이 아닌 호스트를 지정하여 세션을 백업하도록 하는 방식은
`BackupManager`를 통해 구현한다.<br><br>
`channelSendOptions`, `channelStartOptions`는 tomcat이 세션백업을 위해 메시지를 보낼 때에 어떤 방식으로 보낼것인가에 대한 옵션이다. 
TCP 통신에 대한 설정이다. Async/Sync 방식, 세션을 보낸 후 응답여부를 확인할지 등의 여부를 설정할 수 있다.
<br><br>
여기서, `channelSendOptions="6"`은
```
Channel.SEND_OPTIONS_SYNCHRONIZED_ACK = 0x0004
Channel.SEND_OPTIONS_ASYNCHRONOUS = 0x0008
Channel.SEND_OPTIONS_USE_ACK = 0x0002
```
중 1,3번째를 더한 값이기 때문에 두 옵션을 채택한 SYNC 방식으로 ACK를 확인하겠다는 뜻이다.
`channelStartOptions`는
```
Channel.DEFAULT = Channel.SND_RX_SEQ (1) |
                  Channel.SND_TX_SEQ (2) |
                  Channel.MBR_RX_SEQ (4) |
                  Channel.MBR_TX_SEQ (8);
```
값들 중의 값인데, 현재 3으로 설정한  SND_RX_SEQ, SND_RX_SEQ를 설정하겠다는 의미이다. 
```
SND_RX_SEQ - starts or stops the data receiver. Start means opening a server socket in case of a TCP implementation
SND_TX_SEQ - starts or stops the data sender. This should not open any sockets, as sockets are opened on demand when a message is being sent
```
각각의 의미는 [documentation](https://tomcat.apache.org/tomcat-7.0-doc/api/org/apache/catalina/tribes/Channel.html)
에서 자세히 확인할 수 있다. 사실 명확이 이해하지 못해서 설명을 달 수가 없다. 
나중에 이해가 가는 순간이 온다면 다시 봐야겠다.

## Channel
```
<Channel className="org.apache.catalina.tribes.group.GroupChannel">
```
Channel은 톰캣 안에서 그룹 통신을 지원하는 프레임웤이다. 채널 안에 속한 요소들을 Tribes라고 한다. 
Channel element는 membership 간 통신을 구현한다.
```
      <Membership className="org.apache.catalina.tribes.membership.McastService"
                  address="228.0.0.4"
                  port="45564"
                  frequency="500"
                  dropTime="3000"/>
```
[Membership](https://tomcat.apache.org/tomcat-8.0-doc/config/cluster-membership.html)은 기본적으로 [multicast](https://ko.wikipedia.org/wiki/%EB%A9%80%ED%8B%B0%EC%BA%90%EC%8A%A4%ED%8A%B8)를 통해 구현된는데, 
`StaticMembershipIntercepter`를 통해 multicasting이 아닌 static membership로도 정의할 수 있다. Static member는 해당 노드의 IP, 통신할Port를 직접 명시하여 작성한다.
(현재 우리는 어떤 이유에서인지 multicast 통신이 되지 않아서 StaticMembership을 사용하고 있다.)
<br><br>
`Membership`은 TCP 주소/포트로 broadcasts를 통해 노드 간 연결한다. 이 때 broadcast 되는 주소는 Receiver.address 이다.
(사실 나는 broadcast, multicast의 차이와 왜 ip는 228.0.0.4와 같이 이상한 숫자를 쓰는지 명확히 모르고있다.
[이 글](https://www.esds.co.in/blog/difference-between-unicast-broadcast-and-multicast/#sthash.JljMsohF.dpbs)에서 자세히 설명하고 있는데,
꼭 시간을 내서 읽어보고 따로 정리해야겠다.)<br>
`Channel`의 `Membership` 구성요소는 클러스터에 있는 멤버(노드)를 동적으로 찾는 역할을 한다. 
기본적으로 multicast IP에 UDP packet 을 보냄으로써 multicast hearbeats 을 수신하고, heartbeat을 통해서 동적 탐색을 한다.
## Sender, Receiver
```
      <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                address="auto"
                port="5000"
                selectorTimeout="100"
                maxThreads="6"/>
      <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter">
        <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
      </Sender>
```
Receiver에 정의된 address로 다른 노드로부터 broadcast 되어 데이터를 받을 수 있다.
[Receiver](https://tomcat.apache.org/tomcat-8.0-doc/config/cluster-receiver.html)와 [Sender](https://tomcat.apache.org/tomcat-8.0-doc/config/cluster-sender.html)는 `Blocking`과 `Non-Blocking` 방식 두 가지를 지원한다.
`Blocking`은 parallel하게, `NIO`는 concurrent하게 메시지를 보낸다. 
Concurrent하다는 것은 한 메시지를 한 번에 다수의 senders 에게 보내는 것이고 Parallel하다는 것은 여러 메시지를 한 번에 다수의 senders에게 보낸다는 뜻이다.
## Interceptor
```
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/>
            <Interceptor className="org.apache.catalina.tribes.group.interceptors.ThroughputInterceptor"/>
          </Channel>
```
Channel이 메시지를 보낼 때에 stack으로 보낸다. stack에 있는 요소들을 [Interceptor](https://tomcat.apache.org/tomcat-8.0-doc/config/cluster-interceptor.html) 라고 하는데, `servel.xml`의 `Valve` 처럼 동작한다.
Interceptor는 순서가 있어 위와 같이 정의된 순서에 따라서 channel stack에 쌓인다.
## Deployer
```
          <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
                    tempDir="/tmp/war-temp/"
                    deployDir="/tmp/war-deploy/"
                    watchDir="/tmp/war-listen/"
                    watchEnabled="false"/>
```
톰캣 클러스터는 디폴트로 [farmed deployment](https://tomcat.apache.org/tomcat-8.5-doc/config/cluster-deployer.html)를 지원한다. 클러스터에 속한 모든 노드에 앱을 디플로이해주는 속성이다.
실제로 배포를 이 방식으로 하지는 않을 것 같아서 필요하지 않는 경우에는 삭제해도 될 것 같다.

## 클러스터 작동 순서
마지막으로 Tomcat A, Tomcat B가 서로 클러스터링 되어있을 때에 어떤 방식으로 세션백업이 동작하는지 그 순서에 대해 알아보자.
1. TomcatA 기동<br>
일반 기동과 동일, ClusterManager인 DeltaManaer를 생성한다. cluster class는  membership service와 replication service 를 기동시킨다.
2. TomcatB 기동<br>
클러스터에 속한 서버에 session state를 요청한다. HTTP 요청을 listen하기 전에(**<=> 리슨 포트가 뜨기 이전**) 세션을 백업받는다. 이 때 TomcatA가 응답이 없을 수도 있으므로 타임아웃은 60s 이다.
3. TomcatA로 요청을 받고, 세션 S1이 생성됨.<br>
request 요청을 받으면 ReplicationValve가 요청을 intercept해서 response를 돌려주기 이전에 복제 작업을 한다. serialized 된 데이터가 클러스터에 속한 다른 노드의 **OS TCP logic에 도착하면** 그 때 톰캣이 response를 돌려준다.
4. TomcatA가 죽음<br>
TomcatB는 TomcatA가 죽으면 notification을 받아 membership list에서 제거한다. 로드밸런서가 TomcatA로부터의 요청을 TomcatB로 리다이렉트 시키고, TomcatB가 세션 S1 요청에 대한 응답을 돌려준다.
5. TomcatA 살아남<br>
다시 TomcatA가 살아나면, TomcatB에게 현재 세션 상태를 업데이트 받고 HTTP/mod_jk 포트를 오픈한다.
6. 세션 S1 만료<br>
세션 S1이 클라이언트 요청에 의해 만료되면 invalidate call이 intercept해서 invalidated session 에 큐잉된다. TomcatA에서 S1이 invalidate되면, TomcatB로 expire메시지를 보내고 TomcatB에서도 invalidate한다.
<br><br>
이렇게 클러스터링을 위한 각 설정의 의미와 클러스터링 동작 순서에 대해 알아보았다. 세션이 다른 노드로 백업된 이후에 응답을 돌려준다고 하는데, 이 부분도 TCP 옵션설정 중 비동기방식으로 설정을 하면 기다리는 시간을 줄일 수 있지 않을까 싶다. 사실 설명은 위와 같지만 실제 운영중 일어나는 이슈를 경험해야 어떤 부분이 문제가 되고 어떻게 해결해야하는지 체감할 수 있을 것 같다.   



