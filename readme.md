# Nextcloud and NFS as external storage for openBIS instances
This is a general guide to setup nextcloud as external storage device and connect it to an openBIS instance to allow uploading big files.
The paper discusses the implementation of a data synchronization system for large raw mass spectrometry files in the SMART-CARE research project. To address the challenge of efficiently uploading these large files to the openBIS-based repository, the authors introduced the Nextcloud data cloud system. This system automates data import into the repository, ensuring data provenance and reducing the burden on lab staff. The approach is generic and can be adopted by other projects using openBIS for data management.
The publication can be found here: https://pubmed.ncbi.nlm.nih.gov/35612108/


## Nextcloud
### Installation
Install Nexcloud according to these instructions: https://docs.nextcloud.com/server/latest/admin_manual/installation/example_centos.html

Exceptions:
* Do not install recommended apps during first login (remove check mark)

In addition, install the following apps:
* External storage support 
* LDAP user and group backend 

For reverse proxy, make sure to add the proxy domain to `/var/www/html/nextcloud/config/config.php`, for example:

```php
 'trusted_domains' =>
  array (
    0 => 'some.page',
    1 => 'other.page',
  ),
```

### LDAP/AD integration
LDAP configuration is described here: https://docs.nextcloud.com/server/latest/admin_manual/configuration_user/user_auth_ldap.html
It can be also configured on the command line: https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/occ_command.html?highlight=ldap#ldap-commands-label


For the nextcloud desktop client it is necessary to change the way nextcloud uses for the internal user id. Switch to expert mode and enter `sAMAccountname` as UUID attribute for users.

With `sudo -u apache php occ ldap:show-config` issued in folder `/var/www/html/nextcloud` you can show the configuration. It should look like the following:


| Configuration                 | s01                                                                                                                                                          |
|-------------------------------|------------------------------------------------------|
| hasMemberOfFilterSupport      | 0                                                                                                                                                            |
| homeFolderNamingRule          |                                                                                                                                                              |
| lastJpegPhotoLookup           | 0                                                                                                                                                            |
| ldapAgentName                 | yourconfiguration                                                                                                                                            |
| ldapAgentPassword             | ***                                                                                                                                                          |
| ldapAttributesForGroupSearch  |                                                                                                                                                              |
| ldapAttributesForUserSearch   |                                                                                                                                                              |
| ldapBackupHost                |                                                                                                                                                              |
| ldapBackupPort                |                                                                                                                                                              |
| ldapBase                      |  yourconfiguration                                                                                                                                           |
| ldapBaseGroups                |  yourconfiguration                                                                                                                                           |
| ldapBaseUsers                 |  yourconfiguration                                                                                                                                           |
| ldapCacheTTL                  | 600                                                                                                                                                          |
| ldapConfigurationActive       | 1                                                                                                                                                            |
| ldapDefaultPPolicyDN          |                                                                                                                                                              |
| ldapDynamicGroupMemberURL     |                                                                                                                                                              |
| ldapEmailAttribute            | mail                                                                                                                                                         |
| ldapExperiencedAdmin          | 1                                                                                                                                                            |
| ldapExpertUUIDGroupAttr       |                                                                                                                                                              |
| ldapExpertUUIDUserAttr        | sAMAccountname                                                                                                                                               |
| ldapExpertUsernameAttr        |                                                                                                                                                              |
| ldapExtStorageHomeAttribute   |                                                                                                                                                              |
| ldapGidNumber                 | gidNumber                                                                                                                                                    |
| ldapGroupDisplayName          | cn                                                                                                                                                           |
| ldapGroupFilter               |  (&(|(objectclass=group)))                                                                                                                                   |
| ldapGroupFilterGroups         |                                                                                                                                                              |
| ldapGroupFilterMode           | 0                                                                                                                                                            |
| ldapGroupFilterObjectclass    | group                                                                                                                                                        |
| ldapGroupMemberAssocAttr      | member                                                                                                                                                       |
| ldapHost                      |  yourconfiguration                                                                                                                                           |
| ldapIgnoreNamingRules         |                                                                                                                                                              |
| ldapLoginFilter               |  yourconfiguration                                                                                                                                           |
| ldapLoginFilterAttributes     |                                                                                                                                                              |
| ldapLoginFilterEmail          | 0                                                                                                                                                            |
| ldapLoginFilterMode           | 0                                                                                                                                                            |
| ldapLoginFilterUsername       | 1                                                                                                                                                            |
| ldapMatchingRuleInChainState  | unknown                                                                                                                                                      |
| ldapNestedGroups              | 0                                                                                                                                                            |
| ldapOverrideMainServer        |                                                                                                                                                              |
| ldapPagingSize                | 500                                                                                                                                                          |
| ldapPort                      | 636                                                                                                                                                          |
| ldapQuotaAttribute            |                                                                                                                                                              |
| ldapQuotaDefault              |                                                                                                                                                              |
| ldapTLS                       | 0                                                                                                                                                            |
| ldapUserAvatarRule            | default                                                                                                                                                      |
| ldapUserDisplayName           | displayname                                                                                                                                                  |
| ldapUserDisplayName2          |                                                                                                                                                              |
| ldapUserFilter                |  yourconfiguration                                                                                                                                           |
| ldapUserFilterGroups          |                                                                                                                                                              |
| ldapUserFilterMode            | 0                                                                                                                                                            |
| ldapUserFilterObjectclass     |                                                                                                                                                              |
| ldapUuidGroupAttribute        | auto                                                                                                                                                         |
| ldapUuidUserAttribute         | auto                                                                                                                                                         |
| turnOffCertCheck              | 1                                                                                                                                                            |
| turnOnPasswordChange          | 0                                                                                                                                                            |
| useMemberOfToDetectMembership | 1                                                                                                                                                            |


## NFS server

*Important:* A seperate VM for NFS-hosting is recommended.

### Add data directory on openBIS VM 
For data exchange, a separate folder should be created, for example `/data`:
```bash
mkdir /data
mkdir /data/drop
mkdir /data/store
chmod a+w /data/drop
chmod a+w /data/store
chown -R apache:apache /data
semanage fcontext -a -t httpd_sys_rw_content_t '/data(/.*)?'
restorecon -R '/data'
```

Now, the folder `/data/drop` can be made available as `drop` in Nextcloud via the external storage app using a web browser. It is important to declare it as external storage, since we also want to access it via NFS.


### Installation
See https://www.tecmint.com/install-nfs-server-on-centos-8/

```bash
dnf install -y nfs-utils
systemctl enable --now nfs-server.service
```

Create file `/etc/exports` to make folders available:

```
/data/drop  yourIP(rw,sync)
/data/store yourIP(rw,sync)
```

Activate shares via `exportfs -arv`.

Configure firewall:

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload
```


## NFS client
### Install
```dnf install -y nfs-utils nfs4-acl-tools```

Show available shares via `showmount -e yourVM`.

Create mount points:

```
mkdir /drop
mkdir /store
chown openbis:openbis /drop
chown openbis:openbis /store
```

To test mounting use the following (from showmount):
```
 mount -t nfs yourVM:/data/drop /drop
 mount -t nfs yourVM:/data/store /store
```

Now you should be able to read/write the remote file system.

To automatically mount the shares at boot time add the following to `/etc/fstab`. *Warning:* If you do something wrong in this file, the system might not boot anymore.

```
yourVM:/data/drop       /drop   nfs defaults 0 0
yourVM:/data/store      /store  nfs defaults 0 0
```

After sucessful installation, users can now 






