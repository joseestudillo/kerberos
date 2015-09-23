# Kerberos

# Concepts

- _ticket_: proof of identity encrypted with a secret key for the particular service requested.

- _realm_: defines what Kerberos manages in terms of who can access what.

- _Principal_: is a unique identity to which Kerberos can assign tickets.

- _keytab_: A keytab is a file containing pairs of Kerberos principals and encrypted keys (which are derived from the Kerberos password). You can use a keytab file to authenticate to various remote systems using Kerberos without entering a password.

- _KDC_: Key Distribution center. It consists in an _Authentication Server_ and a _Ticket Granting Server_. 

- _Service or host machine_: the machine to access to. 

For further information go to [kerberos official site][kerberos-official-site]



# How does it work?

- How to get a ticket: `kinit USERNAME`

# How to's

## Login using keytab

in any linux distribution:

`kinit <PRINCIPAL> -k -t <KEYTAB_FILE_PATH>` 

as always mac being... different...

`kinit --use-keytab --keytab=<KEYTAB_FILE_PATH> <PRINCIPAL>`

## Get all principals

type `kadmin.local` once logged in `listprincs`

## Create a keytab file

type `kadmin.local` once logged in `xst -norandkey -k <KEYTAB_FILE_PATH> PRINCIPALS_CSV`. This will generate a file containing all the given principals.

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
 EXAMPLE.COM = {
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
/usr/kerberos/sbin/kdb5_util create -s
```

the database will be created by default under `/var/kerberos/krb5kdc/`
 
- create `/var/kerberos/krb5kdc/kadm5.acl`:

```bash
*/admin@JOSEESTUDILLO.COM	*
```

This file is used by `kadmin` to determine which principals have administrative access to the Kerberos database and their level of access.


- Create a principal

```bash
/usr/kerberos/sbin/kadmin.local -q "addprinc <USERNAME>/admin"
```

to for example create a principal for myself:

```bash
/usr/kerberos/sbin/kadmin.local -q "addprinc jose/admin"
```
so the generated principal would be `jose/admin@JOSEESTUDILLO.COM`


- start services:

```bash
/sbin/service krb5kdc start
/sbin/service kadmin start
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

/usr/kerberos/sbin/kadmin.local -q "addprinc root/admin"
  
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
- _Admin Password: the password you set during the principal creation


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


## Fixing access problems after restarting

set the principals to use the proper host

grep -l -r "\/_HOST" /etc/hadoop/ | xargs sed -i -e "s&/_HOST&/sandbox.hortonworks.com&g"


# Accesing to services


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

even having a valid kerberos ticket locally. [TODO FIX]



## HDFS Hadoop API

This case will be similar the Hive JDBC access so given a Hadoop configuration stored in `conf` something like the code above will do the job:

```java
conf.set("hadoop.security.authentication", "kerberos");
UserGroupInformation.setConfiguration(conf);
UserGroupInformation.loginUserFromKeytab(principal, keytabPath);
``` 

There is a complete example of this in the hadoop-quick-start project in this repository.


[kerberos-centos-doc]: https://www.centos.org/docs/5/html/5.1/Deployment_Guide/s1-kerberos-server.html
[hortonworks-kerberos-wizard]: http://hortonworks.com/blog/enabling-kerberos-hdp-active-directory-integration/
[tableau-kerberos-hive]: http://kb.tableau.com/articles/knowledgebase/connecting-to-hive-server-2-in-secure-mode
[kerberos-official-site]: http://web.mit.edu/kerberos/www/
