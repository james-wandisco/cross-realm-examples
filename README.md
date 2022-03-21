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
