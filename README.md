# mongo-admin
Basic Cluster Administration

### mongod shell command

mongod --dbpath /data/db --logpath /data/log/mongod.log --fork --replSet "M103" --keyFile /data/keyfile --bind_ip "127.0.0.1,192.168.103.100" --tlsMode requireTLS --tlsCAFile "/etc/tls/TLSCA.pem" --tlsCertificateKeyFile "/etc/tls/tls.pem"

note : --bind_ip 127.0.0.1,192.168.103.100

#### Equivalent mongod.conf

```
storage:
  dbPath: /data/db/
  journal:
    enabled: true
#  engine:
#  mmapv1:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: first_mongod/mongod.log

# network interfaces
net:
  port: 27000
  bindIp: "127.0.0.1,192.168.103.100"


# how the process runs
processManagement:
   fork: true
   timeZoneInfo: /usr/share/zoneinfo
tls:
  mode: "requireTLS"
  certificateKeyFile: "/etc/tls/tls.pem"
  CAFile: "/etc/tls/TLSCA.pem"
security:
  keyFile: "/data/keyfile"
processManagement:
  fork: true
security:
  authorization: enabled
```  
  
### Create admin user 
```
mongo admin --host localhost:27000 --eval '
  db.createUser({
    user: "m103-admin",
    pwd: "m103-pass",
    roles: [
      {role: "root", db: "admin"}
    ]
  })
'
````  
  
  
### Assign conf file to mongod

mongod --config /etc/mongod.conf

## Common issue faced while using conf

1. ERROR: child process failed, exited with error number 1
To see additional information in this output, start without the "--fork" option.
vagrant@m103:~$ sudo nano /etc/mongod.conf
vagrant@m103:~$ mongod --config /etc/mongod.conf
2020-03-09T12:28:25.617+0000 F CONTROL  [main] Failed global initialization: FileNotOpen: Failed to open "/first_mongod/mongod.log"

  Solution : Make sure the folder is created, else create it using mkdir and assign RW access
  
  `sudo chown vagrant:vagrant first_mongod/mongod.log`
  
 2.
 2020-03-09T12:40:31.616+0000 E QUERY    [thread1] Error: listDatabases failed:{
	"ok" : 0,
	"errmsg" : "there are no users authenticated",
	"code" : 13,
	"codeName" : "Unauthorized"
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
Mongo.prototype.getDBs@src/mongo/shell/mongo.js:67:1
shellHelper.show@src/mongo/shell/utils.js:860:19
shellHelper@src/mongo/shell/utils.js:750:15
@(shellhelp2):1:1
MongoDB Enterprise > db.auth('m103-admin','m103-pass')
Error: Authentication failed.
  Solution: Login with user credentials 
  `mongo --host localhost:27000  --authenticationDatabase "admin" -u "m103-admin" -p "m103-pass"`
  
3. E QUERY    [thread1] Error: listDatabases failed:{
	"ok" : 0,
	"errmsg" : "not master and slaveOk=false",
	"code" : 13435,
	"codeName" : "NotMasterNoSlaveOk"
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
Mongo.prototype.getDBs@src/mongo/shell/mongo.js:67:1
shellHelper.show@src/mongo/shell/utils.js:860:19
shellHelper@src/mongo/shell/utils.js:750:15
@(shellhelp2):1:1

Solution

MongoDB Enterprise > `` rs.status() ``
{
	"info" : "run rs.initiate(...) if not yet done for the set",
	"ok" : 0,
	"errmsg" : "no replset config has been received",
	"code" : 94,
	"codeName" : "NotYetInitialized"
}
MongoDB Enterprise > ``rs.initiate() ``
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "192.168.103.100:27000",
	"ok" : 1
}
MongoDB Enterprise M103:OTHER> ``rs.status()``
{
	"set" : "M103",
	"date" : ISODate("2020-03-09T12:49:58.116Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"syncingTo" : "",
	"syncSourceHost" : "",
	"syncSourceId" : -1,
	"heartbeatIntervalMillis" : NumberLong(2000),
  ....... Not complete
  
  
  
  
