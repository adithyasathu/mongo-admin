# mongo-admin
Basic Cluster Administration

### mongod shell command
```
mongod --dbpath /data/db --logpath /data/log/mongod.log --fork --replSet "M103" --keyFile /data/keyfile --bind_ip "127.0.0.1,192.168.103.100" --tlsMode requireTLS --tlsCAFile "/etc/tls/TLSCA.pem" --tlsCertificateKeyFile "/etc/tls/tls.pem"
```
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
```
  
  
### Assign conf file to mongod
```
mongod --config /etc/mongod.conf
```
## Common issue faced while using conf

1. ERROR: child process failed, exited with error number 1
To see additional information in this output, start without the "--fork" option
```
vagrant@m103:~$ sudo nano /etc/mongod.conf
vagrant@m103:~$ mongod --config /etc/mongod.conf
```
2020-03-09T12:28:25.617+0000 F CONTROL  [main] Failed global initialization: FileNotOpen: Failed to open "/first_mongod/mongod.log"

_Solution_: Make sure the folder is created, else create it using mkdir and assign RW access
  
  ```
  sudo chown vagrant:vagrant first_mongod/mongod.log
  ```
  
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

_Solution_: Login with user credentials 
  ```
  mongo --host localhost:27000  --authenticationDatabase "admin" -u "m103-admin" -p "m103-pass"
  ```
  
3.  QUERY    [thread1] Error: listDatabases failed:{
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

_Solution_:
Note: rs is the replication set

MongoDB Enterprise > 
```
rs.status() 
{
	"info" : "run rs.initiate(...) if not yet done for the set",
	"ok" : 0,
	"errmsg" : "no replset config has been received",
	"code" : 94,
	"codeName" : "NotYetInitialized"
}
```
MongoDB Enterprise > 
```
rs.initiate() 
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "192.168.103.100:27000",
	"ok" : 1
}
```
MongoDB Enterprise M103:OTHER>
```
rs.status()
{
	"set" : "M103",
	"date" : ISODate("2020-03-09T12:49:58.116Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"syncingTo" : "",
	"syncSourceHost" : "",
	"syncSourceId" : -1,
	"heartbeatIntervalMillis" : NumberLong(2000),
 ( Not complete)
  ```
   

4.  Different DB path than the default path /data/db. Mongod will now store data files in this new directory instead.
_Solution_:
create a new folder /var/mongodb/db/ and allow mongod to write files to this directory
create this directory with sudo, because /var is owned by root
use chown to change the owner of this directory to vagrant:vagrant
edit your config file to use this new directory as the dbpath
  
  ```
  use admin
  db.shutdownServer()
  quit()
  ```
Start to see reflected changes

5.
```
~$ mongod --config /etc/mongod.conf
Aborted (core dumped)
```
_Solution_: 
Likely there is an other instance of mongod running. Find the process and kill it

**To find process**
```
vagrant@m103:~$ sudo lsof -i -P -n | grep 27000 
mongod    2109 vagrant   12u  IPv4  13601      0t0  TCP 127.0.0.1:27000 (LISTEN)
mongod    2109 vagrant   13u  IPv4  13602      0t0  TCP 192.168.103.100:27000 (LISTEN)
mongo     2138 vagrant    4u  IPv4  13619      0t0  TCP 127.0.0.1:56336->127.0.0.1:27000 (ESTABLISHED)


vagrant@m103:~$ sudo lsof -i -P -n | grep mongod
mongod    2109 vagrant   12u  IPv4  13601      0t0  TCP 127.0.0.1:27000 (LISTEN)
mongod    2109 vagrant   13u  IPv4  13602      0t0  TCP 192.168.103.100:27000 (LISTEN)
```
**To kill the process**
```kill -9 2109```

6. Detected unclean shutdown - /data/db/mongod.lock is not empty.

_Solution_: 
```
mongod --repair
```

**Not recommanded but if you want to repair your data files without preserving the original files**
```
sudo rm /var/lib/mongodb/mongod.lock
sudo mongod --dbpath /var/lib/mongodb/ --repair
sudo mongod --dbpath /var/lib/mongodb/ --journal
```
Make sure that you leave you terminal running in which you have run above lines, don't press 'Ctrl+c' or quit it. Type the command to start mongo now in another window.

7. Error: couldn't connect to server localhost:27000, connection attempt failed: SocketException: Error connecting to localhost:27000 (127.0.0.1:27000) :: caused by :: Connection refused :
connect@src/mongo/shell/mongo.js:343:13
@(connect):2:6
exception: connect failed

_Hint_: Check the mongod logs for failure

8. exception in initAndListen: 29 Data directory /data/db not found., terminating
_Solution_:
Error happen because dbpath /data/db/ (default config) does not exist. You need to create data folder and set permission for it.

```
sudo mkdir -p /data/db/ 
sudo chown id -u /data/db
```

then start the mongo


   
