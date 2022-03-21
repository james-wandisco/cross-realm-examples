# cross-realm-examples
How to set up cross realm trust between two secure hadoop clusters with their own Kerberos realms(KDC’s).  This document highlights the concepts and is applied to two hadoop 3 clusters where configuration files are not managed by a config manager. This will help you understand what you are doing but refer to the vendor's official doc to apply similar concepts to CDP, CDH, HDP.
## Prerequisites
Two hadoop clusters with Kerberos enabled.
Network access from the client host to the KDC for each Realm
The hostnames must be resolvable.
Both clusters must be running JDK 1.7 or higher. JDK 1.6 has some known issues.
 
## Specification.
Hadoop version:  hadoop-3.1.1
OS version: CentOS Linux release 7.5.1804 (Core)
Kerberos: 1.15.1-51.el7_9.x86_64
Java: openjdk version "1.8.0_322"
Note: No Cloudera or Hortonworks, this is pure hadoop to test and become familiar with the core concepts applied.

## Example 1: Make Cluster 2 'trust' Cluster 1. (one way.)
For a client of Cluster 1 to access a service in realm Cluster 2, do the following 4 steps.

- Create a shared krbtgt(ticket granting ticket) principal on both clusters.
- Add auth_to_local entries on Cluster 2 for Cluster 1 principals.
- Edit the **[realms]** section krb5.conf on Cluster 1 to provide details of the Cluster 2 realm.
- Edit the **[domain_realm]** section of krb5.conf on Cluster 1 to provide translation from a domain name or hostname to a Kerberos realm name.

**Realms used.**
- Cluster 1 realm is CLOUDREALM.HADOOP
- Cluster 2 realm is TARGETREALM.HADOOP

### Step 1: Add krbtgt principal.

**Brief explanation**
Both realms must share a key for a principal named..    
krbtgt/TARGETREALM.HADOOP@CLOUDREALM.HADOOP
(Both keys must have the same key version number associated with them.)

**What to do.**
Add this principal to both clusters.

Cluster 1 cli
```
kadmin.local
addprinc krbtgt/TARGETREALM.HADOOP@CLOUDREALM.HADOOP
```


Cluster 2 cli
```
kadmin.local
addprinc krbtgt/TARGETREALM.HADOOP@CLOUDREALM.HADOOP
```

**How to verify.**
Verify entries have matching kvno and encryption types using
```
kadmin.local
getprinc krbtgt/TARGETREALM.HADOOP@CLOUDREALM.HADOOP
```

### Step 2: auth_to_local additions.

**Brief explanation**
Cluster 2 needs to be able to map the Kerberos principal to the user ID short name. This can be done with the 'hadoop.security.auth_to_local' mapping rules.

The syntax of a mapping rule is:
```
RULE:[<principal translation>](<acceptance filter>)<short name substitution>
```

Add in one or more rules like below, depending on which users you need.

**What to do.**
Edit core-site.xml on Cluster 2 and add the following in the 'hadoop.security.auth_to_local' values.

```
RULE:[2:$1/$2@$0](hdfs/.*@CLOUDREALM.HADOOP)s/.*/hdfs/
```

#### How to verify.
You can use the following from the CLI on Cluster 2 to verify what this rule will map too, in this example we want it to return simply 'hdfs'
```
hadoop org.apache.hadoop.security.HadoopKerberosName hdfs/host@CLOUDREALM.HADOOP
```

*Note. Remember this is a non managed hadoop cluster, if you are using a vendor distribution in a differing environment you will need to push the config to the cluster.*

See **References** if you want to understand auth_to_local syntax further.


### Step 3: Edit [realms]

**Brief explanation.**
Each tag in the [realms] section of the file is the name of a Kerberos realm.
The value of the tag is a subsection with the properties of that particular realm.
We add the Cluster 2 realm details under [realms] of the Cluster 1 krb5.conf file.  i.e details of TARGETREALM.HADOOP. You will then have two realms defined here.

**What to do.**
Edit the /etc/krb5.conf (all nodes) on the Cluster 1 add the details of the Cluster 2 realm to the [realms] section

```
vi  /etc/krb5.conf
```

```
[realms]
..
.. <Your existing realm will be defined here.>
..

TARGETREALM.HADOOP = {
kdc = azvmcloud2
admin_server = azvmcloud2
master_kdc = azvmcloud2
}
```

### Step 4: Edit [domain_realm]

**Brief explanation.**
The [domain_realm] section of the Kerberos configuration file maps either a computer hostname or network domain name to a Kerberos Realm.  The value in the configuration file can be a hostname or domain name. This example uses hostnames.

See **References** for a full explanation of all krb5.conf properties.

**What to do.**
Edit the /etc/krb5.conf (all nodes) on the Cluster 1. Add an entry that will point the hostname to the correct realm.
```
vi  /etc/krb5.conf
```
```
[domain_realm]
azvmcloud1 = CLOUDREALM.HADOOP
azvmcloud2 = TARGETREALM.HADOOP
```

**How to verify.**
You should now be done. Test with similar commands as below to see if you can access the hdfs filesystem on Cluster 2.
```
hdfs dfs -ls  hdfs://azvmcloud2:9000/
hdfs dfs -mkdir  hdfs://azvmcloud2:9000/tmp/test1
```

## Example 2: Trust in both directions.

For a client of Cluster 1 to access a service in realm Cluster 2, and vice versa, do the following 4 steps.

- Create **two** shared krbtgt principals on both clusters.
- Add auth_to_local entries on Cluster 1 and Cluster 2.
- Edit the [realms] section krb5.conf on Cluster 1 to provide details of the Cluster 2 realm.
- Edit the [realms] section krb5.conf on Cluster 2 to provide details of the Cluster 1 realm.
- Edit the [domain_realm] section of krb5.conf on both clusters to provide translation of the hostname to a Kerberos realm name.

**Realms used.**
- Cluster 1 realm is CLOUDREALM.HADOOP
- Cluster 2 realm is TARGETREALM.HADOOP

### Step 1: Add principals.

**Brief explanation.**
Same as above but you need another principal for trust the other way.
Both realms must share a key for two principals named..
krbtgt/TARGETREALM.HADOOP@CLOUDREALM.HADOOP
krbtgt/CLOUDREALM.HADOOP@TARGETREALM.HADOOP

**What to do.**
Add the following to add 2 principals on both clusters. Don’t add the same one again if you already have it!

On Cluster 1
```
kadmin.local
addprinc krbtgt/CLOUDREALM.HADOOP@TARGETREALM.HADOOP
addprinc krbtgt/TARGETREALM.HADOOP@CLOUDREALM.HADOOP
```

On Cluster 2
```
kadmin.local
addprinc krbtgt/CLOUDREALM.HADOOP@TARGETREALM.HADOOP
addprinc krbtgt/TARGETREALM.HADOOP@CLOUDREALM.HADOOP
```

**How to verify.**
Verify entries have matching kvno and encryption types using
```
getprinc krbtgt/CLOUDREALM.HADOOP@TARGETREALM.HADOOP
getprinc krbtgt/TARGETREALM.HADOOP@CLOUDREALM.HADOOP
```

### Step 2: auth_to_local additions.

**Brief explanation**
Same as above but on both clusters.

**What to do.**
Edit core-site.xml on Cluster 2 and add the following in the 'hadoop.security.auth_to_local' values to map principals coming in from CLOUDREALM.HADOOP to local usernames.
```
RULE:[2:$1/$2@$0](hdfs/.*@CLOUDREALM.HADOOP)s/.*/hdfs/
```

Edit core-site.xml on Cluster 1 and add the following in the 'hadoop.security.auth_to_local' values to map principals coming in from TARGETREALM.HADOOP to local usernames.
```
RULE:[2:$1/$2@$0](hdfs/.*@TARGETREALM.HADOOP)s/.*/hdfs/
```

**How to verify.**
You can use the following from the CLI on to verify what this rule will map too, in this example we want it to return simply 'hdfs'

```
hadoop org.apache.hadoop.security.HadoopKerberosName hdfs/host@CLOUDREALM.HADOOP
hadoop org.apache.hadoop.security.HadoopKerberosName hdfs/host@TARGETREALM.HADOOP
```


### Step 3: Edit [realms] on Cluster 1

**What to do.**
Edit the /etc/krb5.conf (all nodes) on the Cluster 1 add the details of the Cluster 2 realm to the [realms] section.

```
vi  /etc/krb5.conf
```

```
..
[realms]
..
.. Your existing realm will be defined here.
..
TARGETREALM.HADOOP = {
kdc = azvmcloud2
admin_server = azvmcloud2
master_kdc = azvmcloud2
}
```

### Step 4: Edit [realms] on Cluster 2

**What to do**
Edit the /etc/krb5.conf (all nodes) on the Cluster 2 add the details of the Cluster 1 realm to the [realms] section.

```
vi  /etc/krb5.conf
```

```
[realms]
..
.. Your existing realm will be defined here.
..
CLOUDREALM.HADOOP = {
kdc = azvmcloud1
admin_server = azvmcloud1
master_kdc = azvmcloud1
}
```

### Step 5: Edit [domain_realm]

**What to do.**
Edit the /etc/krb5.conf (all nodes) on both clusters. Add an entry that will point your hostname to the correct realm.

```
vi /etc/krb5.conf
```

```
[domain_realm]
azvmcloud1 = CLOUDREALM.HADOOP
azvmcloud2 = TARGETREALM.HADOOP
```

**How to verify.**
You should now be done. Test with similar commands as below to see if you can access the hdfs filesystem on Cluster 2.
```
hdfs dfs -ls  hdfs://azvmcloud2:9000/
hdfs dfs -mkdir  hdfs://azvmcloud2:9000/tmp/test1
```



## References

https://web.mit.edu/kerberos/krb5-1.5/krb5-1.5.4/doc/krb5-admin/Cross_002drealm-Authentication.html

https://support.sas.com/resources/papers/proceedings17/SAS0623-2017.pdf

https://docs.fedoraproject.org/en-US/Fedora/17/html/Security_Guide/sect-Security_Guide-Kerberos-Setting_Up_Cross_Realm_Authentication.html

https://web.mit.edu/kerberos/krb5-1.12/doc/admin/conf_files/krb5_conf.html#realms

https://docs.cloudera.com/documentation/enterprise/6/6.2/topics/cdh_sg_kerbprin_to_sn.html 

