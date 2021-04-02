# Springboot / LDAP integration

The following tutorial describes LDAP server startup with an initial setup of user, group and password. The tutorial describes also basic LDAP search commands.

## Run LDAP Server

Run LDAP server with the administration user ```admin``` (wich is the default admin user) and its password ```admin```:
```
mkdir ldif

sudo docker run -d --name openldap -p 389:389 -v $(pwd)/ldif:/ldif --env LDAP_DOMAIN="MYCOMPANY.NET" --env LDAP_ADMIN_PASSWORD="admin" --env LDAP_BASE_DN="DC=MYCOMPANY,DC=NET" osixia/openldap:1.5.0
```

Check that the server is up and running by executing a ```ldapsearch``` on a given LDAP base:
```
sudo docker exec openldap ldapsearch -w "admin" -x -h localhost -p 389 -D 'CN=admin,DC=MYCOMPANY,DC=NET' -b 'DC=MYCOMPANY,DC=NET'
```

## LDAP structure

Create the Organization Units ```APPLICATIONS``` and ```INSURANCE```:
```
cat << EOF>> ldif/ou1.ldif
dn: ou=APPLICATIONS,dc=MYCOMPANY,dc=NET
objectClass: organizationalUnit
ou: posix
EOF

sudo docker exec openldap ldapadd -w "admin" -x -h localhost -p 389 -D "CN=admin,DC=MYCOMPANY,DC=NET" -f /ldif/ou1.ldif

cat << EOF>> ldif/ou2.ldif
dn: OU=INSURANCE,ou=APPLICATIONS,dc=MYCOMPANY,dc=NET
objectClass: organizationalUnit
ou: posix
EOF

sudo docker exec openldap ldapadd -w "admin" -x -h localhost -p 389 -D "CN=admin,DC=MYCOMPANY,DC=NET" -f /ldif/ou2.ldif
```

Create the group ```MY_GROUPE_1```:
```
cat << EOF>> ldif/group1.ldif
dn: cn=MY_GROUPE_1,OU=INSURANCE,OU=APPLICATIONS,dc=MYCOMPANY,dc=NET
changetype: add
objectClass: top
objectClass: posixGroup
gidNumber: 22222
EOF

sudo docker exec openldap ldapadd -w "admin" -x -h localhost -p 389 -D "CN=admin,DC=MYCOMPANY,DC=NET" -f /ldif/group1.ldif
```

Create the user ```user1```:
```
cat << EOF>> ldif/user1.ldif
dn: uid=user1,OU=INSURANCE,OU=APPLICATIONS,dc=MYCOMPANY,dc=NET
changetype: add
uid: user1
cn: user1
sn: 3
objectClass: top
objectClass: inetOrgPerson
EOF

sudo docker exec openldap ldapadd -w "admin" -x -h localhost -p 389 -D "CN=admin,DC=MYCOMPANY,DC=NET" -f /ldif/user1.ldif
```

Assign user ```user1``` to group ```groupe MY_GROUPE_1```:
```
cat << EOF>> ldif/moduser1.ldif
dn: cn=MY_GROUPE_1,OU=INSURANCE,OU=APPLICATIONS,dc=MYCOMPANY,dc=NET
changetype: modify
add: memberuid
memberuid: user1
EOF

sudo docker exec openldap ldapadd -w "admin" -x -h localhost -p 389 -D "CN=admin,DC=MYCOMPANY,DC=NET" -f /ldif/moduser1.ldif
```

Set password ```mypassword1``` to user ```user1``` :
```
sudo docker exec openldap ldappasswd -s mypassword1 -w "admin" -w "admin" -x -h localhost -p 389 -D "CN=admin,DC=MYCOMPANY,DC=NET" -x "uid=user1,ou=INSURANCE,ou=APPLICATIONS,dc=MYCOMPANY,dc=NET"
```

LDAP search on branch ```OU=INSURANCE,OU=APPLICATIONS,dc=MYCOMPANY,dc=NET```:
```
sudo docker exec openldap ldapsearch -w "admin" -x -h localhost -p 389 -D 'CN=admin,DC=MYCOMPANY,DC=NET' -b 'ou=INSURANCE,ou=APPLICATIONS,DC=MYCOMPANY,DC=NET'
```

Result:
```
# extended LDIF
#
# LDAPv3
# base <ou=INSURANCE,ou=APPLICATIONS,DC=MYCOMPANY,DC=NET> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#
# INSURANCE, APPLICATIONS, MYCOMPANY.NET
dn: ou=INSURANCE,ou=APPLICATIONS,dc=MYCOMPANY,dc=NET
objectClass: organizationalUnit
ou: posix
ou: INSURANCE
# user1, INSURANCE, APPLICATIONS, MYCOMPANY.NET
dn: uid=user1,ou=INSURANCE,ou=APPLICATIONS,dc=MYCOMPANY,dc=NET
uid: user1
cn: user1
sn: 3
objectClass: top
objectClass: inetOrgPerson
userPassword:: e1NTSEF9M3RwK2tpOHJZZlNOK3BpMDJNeitLTE96VG1kd3pKWmU=
# MY_GROUPE_1, INSURANCE, APPLICATIONS, MYCOMPANY.NET
dn: cn=MY_GROUPE_1,ou=INSURANCE,ou=APPLICATIONS,dc=MYCOMPANY,dc=NET
objectClass: top
objectClass: posixGroup
gidNumber: 22222
cn: MY_GROUPE_1
memberUid: user1
# search result
search: 2
result: 0 Success
# numResponses: 4
# numEntries: 3
```

Note the attributes ```userPassword``` of user ```user1``` and ```memberUid``` of group  ```MY_GROUPE_1```

## Stop LDAP server

Stop the LDAP server container:
```
sudo docker stop openldap; sudo docker rm openldap
```