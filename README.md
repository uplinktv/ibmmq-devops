## Uplink TV - Install IBM MQ on CentOS/RedHat Linux

This document describes the steps for installing IBM MQ on CentOS/RedHat Linux

### Installation Steps

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
$ yum -y install bash bc ca-certificates file findutils gawk glibc-common grep passwd procps-ng sed shadow-utils tar util-linux which wget

$ wget -T5 -q -O mqadv_dev921_linux_x86-64.tar.gz https://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/messaging/mqadv/mqadv_dev921_linux_x86-64.tar.gz

$ tar -zxvf mqadv_dev921_linux_x86-64.tar.gz

$ cd MQServer

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

##### Configure per-user security

As the root, add a user and a group 
```
# groupadd ordergroup
# useradd -G ordergroup order
```

As the 'mqm' user, configure permissions
```
$ /opt/mqm/bin/setmqaut -m QMLAB1 -t qmgr -g ordergroup +connect +inq +dsp
$ /opt/mqm/bin/setmqaut -m QMLAB1 -n ORDER.** -t queue -g ordergroup +allmqi +dsp
```

As the 'mqm' user, configure channel auth
```
$ . /opt/mqm/bin/setmqenv -n Installation1
$ /opt/mqm/bin/runmqsc QMLAB1 << EOF

SET CHLAUTH(QMLAB1.SVRCONN) TYPE(ADDRESSMAP) ADDRESS('*') USERSRC(NOACCESS)
SET CHLAUTH(QMLAB1.SVRCONN) TYPE(USERMAP) CLNTUSER('order') USERSRC(MAP) MCAUSER('order') ADDRESS('*') ACTION(ADD)

EOF
```

As the 'mqm' user, validate channel auth
```
$ . /opt/mqm/bin/setmqenv -n Installation1
$ /opt/mqm/bin/runmqsc QMLAB1 << EOF

DISPLAY CHLAUTH(QMLAB1.SVRCONN) MATCH(RUNCHECK) CLNTUSER('order') ADDRESS('1.2.3.4')

EOF
```

##### Optionally, disable Channel Auth to allow _all_ network users to connect
```
$ . /opt/mqm/bin/setmqenv -n Installation1
$ /opt/mqm/bin/runmqsc QMLAB1 << EOF

ALTER QMGR CHLAUTH(DISABLED)

EOF
```

##### Optionally, create a user with admin-level privileges 'mqadmin'

Create UNIX account
```
$ useradd mqadmin
```

Apply IBM MQ QMgr privileges
```
$ . /opt/mqm/bin/setmqenv -n Installation1
$ /opt/mqm/bin/runmqsc QMLAB1 << EOF

SET AUTHREC PROFILE(*) OBJTYPE(QMGR) GROUP('mqadmin') AUTHADD(CONNECT, ALL) 
SET AUTHREC PROFILE('@class') OBJTYPE(AUTHINFO) GROUP('mqadmin') AUTHADD(CRT)
SET AUTHREC PROFILE('**') OBJTYPE(AUTHINFO) GROUP('mqadmin') AUTHADD(ALL)
SET AUTHREC PROFILE('@class') OBJTYPE(CHANNEL) GROUP('mqadmin') AUTHADD(CRT)
SET AUTHREC PROFILE('**') OBJTYPE(CHANNEL) GROUP('mqadmin') AUTHADD(ALL)
SET AUTHREC PROFILE('@class') OBJTYPE(CLNTCONN) GROUP('mqadmin') AUTHADD(CRT)
SET AUTHREC PROFILE('**') OBJTYPE(CLNTCONN) GROUP('mqadmin') AUTHADD(ALL)
SET AUTHREC PROFILE('@class') OBJTYPE(COMMINFO) GROUP('mqadmin') AUTHADD(CRT)
SET AUTHREC PROFILE('**') OBJTYPE(COMMINFO) GROUP('mqadmin') AUTHADD(ALL)
SET AUTHREC PROFILE('@class') OBJTYPE(LISTENER) GROUP('mqadmin') AUTHADD(CRT)
SET AUTHREC PROFILE('**') OBJTYPE(LISTENER) GROUP('mqadmin') AUTHADD(ALL)
SET AUTHREC PROFILE('@class') OBJTYPE(NAMELIST) GROUP('mqadmin') AUTHADD(CRT)
SET AUTHREC PROFILE('**') OBJTYPE(NAMELIST) GROUP('mqadmin') AUTHADD(ALL)
SET AUTHREC PROFILE('@class') OBJTYPE(PROCESS) GROUP('mqadmin') AUTHADD(CRT)
SET AUTHREC PROFILE('**') OBJTYPE(PROCESS) GROUP('mqadmin') AUTHADD(ALL)
SET AUTHREC PROFILE('@class') OBJTYPE(QUEUE) GROUP('mqadmin') AUTHADD(CRT)
SET AUTHREC PROFILE('**') OBJTYPE(QUEUE) GROUP('mqadmin') AUTHADD(ALL)
SET AUTHREC PROFILE('@class') OBJTYPE(SERVICE) GROUP('mqadmin') AUTHADD(CRT) 
SET AUTHREC PROFILE('**') OBJTYPE(SERVICE) GROUP('mqadmin') AUTHADD(ALL) 
SET AUTHREC PROFILE('@class') OBJTYPE(TOPIC) GROUP('mqadmin') AUTHADD(CRT) 
SET AUTHREC PROFILE('**') OBJTYPE(TOPIC) GROUP('mqadmin') AUTHADD(ALL) 

REFRESH SECURITY

EOF
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
