# Kerberos

# Concepts

- _ticket_: proof of identity encrypted with a secret key for the particular service requested.

- _realm_: defines what Kerberos manages in terms of who can access what.

- _Principal_: is a unique identity to which Kerberos can assign tickets to.

- _keytab_: A keytab is a file containing pairs of Kerberos principals and encrypted keys (which are derived from the Kerberos password). You can use a keytab file to authenticate to various remote systems using Kerberos without entering a password.

- _KDC_: Key Distribution center. It consists in an _Authentication Server_ and a _Ticket Granting Server_.

- _Service or host machine_: the machine to access to.

For further information go to [kerberos official site][kerberos-official-site]


# How to's

## Login

`kinit \<PRINCIPAL\>`

## Login using keytab

`kinit -k -t \<KEYTAB_FILE_PATH\> \<PRINCIPAL\>`

## Logout

`kdestroy -A`

## Get all principals

type `kadmin.local` once logged in `listprincs`

## Create a keytab file

type `kadmin.local` once logged in `xst -norandkey -k <KEYTAB_FILE_PATH> <PRINCIPALS_CSV>`. This will generate a file containing all the given principals.

## Reset admin password

type `kadmin.local` once logged in `change_password <PRINCIPAL>`.

`kadmin.local` has more privileges that `kadmin` so it is advisable to use in certain cases (for example to reset the admin password as you are not required to login).

## Changing renewal time

type `kadmin.local` once logged in `modprinc -maxrenewlife 1day <PRINCIPAL>`.

# CentOS

## Kerberos Server Installation

This guide is intended to do a quick installation of kerberos in centos. It is based on the official CentOS [documentation][kerberos-centos-doc]

When Kerberos is used accross many servers Host visibility and time syncronization is required. Refer to the official documentation to configure this.

- Install kerberos packages:

```bash
yum -y install krb5-libs krb5-server krb5-workstation
```

For this example we will use the domain `joseestudillo.com` and machine called `kerberos` where the kerberos server will be installed

- Kerberos configuration file `/etc/krb5.conf`:

Default content
```bash
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = JOSEESTUDILLO.COM
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true

[realms]
 JOSEESTUDILLO.COM = {
  kdc = kerberos.joseestudillo.com
  admin_server = kerberos.joseestudillo.com
 }

[domain_realm]
 .joseestudillo.com = JOSEESTUDILLO.COM
 joseestudillo.com = JOSEESTUDILLO.COM
```  

- KDC configuration file `/var/kerberos/krb5kdc/kdc.conf`:

```bash
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 JOSEESTUDILLO.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

- Once the the file above have been created, It is required to initialize the database that stores keys for the Kerberos realms:

```bash
kdb5_util create -s
```

the database will be created by default under `/var/kerberos/krb5kdc/`

- create the file `/var/kerberos/krb5kdc/kadm5.acl`:

```bash
*/admin@JOSEESTUDILLO.COM	*
```

This file is used by `kadmin` to determine which principals have administrative access to the Kerberos database and their level of access.


- Create a principal

```bash
kadmin.local -q "addprinc <USERNAME>/admin"
```

to for example create a principal for myself:

```bash
kadmin.local -q "addprinc joseestudillo/admin"
```
so the generated principal would be `joseestudillo/admin@JOSEESTUDILLO.COM`

- ensure that the server names can be resolved:

/etc/hosts:

```bash
127.0.0.1 kerberos.joseestudillo.com
```

- start services:

```bash
service krb5kdc start
service kadmin start
```

It is recommended to start these start these services when the box starts:

```bash
chkconfig krb5kdc on
chkconfig kadmin on
```

## Kerberos Client Installation

Install the following packages `yum -y install krb5-libs krb5-auth-dialog krb5-workstation` and then create the file `/etc/krb5.conf` with the same Kerberos configuration set in the cluster, then use `kinit <PRINCIPAL>` to test the access.


# Mac OSX


## Client Installation

At the time of this writing, using OSX's latest version, there is no need to install a kerberos client, you just need to create the file `/etc/krb5.conf` with the same Kerberos configuration set in the cluster, then use `kinit <PRINCIPAL>` to test the access.


# Hortonworks Sandbox Installation

This VM is distributed installed in CentOS. So the configuration described above will be similar replacing `JOSEESTUDILLO.COM` with `HORTONWORKS.COM` and the same with the lowercase version (`joseestudillo.com` with `hortonworks.com`) (Case sensitive!!)


- Initialize the realm database `kdb5_util create -s`, in my case the output was:

```bash
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'HORTONWORKS.COM',
master key name 'K/M@HORTONWORKS.COM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key:
Re-enter KDC database master key to verify:
```

- add a principal, in this case I will use `root`:

`/usr/kerberos/sbin/kadmin.local -q "addprinc root/admin"`

- start the services:

`service krb5kdc start && service kadmin start`

- and ensure the services start automatically (this is very important otherwise the services won't be able to access to each other!):

```bash
chkconfig krb5kdc on
chkconfig kadmin on
```


## Running the Hortonworks kerberos wizard

I have partially followed the instructions in the official [Hortonworks documentation][hortonworks-kerberos-wizard], so this part should be straight forward. Just for clarification, in this case, the values to be introduced during this process are:

- _KDC Host_: `sandbox.hortonworks.com`
- _Realm Name_: `HORTONWORKS.COM`
- _Domains_: `.hortonworks.com,hortonworks.com`
- _Kadmin Host_: `sandbox.hortonworks.com`
- _Admin principal_: `root/admin@HORTONWORKS.COM`
- _Admin Password_: the password you set during the principal creation


## Generating a Keytab with all the principals

As explained before in the how to's, we can get a list of all principals as follows:

```bash
kadmin:  listprincs
HTTP/sandbox.hortonworks.com@HORTONWORKS.COM
K/M@HORTONWORKS.COM
ambari-qa-Sandbox@HORTONWORKS.COM
amshbase/sandbox.hortonworks.com@HORTONWORKS.COM
amszk/sandbox.hortonworks.com@HORTONWORKS.COM
atlas/sandbox.hortonworks.com@HORTONWORKS.COM
dn/sandbox.hortonworks.com@HORTONWORKS.COM
falcon/sandbox.hortonworks.com@HORTONWORKS.COM
hbase-Sandbox@HORTONWORKS.COM
hbase/sandbox.hortonworks.com@HORTONWORKS.COM
hdfs-Sandbox@HORTONWORKS.COM
hive/sandbox.hortonworks.com@HORTONWORKS.COM
jhs/sandbox.hortonworks.com@HORTONWORKS.COM
kadmin/admin@HORTONWORKS.COM
kadmin/changepw@HORTONWORKS.COM
kadmin/sandbox.hortonworks.com@HORTONWORKS.COM
kafka/sandbox.hortonworks.com@HORTONWORKS.COM
knox/sandbox.hortonworks.com@HORTONWORKS.COM
krbtgt/HORTONWORKS.COM@HORTONWORKS.COM
nfs/sandbox.hortonworks.com@HORTONWORKS.COM
nimbus/sandbox.hortonworks.com@HORTONWORKS.COM
nm/sandbox.hortonworks.com@HORTONWORKS.COM
nn/sandbox.hortonworks.com@HORTONWORKS.COM
oozie/sandbox.hortonworks.com@HORTONWORKS.COM
rm/sandbox.hortonworks.com@HORTONWORKS.COM
root/admin@HORTONWORKS.COM
spark-Sandbox@HORTONWORKS.COM
storm@HORTONWORKS.COM
yarn/sandbox.hortonworks.com@HORTONWORKS.COM
zookeeper/sandbox.hortonworks.com@HORTONWORKS.COM
```

generating a keytab file with all the principals is not a good practice but it will make the access to all the services easier, so the following string will do the job:  

```bash
xst -norandkey -k /tmp/full-keytab.keytab HTTP/sandbox.hortonworks.com@HORTONWORKS.COM K/M@HORTONWORKS.COM ambari-qa-Sandbox@HORTONWORKS.COM amshbase/sandbox.hortonworks.com@HORTONWORKS.COM amszk/sandbox.hortonworks.com@HORTONWORKS.COM atlas/sandbox.hortonworks.com@HORTONWORKS.COM dn/sandbox.hortonworks.com@HORTONWORKS.COM falcon/sandbox.hortonworks.com@HORTONWORKS.COM hbase-Sandbox@HORTONWORKS.COM hbase/sandbox.hortonworks.com@HORTONWORKS.COM hdfs-Sandbox@HORTONWORKS.COM hive/sandbox.hortonworks.com@HORTONWORKS.COM jhs/sandbox.hortonworks.com@HORTONWORKS.COM kadmin/admin@HORTONWORKS.COM kadmin/changepw@HORTONWORKS.COM kadmin/sandbox.hortonworks.com@HORTONWORKS.COM kafka/sandbox.hortonworks.com@HORTONWORKS.COM knox/sandbox.hortonworks.com@HORTONWORKS.COM krbtgt/HORTONWORKS.COM@HORTONWORKS.COM nfs/sandbox.hortonworks.com@HORTONWORKS.COM nimbus/sandbox.hortonworks.com@HORTONWORKS.COM nm/sandbox.hortonworks.com@HORTONWORKS.COM nn/sandbox.hortonworks.com@HORTONWORKS.COM oozie/sandbox.hortonworks.com@HORTONWORKS.COM rm/sandbox.hortonworks.com@HORTONWORKS.COM root/admin@HORTONWORKS.COM spark-Sandbox@HORTONWORKS.COM storm@HORTONWORKS.COM yarn/sandbox.hortonworks.com@HORTONWORKS.COM zookeeper/sandbox.hortonworks.com@HORTONWORKS.COM
```
Now we can copy the file `/tmp/full-keytab.keytab` to the client computer to access to the different services.

## Issues found

### Fixing access problems after restarting

set the principals to use the proper host

`grep -l -r "\/_HOST" /etc/hadoop/ | xargs sed -i -e "s&/_HOST&/sandbox.hortonworks.com&g"`



# Accessing to services


## Hive

I have found good documentation in the [Tableau website][tableau-kerberos-hive] but not all the methods they explain worked for me.

### Beeline

The following is supposed to be right, but It didn't work for me on Linux Mint neither on a mac.

```bash
kinit --use-keytab --keytab=<KEYTAB_FILE_PATH> <HIVE_PRINCIPAL>
beeline -u <JDBC_CONN_STR>
```

The following approach worked for both:

```bash
beeline --hiveconf hive.server2.authentication=kerberos --hiveconf hive.metastore.kerberos.keytab.file=<KEYTAB_FILE_PATH>
--hiveconf hive.metastore.kerberos.principal=<HIVE_PRINCIPAL>
-u <JDBC_CONN_STR>
```

for a hortonworks sandbox instance, the values could be:

- `KEYTAB_FILE_PATH`: `/path/to/file.keytab` in this case I have used a keytab file that contained all the principals.
- `HIVE_PRINCIPAL`: ``hive/sandbox.hortonworks.com@HORTONWORKS.COM`
- `JDBC_CONN_STR`: `jdbc:hive2://sandbox.hortonworks.com:10000/default\;principal=<HIVE_PRINCIPAL>`


### HIVE JDBC

Besides adding the principal information in the JDBC string

`jdbc:hive2://sandbox.hortonworks.com:10000/default\;principal=<HIVE_PRINCIPAL>`

You need to add the following

```Java
private static final String HADOOP_AUTH_PROP = "hadoop.security.authentication";
private static final String HADOOP_AUTH_VAL_KERBEROS = "kerberos";

...

if (!principal.isEmpty() && !keytabPath.isEmpty()) {
	log.info("Configuring kerberos access...");
	Configuration conf = new Configuration();
	conf.set(HADOOP_AUTH_PROP, HADOOP_AUTH_VAL_KERBEROS);
	UserGroupInformation.setConfiguration(conf);
	UserGroupInformation.loginUserFromKeytab(principal, keytabPath);
	log.info(String.format("Kerberos access configured. {principal: %s, keytab_file: %s}", principal, keytabPath));
}
```

where `principal` is the Hive principal string and `keytabPath` the path to the keytab file.


## Hadoop


### HDFS CLI tool

`hdfs --config ./resources/config-hortonworks dfs -ls /`

for this case I always get the same error:

```
WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
ls: Failed on local exception: java.io.IOException: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]; Host Details : local host is: "<MY_HOSTNAME>"; destination host is: "sandbox.hortonworks.com":8020;
```

even having a valid kerberos ticket locally.

[TODO FIX]

## HDFS Hadoop API

This case will be similar the Hive JDBC access so given a Hadoop configuration stored in `conf` something like the code below will do the job:

```java
conf.set("hadoop.security.authentication", "kerberos");
UserGroupInformation.setConfiguration(conf);
UserGroupInformation.loginUserFromKeytab(principal, keytabPath);
```

There is a complete example of this in the hadoop-quick-start project in this repository.


# Cloudera Sandbox Installation

Cloudera is distributed using CentOS, which keberos installation process is described above.

In this case we will use the server `quickstart.cloudera` and the realm `QUICKSTART.CLOUDERA` so `/etc/krb5.conf` should look like:

```bash
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = QUICKSTART.CLOUDERA
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true

[realms]
 QUICKSTART.CLOUDERA = {
  kdc = quickstart.cloudera
  admin_server = quickstart.cloudera
 }

[domain_realm]
 .quickstart.cloudera = QUICKSTART.CLOUDERA
 quickstart.cloudera = QUICKSTART.CLOUDERA
```

The command below will do all the substitutions automatically:

`sed -i -e "s/kerberos.//g" -e "s/example.com/quickstart.cloudera/g" -e "s/EXAMPLE.COM/QUICKSTART.CLOUDERA/g" /etc/krb5.conf`

Notice that this will require `quickstart.cloudera` to be in the `/etc/hosts` file.

## Cloudera manager

go to `administration -> security` and then `enable kerberos`

You will be asked to check several ticks in order to proceed, if you have done the previous steps, you are good to go.

In the next screen you will have:

- _KDC Type_:
- _KDC Server Host_: `quickstart.cloudera`
- _Kerberos Security Realm_: `QUICKSTART.CLOUDERA`
- _Kerberos Encryption Types_: set to `rc4-hmac`, no reason to change it.
- _Renewal time_: 5 days, this can be set according to your needs.

We will allow cloudera to manage the `krb5.conf` file.

After the previous selection we need to provide the administrator principal we generated before `root/admin@QUICKSTART.CLOUDERA` and the password we set for it.

For the rest of the sections nothing has to be changed, so continue, continue, continue...

Once the process has finished the list of principals should look as follows:

```bash
HTTP/quickstart.cloudera@QUICKSTART.CLOUDERA
K/M@QUICKSTART.CLOUDERA
hbase/quickstart.cloudera@QUICKSTART.CLOUDERA
hdfs/quickstart.cloudera@QUICKSTART.CLOUDERA
hive/quickstart.cloudera@QUICKSTART.CLOUDERA
hue/quickstart.cloudera@QUICKSTART.CLOUDERA
impala/quickstart.cloudera@QUICKSTART.CLOUDERA
kadmin/admin@QUICKSTART.CLOUDERA
kadmin/changepw@QUICKSTART.CLOUDERA
kadmin/quickstart.cloudera@QUICKSTART.CLOUDERA
krbtgt/QUICKSTART.CLOUDERA@QUICKSTART.CLOUDERA
mapred/quickstart.cloudera@QUICKSTART.CLOUDERA
oozie/quickstart.cloudera@QUICKSTART.CLOUDERA
root/admin@QUICKSTART.CLOUDERA
solr/quickstart.cloudera@QUICKSTART.CLOUDERA
spark/quickstart.cloudera@QUICKSTART.CLOUDERA
sqoop2/quickstart.cloudera@QUICKSTART.CLOUDERA
yarn/quickstart.cloudera@QUICKSTART.CLOUDERA
zookeeper/quickstart.cloudera@QUICKSTART.CLOUDERA
```

In order to make development easier, we will generate a keytab will all the principals:
```
kadmin.local -q "xst -norandkey -k /tmp/full-keytab.keytab HTTP/quickstart.cloudera@QUICKSTART.CLOUDERA K/M@QUICKSTART.CLOUDERA hbase/quickstart.cloudera@QUICKSTART.CLOUDERA hdfs/quickstart.cloudera@QUICKSTART.CLOUDERA hive/quickstart.cloudera@QUICKSTART.CLOUDERA hue/quickstart.cloudera@QUICKSTART.CLOUDERA impala/quickstart.cloudera@QUICKSTART.CLOUDERA kadmin/admin@QUICKSTART.CLOUDERA kadmin/changepw@QUICKSTART.CLOUDERA kadmin/quickstart.cloudera@QUICKSTART.CLOUDERA krbtgt/QUICKSTART.CLOUDERA@QUICKSTART.CLOUDERA mapred/quickstart.cloudera@QUICKSTART.CLOUDERA oozie/quickstart.cloudera@QUICKSTART.CLOUDERA root/admin@QUICKSTART.CLOUDERA solr/quickstart.cloudera@QUICKSTART.CLOUDERA spark/quickstart.cloudera@QUICKSTART.CLOUDERA sqoop2/quickstart.cloudera@QUICKSTART.CLOUDERA yarn/quickstart.cloudera@QUICKSTART.CLOUDERA zookeeper/quickstart.cloudera@QUICKSTART.CLOUDERA"
```

```
`kinit -k -t /tmp/full-keytab.keytab root/admin@QUICKSTART.CLOUDERA`
```

## Trying HDFS

```
hdfs dfs -ls /
```

## Trying Beeline

```
beeline --hiveconf hive.server2.authentication=kerberos --hiveconf hive.metastore.kerberos.keytab.file=/tmp/full-keytab.keytab --hiveconf hive.metastore.kerberos.principal=hive/quickstart.cloudera@QUICKSTART.CLOUDERA -u "jdbc:hive2://quickstart.cloudera:10000/default;principal=hive/quickstart.cloudera@QUICKSTART.CLOUDERA"
```

## Trying Spark

```
JAR=`ls /usr/lib/spark/lib/spark-examples*.jar | head -1`
MASTER=spark://quickstart.cloudera:7077
DEPLOY_MODE=cluster
KEYTAB=/tmp/full-keytab.keytab
PRINCIPAL=spark/quickstart.cloudera@QUICKSTART.CLOUDERA
CLASS=org.apache.spark.examples.SparkPi
CMD="
spark-submit \
--keytab $KEYTAB \
--principal $PRINCIPAL \
--deploy-mode $DEPLOY_MODE \
--class $CLASS \
--master $MASTER \
$JAR \
100
"
echo $CMD
eval $CMD
```
- issues with kerberized instance

```
su spark
hdfs dfs -mkdir -p /user/spark/share/lib/
hdfs dfs -put /usr/lib/spark/assembly/lib/spark-assembly.jar /user/spark/share/lib/
```

# Issues

## Permission issues

```
Exception in thread "main" org.apache.hadoop.security.AccessControlException: Permission denied: user=spark, access=WRITE, inode="/user/spark":hdfs:supergroup:drwxr-xr-x
```

By default in the cloudera sandbox the spark user doesn't have access to its home folder. From Hue, setting a password for the user `hdfs` and configuring it as super user, will allow to do any modification on the filesystem.

## User not whitelisted

```
main : run as user is spark
main : requested yarn user is spark
Requested user spark is not whitelisted and has id 486,which is below the minimum allowed 1000
```

To fix this go to the YARN configuration and change the value of _Minimum User ID_ (`min.user.id`) to a smaller value. This is not a good practice but will be fine for a development cluster. In case you are going to be using the same user all the time, the field _Allowed System Users_ (`allowed.system.users`) will contain a list of whitelisted users so the user Id won't be checked.




[kerberos-centos-doc]: https://www.centos.org/docs/5/html/5.1/Deployment_Guide/s1-kerberos-server.html
[hortonworks-kerberos-wizard]: http://hortonworks.com/blog/enabling-kerberos-hdp-active-directory-integration/
[tableau-kerberos-hive]: http://kb.tableau.com/articles/knowledgebase/connecting-to-hive-server-2-in-secure-mode
[kerberos-official-site]: http://web.mit.edu/kerberos/www/
