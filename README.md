## Uplink TV - Install IBM MQ


### Installation Steps

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for


##### Add UNIX user account 'mqm'

NOTE: This is a specific user account that IBM MQ *requires*

```
$ useradd mqm
```

##### Update system resource limits

```
$ vi /etc/security/limits.d/30-ibmmq.conf
# IBM MQ nofile limits
mqm 	- 	nofile 	65536
root	-	nofile	65536

$ sysctl -p
```

##### Install dependencies and IBM MQ RPMS

```
$ yum -y install bash bc ca-certificates file findutils gawk glibc-common grep passwd procps-ng sed shadow-utils tar util-linux which

$ wget -T5 -q -O mqadv_dev915_linux-x86-64.tar.gz https://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/messaging/mqadv/mqadv_dev915_linux_x86-64.tar.gz

$ ./mqlicense.sh -text_only -accept

$ rpm -Uvh MQSeriesAMQP-*.rpm MQSeriesAMS-*.rpm MQSeriesBCBridge-*.rpm MQSeriesClient-*.rpm MQSeriesGSKit-*.rpm MQSeriesJava-*.rpm MQSeriesJRE-*.rpm MQSeriesRuntime-*.rpm MQSeriesSamples-*.rpm MQSeriesSDK-*.rpm MQSeriesServer-*.rpm MQSeriesSFBridge-*.rpm MQSeriesXRService-*.rpm
```


##### Create the Queue Manager

```
$ . /opt/mqm/bin/setmqenv -n Installation1
$ /opt/mqm/bin/crtmqm -lc -lf 65535 -lp 3 -ls 2 -u SYSTEM.DEAD.LETTER.QUEUE QMLAB1
$ /opt/mqm/bin/strmqm QMLAB1
```


##### Create base objects within the Queue Manager

```
$ . /opt/mqm/bin/setmqenv -n Installation1
$ /opt/mqm/bin/runmqsc QMLAB1 << EOF

ALTER QMGR MAXMSGL(4194304)
ALTER QL(SYSTEM.CLUSTER.TRANSMIT.QUEUE) MAXMSGL(4194304)
DEFINE LISTENER(QMLAB1.LISTENER) TRPTYPE(TCP) CONTROL(QMGR) PORT(1414)
START LISTENER(QMLAB1.LISTENER)
DEFINE CHANNEL(QMLAB1.SVRCONN) CHLTYPE(SVRCONN) MCAUSER('mqm') MAXMSGL(4194304) DESCR('Channel for incoming clients')
DEFINE QL(ORDER.INPUT)

end
EOF
```

##### Test sending and receiving a message

```
$ printf "%s\n\n" TestMessage1 | /opt/mqm/samp/bin/amqsput ORDER.INPUT QMLAB1
$ /opt/mqm/samp/bin/amqsget ORDER.INPUT QMLAB1
```


### Uninstall steps

1. Stop the running Queue Manager
2. Delete the created Queue Manager
3. Uninstall the IBM MQ RPMS

``` 
$ . /opt/mqm/bin/setmqenv -n Installation1
$ /opt/mqm/bin/endmqm QMLAB1
$ /opt/mqm/bin/dltmqm QMLAB1
$ yum -y erase MQSeriesAMQP MQSeriesAMS MQSeriesBCBridge MQSeriesClient MQSeriesGSKit MQSeriesJava MQSeriesJRE MQSeriesRuntime MQSeriesSamples MQSeriesSDK MQSeriesServer MQSeriesSFBridge MQSeriesXRService
```

### Links

IBM MQ default objects [IBM MQ documentation](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.1.0/com.ibm.mq.ref.con.doc/q081590_.htm)

IBM MQ on Linux Installation video [UplinkTV YouTube video](https://www.youtube.com/watch?v=DVKt_ytI7OQ)

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
