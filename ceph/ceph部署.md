# ceph部署

## 启动bootstrap node

Run the ceph bootstrap command:

```shell
cephadm bootstrap --mon-ip *<mon-ip>*

mon-ip 就是第一个monitor daemon的IP地址     
```

This command will:

* Create a monitor and manager daemon for the new cluster on the local host.

* Generate a new SSH key for the Ceph cluster and add it to the root user’s `/root/.ssh/authorized_keys` file.

* Write a copy of the public key to `/etc/ceph/ceph.pub`.

* Write a minimal configuration file to `/etc/ceph/ceph.conf`. This file is needed to communicate with the new cluster.

* Write a copy of the client.admin administrative (privileged!) secret key to `/etc/ceph/ceph.client.admin.keyring`.

* Add the _admin label to the bootstrap host. By default, any host with this label will (also) get a copy of /etc/ceph/ceph.conf and /etc/ceph/ceph.client.admin.keyring.

* 启动以下容器：
  * ceph-mgr ceph管理程序
  * ceph-monitor ceph监视器
  * ceph-crash 崩溃数据收集模块
  * prometheus prometheus监控组件
  * grafana 监控数据展示dashboard
  * alertmanager prometheus告警组件
  * node_exporter prometheus节点数据收集组件

## 将主机添加到集群中  

在新节点的root用户的authorized_keys文件中安装集群的公共SSH密钥：

```sh
cd /etc/ceph/
ssh-copy-id -f -i ceph.pub root@domain_name ip
```

添加节点

```sh
cephadm shell -- ceph orch host add ip/domain_name
```

查看节点

```sh
cephadm shell -- ceph orch host ls
```

## 部署mon,mgr,osd

给需要部署mon进程的节点打上标签

```sh
cephadm shell -- ceph orch host label add domain_name ip
```

根据标签部署monitor

```sh
cephadm shell -- ceph orch apply ip/domain_name label:lable_name
```

列出节点上的所有可用设备

```sh
cephadm shell -- ceph orch device ls   
如果满足以下所有条件，则认为存储设备可用：

设备必须没有分区。
设备不得具有任何LVM状态。
设备不能被mounted。
该设备不得包含文件系统。
该设备不得包含Ceph BlueStore OSD。
设备必须大于5 GB。

sgdisk --zap-all 磁盘名 ，使其满足条件。   
```

创建osd

方法1：从特定主机上的特定设备创建OSD

```sh
cephadm shell -- ceph orch daemon add osd ceph-mon1:/dev/sdb
cephadm shell -- ceph orch daemon add osd ceph-mon1:/dev/sdc
cephadm shell -- ceph orch daemon add osd ceph-mon2:/dev/sdb
cephadm shell -- ceph orch daemon add osd ceph-mon2:/dev/sdc
cephadm shell -- ceph orch daemon add osd ceph-mon3:/dev/sdb
cephadm shell -- ceph orch daemon add osd ceph-mon3:/dev/sdc
cephadm shell -- ceph orch daemon add osd ceph-osd4:/dev/sdb
cephadm shell -- ceph orch daemon add osd ceph-osd4:/dev/sdc
```

方法2：添加任何可用和未使用的存储设备

```sh
ceph orch apply osd --all-available-devices
```

## ceph命令

General usage
===

```note
usage: ceph [-h] [-c CEPHCONF] [-i INPUT_FILE] [-o OUTPUT_FILE]
            [--setuser SETUSER] [--setgroup SETGROUP] [--id CLIENT_ID]
            [--name CLIENT_NAME] [--cluster CLUSTER]
            [--admin-daemon ADMIN_SOCKET] [-s] [-w] [--watch-debug]
            [--watch-info] [--watch-sec] [--watch-warn] [--watch-error]
            [-W WATCH_CHANNEL] [--version] [--verbose] [--concise]
            [-f {json,json-pretty,xml,xml-pretty,plain,yaml}]
            [--connect-timeout CLUSTER_TIMEOUT] [--block] [--period PERIOD]

Ceph administration tool
  
optional arguments

| optional arguments | reference |
| --- | --- |
|  -h, --help                    |request mon help|
|  -c CEPHCONF, --conf CEPHCONF  |ceph configuration file|
|  -i INPUT_FILE, --in-file INPUT_FILE   |input file, or "-" for stdin|
|  -o OUTPUT_FILE, --out-file OUTPUT_FILE    |output file, or "-" for stdout|
|  --setuser SETUSER     |set user file permission|
|  --setgroup SETGROUP   |set group file permission|
|  --id CLIENT_ID, --user CLIENT_ID |client id for authentication|
|  --name CLIENT_NAME, -n CLIENT_NAME |client name for authentication|
|  --cluster CLUSTER     |cluster name|
|  --admin-daemon ADMIN_SOCKET|submit admin-socket commands ("help" for help)|
|  -s, --status          |show cluster status|
|  -w, --watch           |watch live cluster changes|
|  --watch-debug         |watch debug events|
|  --watch-info          |watch info events|
|  --watch-sec           |watch security events|
|  --watch-warn          |watch warn events|
|  --watch-error         |watch error events|
|  -W WATCH_CHANNEL, --watch-channel WATCH_CHANNEL|watch live cluster changes on a specific channel(e.g., cluster, audit, cephadm, or '*' for all)|
|  --version, -v         |display version|
|  --verbose             |make verbose|
|  --concise             |make less verbose|
|  -f {json,json-pretty,xml,xml-pretty,plain,yaml}, --format {json,json-pretty,xml,xml-pretty,plain,yaml} | |
|  --connect-timeout CLUSTER_TIMEOUT | set a timeout for connecting to the cluster |
|  --block               | block until completion (scrub and deep-scrub only) |
|  --period PERIOD, -p PERIOD | polling period, default 1.0 second (for pollingcommands only) |

Local commands
 ===============

ping <mon.id>           Send simple presence/life test to a mon , <mon.id> may be 'mon.*' for all mons
daemon {type.id|path} <cmd> Same as --admin-daemon, but auto-find admin socket
daemonperf {type.id | path} [stat-pats] [priority] [<interval>] [<count>]
daemonperf {type.id | path} list|ls [stat-pats] [priority]
                        Get selected perf stats from daemon/admin socket
                        Optional shell-glob comma-delim match string stat-pats
                        Optional selection priority (can abbreviate name):
                         critical, interesting, useful, noninteresting, debug
                        List shows a table of all available stats
                        Run <count> times (default forever),
                         once per <interval> seconds (default 1)

Monitor commands
 =================

alerts send                             (re)send alerts immediately
auth add <entity> [<caps>...]           add auth info for <entity> from input
                                         file, or random key if no input is
                                         given, and/or any caps specified in
                                         the command
auth caps <entity> <caps>...            update caps for <name> from caps
                                         specified in the command
auth export [<entity>]                  write keyring for requested entity, or
                                         master keyring if none given
auth get <entity>                       write keyring file with requested key
auth get-key <entity>                   display requested key
auth get-or-create <entity> [<caps>...] add auth info for <entity> from input
                                         file, or random key if no input given,
                                         and/or any caps specified in the
                                         command
auth get-or-create-key <entity>         get, or add, key for <name> from system/
 [<caps>...]                             caps pairs specified in the command.  
                                         If key already exists, any given caps
                                         must match the existing caps for that
                                         key.
auth import                             auth import: read keyring file from -i
                                         <file>
auth ls                                 list authentication state
auth print-key <entity>                 display requested key
auth print_key <entity>                 display requested key
auth rm <entity>                        remove all caps for <name>
balancer dump <plan>                    Show an optimization plan
balancer eval [<option>]                Evaluate data distribution for the
                                         current cluster or specific pool or
                                         specific plan
balancer eval-verbose [<option>]        Evaluate data distribution for the
                                         current cluster or specific pool or
                                         specific plan (verbosely)
balancer execute <plan>                 Execute an optimization plan
balancer ls                             List all plans
balancer mode none|crush-compat|upmap   Set balancer mode
balancer off                            Disable automatic balancing
balancer on                             Enable automatic balancing
balancer optimize <plan> [<pools>...]   Run optimizer to create a new plan
balancer pool add <pools>...            Enable automatic balancing for specific
                                         pools
balancer pool ls                        List automatic balancing pools. Note
                                         that empty list means all existing
                                         pools will be automatic balancing
                                         targets, which is the default
                                         behaviour of balancer.
balancer pool rm <pools>...             Disable automatic balancing for
                                         specific pools
balancer reset                          Discard all optimization plans
balancer rm <plan>                      Discard an optimization plan
balancer show <plan>                    Show details of an optimization plan
balancer status                         Show balancer status
cephadm check-host <host> [<addr>]      Check whether we can access and manage
                                         a remote host
cephadm clear-exporter-config           Clear the SSL configuration used by
                                         cephadm exporter daemons
cephadm clear-key                       Clear cluster SSH key
cephadm clear-ssh-config                Clear the ssh_config file
cephadm config-check disable <check_Disable a specific configuration check
 name>
cephadm config-check enable <check_     Enable a specific configuration check
 name>
cephadm config-check ls [--format       List the available configuration checks
 {plain|json|json-pretty|yaml}]          and their current state
cephadm config-check status             Show whether the configuration checker
                                         feature is enabled/disabled
cephadm generate-exporter-config        Generate default SSL crt/key and token
                                         for cephadm exporter daemons
cephadm generate-key                    Generate a cluster SSH key (if not
                                         present)
cephadm get-exporter-config             Show the current cephadm-exporter
                                         configuraion (JSON)'
cephadm get-extra-ceph-conf             Get extra ceph conf that is appended
cephadm get-pub-key                     Show SSH public key for connecting to
                                         cluster hosts
cephadm get-ssh-config                  Returns the ssh config as used by
                                         cephadm
cephadm get-user                        Show user for SSHing to cluster hosts
cephadm osd activate <host>...          Start OSD containers for existing OSDs
cephadm prepare-host <host> [<addr>]    Prepare a remote host for use with
                                         cephadm
cephadm registry-login [<url>]          Set custom registry login info by
 [<username>] [<password>]               providing url, username and password
                                         or json file with login info (-i
                                         <file>)
cephadm set-exporter-config             Set custom cephadm-exporter
                                         configuration from a json file (-i
                                         <file>). JSON must contain crt, key,
                                         token and port
cephadm set-extra-ceph-conf             Text that is appended to all daemon's
                                         ceph.conf. Mainly a workaround, till `
                                         config generate-minimal-conf`
                                         generates a complete ceph.conf.  
                                         Warning: this is a dangerous operation.
cephadm set-priv-key                    Set cluster SSH private key (use -i
                                         <private_key>)
cephadm set-pub-key                     Set cluster SSH public key (use -i
                                         <public_key>)
cephadm set-ssh-config                  Set the ssh_config file (use -i <ssh_
                                         config>)
cephadm set-user <user>                 Set user for SSHing to cluster hosts,
                                         passwordless sudo will be needed for
                                         non-root users
config assimilate-conf                  Assimilate options from a conf, and
                                         return a new, minimal conf file
config dump                             Show all configuration option(s)
config generate-minimal-conf            Generate a minimal ceph.conf file
config get <who> [<key>]                Show configuration option(s) for an
                                         entity
config help <key>                       Describe a configuration option
config log [<num:int>]                  Show recent history of config changes
config ls                               List available configuration options
config reset <num:int>                  Revert configuration to a historical
                                         version specified by <num>
config rm <who> <name>                  Clear a configuration option for one or
                                         more entities
config set <who> <name> <value> [--     Set a configuration option for one or
 force]                                  more entities
config show <who> [<key>]               Show running configuration
config show-with-defaults <who>         Show running configuration (including
                                         compiled-in defaults)
config-key dump [<key>]                 dump keys and values (with optional
                                         prefix)
config-key exists <key>                 check for <key>'s existence
config-key get <key>                    get <key>
config-key ls                           list keys
config-key rm <key>                     rm <key>
config-key set <key> [<val>]            set <key> to value <val>
crash archive <id>                      Acknowledge a crash and silence health
                                         warning(s)
crash archive-all                       Acknowledge all new crashes and silence
                                         health warning(s)
crash info <id>                         show crash dump metadata
crash json_report <hours>               Crashes in the last <hours> hours
crash ls                                Show new and archived crash dumps
crash ls-new                            Show new crash dumps
crash post                              Add a crash dump (use -i <jsonfile>)
crash prune <keep>                      Remove crashes older than <keep> days
crash rm <id>                           Remove a saved crash <id>
crash stat                              Summarize recorded crashes
dashboard ac-role-add-scope-perms       Add the scope permissions for a role
 <rolename> <scopename> [<permissions>.
 ..]
dashboard ac-role-create [<rolename>]   Create a new access control role
 [<description>]
dashboard ac-role-del-scope-perms       Delete the scope permissions for a role
 <rolename> [<scopename>]
dashboard ac-role-delete [<rolename>]   Delete an access control role
dashboard ac-role-show [<rolename>]     Show role info
dashboard ac-user-add-roles <username>  Add roles to user
 [<roles>...]
dashboard ac-user-create <username>     Create a user. Password read from -i
 [<rolename>] [<name>] [<email>] [--     <file>
 enabled] [--force-password] [<pwd_
 expiration_date:int>] [--pwd-update-
 required]
dashboard ac-user-del-roles <username>  Delete roles from user
 [<roles>...]
dashboard ac-user-delete [<username>]   Delete user
dashboard ac-user-disable [<username>]  Disable a user
dashboard ac-user-enable [<username>]   Enable a user
dashboard ac-user-set-info <username>   Set user info
 <name> [<email>]
dashboard ac-user-set-password          Set user password from -i <file>
 <username> [--force-password]
dashboard ac-user-set-password-hash     Set user password bcrypt hash from -i
 <username>                              <file>
dashboard ac-user-set-roles <username>  Set user roles
 [<roles>...]
dashboard ac-user-show [<username>]     Show user info
dashboard create-self-signed-cert       Create self signed certificate
dashboard debug [enable|disable|status] Control and report debug status in Ceph-
                                         Dashboard
dashboard feature [enable|disable|      Enable or disable features in Ceph-Mgr
 status] [rbd|mirroring|iscsi|cephfs|    Dashboard
 rgw|nfs...]
dashboard get-account-lockout-attempts  Get the ACCOUNT_LOCKOUT_ATTEMPTS option
                                         value
dashboard get-alertmanager-api-host     Get the ALERTMANAGER_API_HOST option
                                         value
dashboard get-alertmanager-api-ssl-     Get the ALERTMANAGER_API_SSL_VERIFY
 verify                                  option value
dashboard get-audit-api-enabled         Get the AUDIT_API_ENABLED option value
dashboard get-audit-api-log-payload     Get the AUDIT_API_LOG_PAYLOAD option
                                         value
dashboard get-enable-browsable-api      Get the ENABLE_BROWSABLE_API option
                                         value
dashboard get-ganesha-clusters-rados-   Get the GANESHA_CLUSTERS_RADOS_POOL_
 pool-namespace                          NAMESPACE option value
dashboard get-grafana-api-password      Get the GRAFANA_API_PASSWORD option
                                         value
dashboard get-grafana-api-ssl-verify    Get the GRAFANA_API_SSL_VERIFY option
                                         value
dashboard get-grafana-api-url           Get the GRAFANA_API_URL option value
dashboard get-grafana-api-username      Get the GRAFANA_API_USERNAME option
                                         value
dashboard get-grafana-frontend-api-url  Get the GRAFANA_FRONTEND_API_URL option
                                         value
dashboard get-grafana-update-dashboards Get the GRAFANA_UPDATE_DASHBOARDS
                                         option value
dashboard get-iscsi-api-ssl-            Get the ISCSI_API_SSL_VERIFICATION
 verification                            option value
dashboard get-jwt-token-ttl             Get the JWT token TTL in seconds
dashboard get-prometheus-api-host       Get the PROMETHEUS_API_HOST option value
dashboard get-prometheus-api-ssl-verify Get the PROMETHEUS_API_SSL_VERIFY
                                         option value
dashboard get-pwd-policy-check-         Get the PWD_POLICY_CHECK_COMPLEXITY_
 complexity-enabled                      ENABLED option value
dashboard get-pwd-policy-check-         Get the PWD_POLICY_CHECK_EXCLUSION_LIST_
 exclusion-list-enabled                  ENABLED option value
dashboard get-pwd-policy-check-length-  Get the PWD_POLICY_CHECK_LENGTH_ENABLED
 enabled                                 option value
dashboard get-pwd-policy-check-oldpwd-  Get the PWD_POLICY_CHECK_OLDPWD_ENABLED
 enabled                                 option value
dashboard get-pwd-policy-check-         Get the PWD_POLICY_CHECK_REPETITIVE_
 repetitive-chars-enabled                CHARS_ENABLED option value
dashboard get-pwd-policy-check-         Get the PWD_POLICY_CHECK_SEQUENTIAL_
 sequential-chars-enabled                CHARS_ENABLED option value
dashboard get-pwd-policy-check-         Get the PWD_POLICY_CHECK_USERNAME_
 username-enabled                        ENABLED option value
dashboard get-pwd-policy-enabled        Get the PWD_POLICY_ENABLED option value
dashboard get-pwd-policy-exclusion-list Get the PWD_POLICY_EXCLUSION_LIST
                                         option value
dashboard get-pwd-policy-min-complexity Get the PWD_POLICY_MIN_COMPLEXITY
                                         option value
dashboard get-pwd-policy-min-length     Get the PWD_POLICY_MIN_LENGTH option
                                         value
dashboard get-rest-requests-timeout     Get the REST_REQUESTS_TIMEOUT option
                                         value
dashboard get-rgw-api-access-key        Get the RGW_API_ACCESS_KEY option value
dashboard get-rgw-api-admin-resource    Get the RGW_API_ADMIN_RESOURCE option
                                         value
dashboard get-rgw-api-secret-key        Get the RGW_API_SECRET_KEY option value
dashboard get-rgw-api-ssl-verify        Get the RGW_API_SSL_VERIFY option value
dashboard get-user-pwd-expiration-span  Get the USER_PWD_EXPIRATION_SPAN option
                                         value
dashboard get-user-pwd-expiration-      Get the USER_PWD_EXPIRATION_WARNING_1
 warning-1                               option value
dashboard get-user-pwd-expiration-      Get the USER_PWD_EXPIRATION_WARNING_2
 warning-2                               option value
dashboard grafana dashboards update     Push dashboards to Grafana
dashboard iscsi-gateway-add [<name>]    Add iSCSI gateway configuration.
                                         Gateway URL read from -i <file>
dashboard iscsi-gateway-list            List iSCSI gateways
dashboard iscsi-gateway-rm [<name>]     Remove iSCSI gateway configuration
dashboard reset-account-lockout-        Reset the ACCOUNT_LOCKOUT_ATTEMPTS
 attempts                                option to its default value
dashboard reset-alertmanager-api-host   Reset the ALERTMANAGER_API_HOST option
                                         to its default value
dashboard reset-alertmanager-api-ssl-   Reset the ALERTMANAGER_API_SSL_VERIFY
 verify                                  option to its default value
dashboard reset-audit-api-enabled       Reset the AUDIT_API_ENABLED option to
                                         its default value
dashboard reset-audit-api-log-payload   Reset the AUDIT_API_LOG_PAYLOAD option
                                         to its default value
dashboard reset-enable-browsable-api    Reset the ENABLE_BROWSABLE_API option
                                         to its default value
dashboard reset-ganesha-clusters-rados- Reset the GANESHA_CLUSTERS_RADOS_POOL_
 pool-namespace                          NAMESPACE option to its default value
dashboard reset-grafana-api-password    Reset the GRAFANA_API_PASSWORD option
                                         to its default value
dashboard reset-grafana-api-ssl-verify  Reset the GRAFANA_API_SSL_VERIFY option
                                         to its default value
dashboard reset-grafana-api-url         Reset the GRAFANA_API_URL option to its
                                         default value
dashboard reset-grafana-api-username    Reset the GRAFANA_API_USERNAME option
                                         to its default value
dashboard reset-grafana-frontend-api-   Reset the GRAFANA_FRONTEND_API_URL
 url                                     option to its default value
dashboard reset-grafana-update-         Reset the GRAFANA_UPDATE_DASHBOARDS
 dashboards                              option to its default value
dashboard reset-iscsi-api-ssl-          Reset the ISCSI_API_SSL_VERIFICATION
 verification                            option to its default value
dashboard reset-prometheus-api-host     Reset the PROMETHEUS_API_HOST option to
                                         its default value
dashboard reset-prometheus-api-ssl-     Reset the PROMETHEUS_API_SSL_VERIFY
 verify                                  option to its default value
dashboard reset-pwd-policy-check-       Reset the PWD_POLICY_CHECK_COMPLEXITY_
 complexity-enabled                      ENABLED option to its default value
dashboard reset-pwd-policy-check-       Reset the PWD_POLICY_CHECK_EXCLUSION_
 exclusion-list-enabled                  LIST_ENABLED option to its default
                                         value
dashboard reset-pwd-policy-check-       Reset the PWD_POLICY_CHECK_LENGTH_
 length-enabled                          ENABLED option to its default value
dashboard reset-pwd-policy-check-       Reset the PWD_POLICY_CHECK_OLDPWD_
 oldpwd-enabled                          ENABLED option to its default value
dashboard reset-pwd-policy-check-       Reset the PWD_POLICY_CHECK_REPETITIVE_
 repetitive-chars-enabled                CHARS_ENABLED option to its default
                                         value
dashboard reset-pwd-policy-check-       Reset the PWD_POLICY_CHECK_SEQUENTIAL_
 sequential-chars-enabled                CHARS_ENABLED option to its default
                                         value
dashboard reset-pwd-policy-check-       Reset the PWD_POLICY_CHECK_USERNAME_
 username-enabled                        ENABLED option to its default value
dashboard reset-pwd-policy-enabled      Reset the PWD_POLICY_ENABLED option to
                                         its default value
dashboard reset-pwd-policy-exclusion-   Reset the PWD_POLICY_EXCLUSION_LIST
 list                                    option to its default value
dashboard reset-pwd-policy-min-         Reset the PWD_POLICY_MIN_COMPLEXITY
 complexity                              option to its default value
dashboard reset-pwd-policy-min-length   Reset the PWD_POLICY_MIN_LENGTH option
                                         to its default value
dashboard reset-rest-requests-timeout   Reset the REST_REQUESTS_TIMEOUT option
                                         to its default value
dashboard reset-rgw-api-access-key      Reset the RGW_API_ACCESS_KEY option to
                                         its default value
dashboard reset-rgw-api-admin-resource  Reset the RGW_API_ADMIN_RESOURCE option
                                         to its default value
dashboard reset-rgw-api-secret-key      Reset the RGW_API_SECRET_KEY option to
                                         its default value
dashboard reset-rgw-api-ssl-verify      Reset the RGW_API_SSL_VERIFY option to
                                         its default value
dashboard reset-user-pwd-expiration-    Reset the USER_PWD_EXPIRATION_SPAN
 span                                    option to its default value
dashboard reset-user-pwd-expiration-    Reset the USER_PWD_EXPIRATION_WARNING_1
 warning-1                               option to its default value
dashboard reset-user-pwd-expiration-    Reset the USER_PWD_EXPIRATION_WARNING_2
 warning-2                               option to its default value
dashboard set-account-lockout-attempts  Set the ACCOUNT_LOCKOUT_ATTEMPTS option
 <value>                                 value
dashboard set-alertmanager-api-host     Set the ALERTMANAGER_API_HOST option
 <value>                                 value
dashboard set-alertmanager-api-ssl-     Set the ALERTMANAGER_API_SSL_VERIFY
 verify <value>                          option value
dashboard set-audit-api-enabled <value> Set the AUDIT_API_ENABLED option value
dashboard set-audit-api-log-payload     Set the AUDIT_API_LOG_PAYLOAD option
 <value>                                 value
dashboard set-enable-browsable-api      Set the ENABLE_BROWSABLE_API option
 <value>                                 value
dashboard set-ganesha-clusters-rados-   Set the GANESHA_CLUSTERS_RADOS_POOL_
 pool-namespace <value>                  NAMESPACE option value
dashboard set-grafana-api-password      Set the GRAFANA_API_PASSWORD option
                                         value read from -i <file>
dashboard set-grafana-api-ssl-verify    Set the GRAFANA_API_SSL_VERIFY option
 <value>                                 value
dashboard set-grafana-api-url <value>   Set the GRAFANA_API_URL option value
dashboard set-grafana-api-username      Set the GRAFANA_API_USERNAME option
 <value>                                 value
dashboard set-grafana-frontend-api-url  Set the GRAFANA_FRONTEND_API_URL option
 <value>                                 value
dashboard set-grafana-update-           Set the GRAFANA_UPDATE_DASHBOARDS
 dashboards <value>                      option value
dashboard set-iscsi-api-ssl-            Set the ISCSI_API_SSL_VERIFICATION
 verification <value>                    option value
dashboard set-jwt-token-ttl <seconds:   Set the JWT token TTL in seconds
 int>
dashboard set-login-credentials         Set the login credentials. Password
 <username>                              read from -i <file>
dashboard set-prometheus-api-host       Set the PROMETHEUS_API_HOST option value
 <value>
dashboard set-prometheus-api-ssl-       Set the PROMETHEUS_API_SSL_VERIFY
 verify <value>                          option value
dashboard set-pwd-policy-check-         Set the PWD_POLICY_CHECK_COMPLEXITY_
 complexity-enabled <value>              ENABLED option value
dashboard set-pwd-policy-check-         Set the PWD_POLICY_CHECK_EXCLUSION_LIST_
 exclusion-list-enabled <value>          ENABLED option value
dashboard set-pwd-policy-check-length-  Set the PWD_POLICY_CHECK_LENGTH_ENABLED
 enabled <value>                         option value
dashboard set-pwd-policy-check-oldpwd-  Set the PWD_POLICY_CHECK_OLDPWD_ENABLED
 enabled <value>                         option value
dashboard set-pwd-policy-check-         Set the PWD_POLICY_CHECK_REPETITIVE_
 repetitive-chars-enabled <value>        CHARS_ENABLED option value
dashboard set-pwd-policy-check-         Set the PWD_POLICY_CHECK_SEQUENTIAL_
 sequential-chars-enabled <value>        CHARS_ENABLED option value
dashboard set-pwd-policy-check-         Set the PWD_POLICY_CHECK_USERNAME_
 username-enabled <value>                ENABLED option value
dashboard set-pwd-policy-enabled        Set the PWD_POLICY_ENABLED option value
 <value>
dashboard set-pwd-policy-exclusion-     Set the PWD_POLICY_EXCLUSION_LIST
 list <value>                            option value
dashboard set-pwd-policy-min-           Set the PWD_POLICY_MIN_COMPLEXITY
 complexity <value>                      option value
dashboard set-pwd-policy-min-length     Set the PWD_POLICY_MIN_LENGTH option
 <value>                                 value
dashboard set-rest-requests-timeout     Set the REST_REQUESTS_TIMEOUT option
 <value>                                 value
dashboard set-rgw-api-access-key        Set the RGW_API_ACCESS_KEY option value
                                         read from -i <file>
dashboard set-rgw-api-admin-resource    Set the RGW_API_ADMIN_RESOURCE option
 <value>                                 value
dashboard set-rgw-api-secret-key        Set the RGW_API_SECRET_KEY option value
                                         read from -i <file>
dashboard set-rgw-api-ssl-verify        Set the RGW_API_SSL_VERIFY option value
 <value>
dashboard set-user-pwd-expiration-span  Set the USER_PWD_EXPIRATION_SPAN option
 <value>                                 value
dashboard set-user-pwd-expiration-      Set the USER_PWD_EXPIRATION_WARNING_1
 warning-1 <value>                       option value
dashboard set-user-pwd-expiration-      Set the USER_PWD_EXPIRATION_WARNING_2
 warning-2 <value>                       option value
dashboard sso disable                   Disable Single Sign-On
dashboard sso enable saml2              Enable SAML2 Single Sign-On
dashboard sso setup saml2 <ceph_Setup SAML2 Single Sign-On
 dashboard_base_url> <idp_metadata>
 [<idp_username_attribute>] [<idp_
 entity_id>] [<sp_x_509_cert>] [<sp_
 private_key>]
dashboard sso show saml2                Show SAML2 configuration
dashboard sso status                    Get Single Sign-On status
device check-health                     Check life expectancy of devices
device get-health-metrics <devid>       Show stored device metrics for the
 [<sample>]                              device
device info <devid>                     Show information about a device
device light on|off <devid> [ident|     Enable or disable the device light.
 fault] [--force]                        Default type is `ident` 'Usage: device
                                         light (on|off) <devid> [ident|fault] [-
                                         -force]'
device ls                               Show devices
device ls-by-daemon <who>               Show devices associated with a daemon
device ls-by-host <host>                Show devices on a host
device ls-lights                        List currently active device indicator
                                         lights
device monitoring off                   Disable device health monitoring
device monitoring on                    Enable device health monitoring
device predict-life-expectancy <devid>  Predict life expectancy with local
                                         predictor
device query-daemon-health-metrics      Get device health metrics for a given
 <who>                                   daemon
device rm-life-expectancy <devid>       Clear predicted device life expectancy
device scrape-daemon-health-metrics     Scrape and store device health metrics
 <who>                                   for a given daemon
device scrape-health-metrics [<devid>]  Scrape and store device health metrics
device set-life-expectancy <devid>      Set predicted device life expectancy
 <from> [<to>]
df [detail]                             show cluster free space stats
features                                report of connected features
fs add_data_pool <fs_name> <pool>       add data pool <pool>
fs authorize <filesystem> <entity> <caps>...       add auth for <entity> to access file system <filesystem> based on following directory and permissions pairs                                  
fs clone cancel <vol_name> < name> [<group_name>]  clone_Cancel an pending or ongoing clone operation.               
fs clone status <vol_name> <clone_Get  name> [<group_name>]             status on a cloned subvolume.
fs compat <fs_name> rm_compat|rm_incompat|add_compat|add_incompat <feature:int> [<feature_str>]       manipulate compat settings
fs compat show <fs_name>                show fs compatibility settings
fs dump [<epoch:int>]                   dump all CephFS status, optionally from epoch                                   
fs fail <fs_name>                       bring the file system down and all of its ranks                                   
fs feature ls                           list available cephfs features to be set/unset                                    
fs flag set enable_multiple <val> [--yes-i-really-mean-it]   Set a global CephFS flag
fs get <fs_name>                        get info about one filesystem
fs ls                                   list filesystems
fs mirror disable <fs_name>             disable mirroring for a ceph filesystem
fs mirror enable <fs_name>              enable mirroring for a ceph filesystem
fs mirror peer_add <fs_name> <uuid> <remote_cluster_spec> <remote_fs_name>    add a mirror peer for a ceph filesystem
 
fs mirror peer_remove <fs_name> <uuid>  remove a mirror peer for a ceph filesystem                                
fs new <fs_name> <metadata> <data> [--  make new filesystem using named pools
 force] [--allow-dangerous-metadata-     <metadata> and <data>
 overlay] [<fscid:int>]
fs perf stats [<mds_rank>] [<client_    retrieve ceph fs performance stats
 id>] [<client_ip>]
fs required_client_features <fs_name>   add/remove required features of clients
 add|rm <val>
fs reset <fs_name> [--yes-i-really-     disaster recovery only: reset to a
 mean-it]                                single-MDS map
fs rm <fs_name> [--yes-i-really-mean-   disable the named filesystem
 it]
fs rm_data_pool <fs_name> <pool>        remove data pool <pool>
fs set <fs_name> max_mds|max_file_size| set fs parameter <var> to <val>
 allow_new_snaps|inline_data|cluster_
 down|allow_dirfrags|balancer|standby_  
 count_wanted|session_timeout|session_  
 autoclose|allow_standby_replay|down|
 joinable|min_compat_client <val> [--
 yes-i-really-mean-it] [--yes-i-really-
 really-mean-it]
fs set-default <fs_name>                set the default to the named filesystem
fs snap-schedule activate [<path>]      Activate a snapshot schedule for <path>
 [<repeat>] [<start>] [<subvol>] [<fs>]
fs snap-schedule add <path> [<snap_Set a snapshot schedule for <path>
 schedule>] [<start>] [<fs>] [<subvol>]
fs snap-schedule deactivate [<path>]    Deactivate a snapshot schedule for
 [<repeat>] [<start>] [<subvol>] [<fs>]  <path>
fs snap-schedule list [<path>]          Get current snapshot schedule for <path>
 [<subvol>] [--recursive] [<fs>]
 [<format>]
fs snap-schedule remove [<path>]        Remove a snapshot schedule for <path>
 [<repeat>] [<start>] [<subvol>] [<fs>]
fs snap-schedule retention add <path>   Set a retention specification for <path>
 [<retention_spec_or_period>]
 [<retention_count>] [<fs>] [<subvol>]  
fs snap-schedule retention remove       Remove a retention specification for
 <path> [<retention_spec_or_period>]     <path>
 [<retention_count>] [<fs>] [<subvol>]  
fs snap-schedule status [<path>]        List current snapshot schedules
 [<subvol>] [<fs>] [<format>]
fs snapshot mirror add <fs_name>        Add a directory for snapshot mirroring
 [<path>]
fs snapshot mirror daemon status        Get mirror daemon status
fs snapshot mirror dirmap <fs_name>     Get current mirror instance map for a
 [<path>]                                directory
fs snapshot mirror disable [<fs_name>]  Disable snapshot mirroring for a
                                         filesystem
fs snapshot mirror enable [<fs_name>]   Enable snapshot mirroring for a
                                         filesystem
fs snapshot mirror peer_add <fs_name>   Add a remote filesystem peer
 [<remote_cluster_spec>] [<remote_fs_
 name>] [<remote_mon_host>] [<cephx_
 key>]
fs snapshot mirror peer_bootstrap       Bootstrap a filesystem peer
 create <fs_name> <client_name> [<site_
 name>]
fs snapshot mirror peer_bootstrap       Import a bootstrap token
 import <fs_name> [<token>]
fs snapshot mirror peer_list [<fs_      List configured peers for a file system
 name>]
fs snapshot mirror peer_remove <fs_     Remove a filesystem peer
 name> [<peer_uuid>]
fs snapshot mirror remove <fs_name>     Remove a snapshot mirrored directory
 [<path>]
fs snapshot mirror show distribution    Get current instance to directory map
 [<fs_name>]                             for a filesystem
fs status [<fs>]                        Show the status of a CephFS filesystem
fs subvolume authorize <vol_name> <sub_Allow a cephx auth ID access to a
 name> <auth_id> [<group_name>]          subvolume
 [<access_level>] [<tenant_id>] [--
 allow-existing-id]
fs subvolume authorized_list <vol_      List auth IDs that have access to a
 name> <sub_name> [<group_name>]         subvolume
fs subvolume create <vol_name> <sub_Create a CephFS subvolume in a volume,
 name> [<size:int>] [<group_name>]       and optionally, with a specific size (
 [<pool_layout>] [<uid:int>] [<gid:      in bytes), a specific data pool layout,
 int>] [<mode>] [--namespace-isolated]   a specific mode, in a specific
                                         subvolume group and in separate RADOS
                                         namespace
fs subvolume deauthorize <vol_name>     Deny a cephx auth ID access to a
 <sub_name> <auth_id> [<group_name>]     subvolume
fs subvolume evict <vol_name> <sub_     Evict clients based on auth IDs and
 name> <auth_id> [<group_name>]          subvolume mounted
fs subvolume getpath <vol_name> <sub_Get the mountpath of a CephFS subvolume
 name> [<group_name>]                    in a volume, and optionally, in a
                                         specific subvolume group
fs subvolume info <vol_name> <sub_Get the metadata of a CephFS subvolume
 name> [<group_name>]                    in a volume, and optionally, in a
                                         specific subvolume group
fs subvolume ls <vol_name> [<group_List subvolumes
 name>]
fs subvolume pin <vol_name> <sub_name>  Set MDS pinning policy for subvolume
 export|distributed|random <pin_
 setting> [<group_name>]
fs subvolume resize <vol_name> <sub_    Resize a CephFS subvolume
 name> <new_size> [<group_name>] [--no-
 shrink]
fs subvolume rm <vol_name> <sub_name>   Delete a CephFS subvolume in a volume,
 [<group_name>] [--force] [--retain-     and optionally, in a specific
 snapshots]                              subvolume group, force deleting a
                                         cancelled or failed clone, and
                                         retaining existing subvolume snapshots
fs subvolume snapshot clone <vol_name>  Clone a snapshot to target subvolume
 <sub_name> <snap_name> <target_sub_
 name> [<pool_layout>] [<group_name>]
 [<target_group_name>]
fs subvolume snapshot create <vol_Create a snapshot of a CephFS subvolume
 name> <sub_name> <snap_name> [<group_   in a volume, and optionally, in a
 name>]                                  specific subvolume group
fs subvolume snapshot info <vol_name>   Get the metadata of a CephFS subvolume
 <sub_name> <snap_name> [<group_name>]   snapshot and optionally, in a specific
                                         subvolume group
fs subvolume snapshot ls <vol_name>     List subvolume snapshots
 <sub_name> [<group_name>]
fs subvolume snapshot protect <vol_     (deprecated) Protect snapshot of a
 name> <sub_name> <snap_name> [<group_CephFS subvolume in a volume, and
 name>]                                  optionally, in a specific subvolume
                                         group
fs subvolume snapshot rm <vol_name>     Delete a snapshot of a CephFS subvolume
 <sub_name> <snap_name> [<group_name>]   in a volume, and optionally, in a
 [--force]                               specific subvolume group
fs subvolume snapshot unprotect <vol_(deprecated) Unprotect a snapshot of a
 name> <sub_name> <snap_name> [<group_   CephFS subvolume in a volume, and
 name>]                                  optionally, in a specific subvolume
                                         group
fs subvolumegroup create <vol_name>     Create a CephFS subvolume group in a
 <group_name> [<pool_layout>] [<uid:     volume, and optionally, with a
 int>] [<gid:int>] [<mode>]              specific data pool layout, and a
                                         specific numeric mode
fs subvolumegroup getpath <vol_name>    Get the mountpath of a CephFS subvolume
 <group_name>                            group in a volume
fs subvolumegroup ls <vol_name>         List subvolumegroups
fs subvolumegroup pin <vol_name>        Set MDS pinning policy for
 <group_name> export|distributed|        subvolumegroup
 random <pin_setting>
fs subvolumegroup rm <vol_name> <group_ Delete a CephFS subvolume group in a
 name> [--force]                         volume
fs subvolumegroup snapshot create <vol_Create a snapshot of a CephFS subvolume
 name> <group_name> <snap_name>          group in a volume
fs subvolumegroup snapshot ls <vol_     List subvolumegroup snapshots
 name> <group_name>
fs subvolumegroup snapshot rm <vol_     Delete a snapshot of a CephFS subvolume
 name> <group_name> <snap_name> [--      group in a volume
 force]
fs volume create <name> [<placement>]   Create a CephFS volume
fs volume ls                            List volumes
fs volume rm <vol_name> [<yes-i-really- Delete a FS volume by passing --yes-i-
 mean-it>]                               really-mean-it flag
fsid                                    show cluster FSID/UUID
health [detail]                         show cluster health
health mute <code> [<ttl>] [--sticky]   mute health alert
health unmute [<code>]                  unmute existing health alert mute(s)
influx config-set <key> <value>         Set a configuration value
influx config-show                      Show current configuration
influx send                             Force sending data to Influx
insights                                Retrieve insights report
insights prune-health [<hours:int>]     Remove health history older than
                                         <hours> hours
iostat [<width:int>] [--print-header]   Get IO rates
k8sevents ceph                          List Ceph events tracked & sent to the
                                         kubernetes cluster
k8sevents clear-config                  Clear external kubernetes configuration
                                         settings
k8sevents ls                            List all current Kuberenetes events
                                         from the Ceph namespace
k8sevents set-access <key>              Set kubernetes access credentials.
                                         <key> must be cacrt or token and use -
                                         i <filename> syntax (e.g., ceph k8
                                         sevents set-access cacrt -i /root/ca.
                                         crt).
k8sevents set-config <key> <value>      Set kubernetes config paramters. <key>
                                         must be server or namespace (e.g.,
                                         ceph k8sevents set-config server https:
                                         //localhost:30433).
k8sevents status                        Show the status of the data gathering
                                         threads
log <logtext>...                        log supplied text to the monitor log
log last [<num:int>] [debug|info|sec|   print last few lines of the cluster log
 warn|error] [*|cluster|audit|cephadm]  
mds count-metadata <property>           count MDSs by metadata field property
mds fail <role_or_gid>                  Mark MDS failed: trigger a failover if
                                         a standby is available
mds metadata [<who>]                    fetch metadata for mds <role>
mds ok-to-stop <ids>...                 check whether stopping the specified
                                         MDS would reduce immediate availability
mds repaired <role>                     mark a damaged MDS rank as no longer
                                         damaged
mds rm <gid:int>                        remove nonactive mds
mds versions                            check running versions of MDSs
mgr count-metadata <property>           count ceph-mgr daemons by metadata
                                         field property
mgr dump [<epoch:int>]                  dump the latest MgrMap
mgr fail [<who>]                        treat the named manager daemon as failed
mgr metadata [<who>]                    dump metadata for all daemons or a
                                         specific daemon
mgr module disable <module>             disable mgr module
mgr module enable <module> [--force]    enable mgr module
mgr module ls                           list active mgr modules
mgr self-test background start          Activate a background workload (one of
 <workload>                              command_spam, throw_exception)
mgr self-test background stop           Stop background workload if any is
                                         running
mgr self-test cluster-log <channel>     Create an audit log record.
 <priority> <message>
mgr self-test config get <key>          Peek at a configuration value
mgr self-test config get_localized      Peek at a configuration value (
 <key>                                   localized variant)
mgr self-test health clear [<checks>... Clear health checks by name. If no
 ]                                       names provided, clear all.
mgr self-test health set <checks>       Set a health check from a JSON-
                                         formatted description.
mgr self-test insights_set_now_offset   Set the now time for the insights
 <hours>                                 module.
mgr self-test module <module>           Run another module's self_test() method
mgr self-test python-version            Query the version of the embedded
                                         Python runtime
mgr self-test remote                    Test inter-module calls
mgr self-test run                       Run mgr python interface tests
mgr services                            list service endpoints provided by mgr
                                         modules
mgr stat                                dump basic info about the mgr cluster
                                         state
mgr versions                            check running versions of ceph-mgr
                                         daemons
mon add <name> <addr> [<location>...]   add new monitor named <name> at <addr>,
                                         possibly with CRUSH location <location>
mon add disallowed_leader <name>        prevent the named mon from being a
                                         leader
mon count-metadata <property>           count mons by metadata field property
mon dump [<epoch:int>]                  dump formatted monmap (optionally from
                                         epoch)
mon enable-msgr2                        enable the msgr2 protocol on port 3300
mon enable_stretch_mode <tiebreaker_    enable stretch mode, changing the
 mon> <new_crush_rule> <dividing_peering rules and failure handling on
 bucket>                                 all pools with <tiebreaker_mon> as the
                                         tiebreaker and setting <dividing_
                                         bucket> locations as the units for
                                         stretching across
mon feature ls [--with-value]           list available mon map features to be
                                         set/unset
mon feature set <feature_name> [--yes-  set provided feature on mon map
 i-really-mean-it]
mon getmap [<epoch:int>]                get monmap
mon metadata [<id>]                     fetch metadata for mon <id>
mon ok-to-add-offline                   check whether adding a mon and not
                                         starting it would break quorum
mon ok-to-rm <id>                       check whether removing the specified
                                         mon would break quorum
mon ok-to-stop <ids>...                 check whether mon(s) can be safely
                                         stopped without reducing immediate
                                         availability
mon rm <name>                           remove monitor named <name>
mon rm disallowed_leader <name>         allow the named mon to be a leader again
mon scrub                               scrub the monitor stores
mon set election_strategy <strategy>    set the election strategy to use;
                                         choices classic, disallow, connectivity
mon set-addrs <name> <addrs>            set the addrs (IPs and ports) a
                                         specific monitor binds to
mon set-rank <name> <rank:int>          set the rank for the specified mon
mon set-weight <name> <weight:int>      set the weight for the specified mon
mon set_location <name> <args>...       specify location <args> for the monitor
                                         <name>, using CRUSH bucket names
mon set_new_tiebreaker <name> [--yes-i- switch the stretch tiebreaker to be the
 really-mean-it]                         named mon
mon stat                                summarize monitor status
mon versions                            check running versions of monitors
nfs cluster config get <cluster_id>     Fetch NFS-Ganesha config
nfs cluster config reset <cluster_id>   Reset NFS-Ganesha Config to default
nfs cluster config set <cluster_id>     Set NFS-Ganesha config by `-i <config_
                                         file>`
nfs cluster create <cluster_id>         Create an NFS Cluster
 [<placement>] [--ingress] [<virtual_
 ip>] [<port:int>]
nfs cluster delete <cluster_id>         Removes an NFS Cluster (DEPRECATED)
nfs cluster info [<cluster_id>]         Displays NFS Cluster info
nfs cluster ls                          List NFS Clusters
nfs cluster rm <cluster_id>             Removes an NFS Cluster
nfs export apply <cluster_id>           Create or update an export by `-i <json_
                                         or_ganesha_export_file>`
nfs export create cephfs <cluster_id>   Create a CephFS export
 <pseudo_path> <fsname> [<path>] [--
 readonly] [<client_addr>...]
 [<squash>]
nfs export create rgw <cluster_id>      Create an RGW export
 <pseudo_path> [<bucket>] [<user_id>]
 [--readonly] [<client_addr>...]
 [<squash>]
nfs export delete <cluster_id> <pseudo_Delete a cephfs export (DEPRECATED)
 path>
nfs export get <cluster_id> <pseudo_Fetch a export of a NFS cluster given
 path>                                   the pseudo path/binding (DEPRECATED)
nfs export info <cluster_id> <pseudo_Fetch a export of a NFS cluster given
 path>                                   the pseudo path/binding
nfs export ls <cluster_id> [--detailed] List exports of a NFS cluster
nfs export rm <cluster_id> <pseudo_     Remove a cephfs export
 path>
node ls [all|osd|mon|mds|mgr]           list all nodes in cluster [type]
orch apply [mon|mgr|rbd-mirror|cephfs-  Update the size or placement for a
 mirror|crash|alertmanager|grafana|      service or apply a large yaml spec
 node-exporter|prometheus|mds|rgw|nfs|  
 iscsi|cephadm-exporter] [<placement>]  
 [--dry-run] [--format {plain|json|
 json-pretty|yaml}] [--unmanaged] [--
 no-overwrite]
orch apply iscsi <pool> <api_user>      Scale an iSCSI service
 <api_password> [<trusted_ip_list>]
 [<placement>] [--unmanaged] [--dry-
 run] [--format {plain|json|json-
 pretty|yaml}] [--no-overwrite]
orch apply mds <fs_name> [<placement>]  Update the number of MDS instances for
 [--dry-run] [--unmanaged] [--format     the given fs_name
 {plain|json|json-pretty|yaml}] [--no-  
 overwrite]
orch apply nfs <svc_id> [<placement>]   Scale an NFS service
 [--format {plain|json|json-pretty|
 yaml}] [<port:int>] [--dry-run] [--
 unmanaged] [--no-overwrite]
orch apply osd [--all-available-devices] [--format {plain|json|json-pretty|yaml}] [--unmanaged] [--dry-run] [--no-overwrite]     Create OSD daemon(s) using a drive group spec
orch apply rgw <svc_id> [<placement>] [<realm>] [<zone>] [<port:int>] [--ssl] [--dry-run] [--format {plain| json|json-pretty|yaml}] [--unmanaged] [--no-overwrite]   Update the number of RGW instances for the given zone
orch cancel                             Cancel ongoing background operations
orch client-keyring ls [--format {plain|json|json-pretty|yaml}]       List client keyrings under cephadm management       
orch client-keyring rm <entity>         Remove client keyring from cephadm management
orch client-keyring set <entity> <placement> [<owner>] [<mode>]       Add or update client keyring under cephadm management     
orch daemon add [mon|mgr|rbd-mirror| cephfs-mirror|crash|alertmanager| grafana|node-exporter|prometheus|mds|rgw|nfs|iscsi|cephadm-exporter]  [<placement>]     Add daemon(s)
orch daemon add iscsi <pool> <api_Start iscsi daemon(s)
 user> <api_password> [<trusted_ip_
 list>] [<placement>]
orch daemon add mds <fs_name>           Start MDS daemon(s)
 [<placement>]
orch daemon add nfs <svc_id>            Start NFS daemon(s)
 [<placement>]
orch daemon add osd [<svc_arg>]         Create an OSD service. Either --svc_arg=
                                         host:drives
orch daemon add rgw <svc_id>            Start RGW daemon(s)
 [<placement>] [<port:int>] [--ssl]
orch daemon redeploy <name> [<image>]   Redeploy a daemon (with a specifc image)
orch daemon rm <names>... [--force]     Remove specific daemon(s)
orch daemon start|stop|restart|         Start, stop, restart, (redeploy,) or
 reconfig <name>                         reconfig a specific daemon
orch device ls [<hostname>...] [--      List devices on a host
 format {plain|json|json-pretty|yaml}]  
 [--refresh] [--wide]
orch device zap <hostname> <path> [--   Zap (erase!) a device so it can be re-
 force]                                  used
orch host add <hostname> [<addr>]       Add a host
 [<labels>...] [--maintenance]
orch host drain <hostname>              drain all daemons from a host
orch host label add <hostname> <label>  Add a host label
orch host label rm <hostname> <label>   Remove a host label
orch host ls [--format {plain|json|     List hosts
 json-pretty|yaml}]
orch host maintenance enter <hostname>  Prepare a host for maintenance by
 [--force]                               shutting down and disabling all Ceph
                                         daemons (cephadm only)
orch host maintenance exit <hostname>   Return a host from maintenance, restarting all Ceph daemons (cephadm
 only)
orch host ok-to-stop <hostname>         Check if the specified host can be safely stopped without reducing availability
orch host rm <hostname> [--force] [--   Remove a host offline]
orch host set-addr <hostname> <addr>    Update a host address
orch ls [<service_type>] [<service_List services known to orchestrator name>] [--export] [--format {plain| json|json-pretty|yaml}] [--refresh]
orch osd rm <osd_id>... [--replace] [-- Remove OSD daemons force] [--zap]
orch osd rm status [--format {plain|    Status of OSD removal operation json|json-pretty|yaml}]
orch osd rm stop <osd_id>...            Cancel ongoing OSD removal operation
orch pause                              Pause orchestrator background work
orch ps [<hostname>] [<service_name>]   List daemons known to orchestrator [<daemon_type>] [<daemon_id>] [-- format {plain|json|json-pretty|yaml}]  [--refresh]
orch resume                             Resume orchestrator background work (if paused)
orch rm <service_name> [--force]        Remove a service
orch set backend [<module_name>]        Select orchestrator module backend
orch start|stop|restart|redeploy|       Start, stop, restart, redeploy, or
 reconfig <service_name>                reconfig an entire service (i.e. all daemons)
orch status [--detail] [--format        Report configured backend and its status{plain|json|json-pretty|yaml}]
orch upgrade check [<image>] [<ceph_    Check service versions vs available and
 version>]                               target containers
orch upgrade ls [<image>] [--tags]      Check for available versions (or tags)
                                         we can upgrade to
orch upgrade pause                      Pause an in-progress upgrade
orch upgrade resume                     Resume paused upgrade
orch upgrade start [<image>] [<ceph_Initiate upgrade
 version>]
orch upgrade status                     Check service versions vs available and
                                         target containers
orch upgrade stop                       Stop an in-progress upgrade
osd blocked-by                          print histogram of which OSDs are
                                         blocking their peers
osd blocklist add|rm <addr> [<expire:   add (optionally until <expire> seconds
 float>]                                 from now) or remove <addr> from
                                         blocklist
osd blocklist clear                     clear all blocklisted clients
osd blocklist ls                        show blocklisted clients
osd count-metadata <property>           count OSDs by metadata field property
osd crush add <id|osd.id> <weight:      add or update crushmap position and
 float> <args>...                        weight for <name> with <weight> and
                                         location <args>
osd crush add-bucket <name> <type>      add no-parent (probably root) crush
 [<args>...]                             bucket <name> of type <type> to
                                         location <args>
osd crush class create <class>          create crush device class <class>
osd crush class ls                      list all crush device classes
osd crush class ls-osd <class>          list all osds belonging to the specific
                                         <class>
osd crush class rename <srcname>        rename crush device class <srcname> to
 <dstname>                               <dstname>
osd crush class rm <class>              remove crush device class <class>
osd crush create-or-move <id|osd.id>    create entry or move existing entry for
 <weight:float> <args>...                <name> <weight> at/to location <args>
osd crush dump                          dump crush map
osd crush get-device-class <ids>...     get classes of specified osd(s) <id>
                                         [<id>...]
osd crush get-tunable straw_calc_get crush tunable <tunable>
 version
osd crush link <name> <args>...         link existing entry for <name> under
                                         location <args>
osd crush ls <node>                     list items beneath a node in the CRUSH
                                         tree
osd crush move <name> <args>...         move existing entry for <name> to
                                         location <args>
osd crush rename-bucket <srcname>       rename bucket <srcname> to <dstname>
 <dstname>
osd crush reweight <name> <weight:      change <name>'s weight to <weight> in
 float>                                  crush map
osd crush reweight-all                  recalculate the weights for the tree to
                                         ensure they sum correctly
osd crush reweight-subtree <name>       change all leaf items beneath <name> to
 <weight:float>                          <weight> in crush map
osd crush rm <name> [<ancestor>]        remove <name> from crush map (
                                         everywhere, or just at <ancestor>)
osd crush rm-device-class <ids>...      remove class of the osd(s) <id> [<id>...
                                         ],or use <all|any> to remove all.
osd crush rule create-erasure <name>    create crush rule <name> for erasure
 [<profile>]                             coded pool created with <profile> (
                                         default default)
osd crush rule create-replicated        create crush rule <name> for replicated
 <name> <root> <type> [<class>]          pool to start from <root>, replicate
                                         across buckets of type <type>, use
                                         devices of type <class> (ssd or hdd)
osd crush rule create-simple <name>     create crush rule <name> to start from
 <root> <type> [firstn|indep]            <root>, replicate across buckets of
                                         type <type>, using a choose mode of
                                         <firstn|indep> (default firstn; indep
                                         best for erasure pools)
osd crush rule dump [<name>]            dump crush rule <name> (default all)
osd crush rule ls                       list crush rules
osd crush rule ls-by-class <class>      list all crush rules that reference the
                                         same <class>
osd crush rule rename <srcname>         rename crush rule <srcname> to <dstname>
 <dstname>
osd crush rule rm <name>                remove crush rule <name>
osd crush set <id|osd.id> <weight:      update crushmap position and weight for
 float> <args>...                        <name> to <weight> with location <args>
osd crush set [<prior_version:int>]     set crush map from input file
osd crush set-all-straw-buckets-to-     convert all CRUSH current straw buckets
 straw2                                  to use the straw2 algorithm
osd crush set-device-class <class>      set the <class> of the osd(s) <id>
 <ids>...                                [<id>...],or use <all|any> to set all.
osd crush set-tunable straw_calc_set crush tunable <tunable> to <value>
 version <value:int>
osd crush show-tunables                 show current crush tunables
osd crush swap-bucket <source> <dest>   swap existing bucket contents from (
 [--yes-i-really-mean-it]                orphan) bucket <source> and <target>
osd crush tree [--show-shadow]          dump crush buckets and items in a tree
                                         view
osd crush tunables legacy|argonaut|     set crush tunables values to <profile>
 bobtail|firefly|hammer|jewel|optimal|  
 default
osd crush unlink <name> [<ancestor>]    unlink <name> from crush map (
                                         everywhere, or just at <ancestor>)
osd crush weight-set create <pool>      create a weight-set for a given pool
 flat|positional
osd crush weight-set create-compat      create a default backward-compatible
                                         weight-set
osd crush weight-set dump               dump crush weight sets
osd crush weight-set ls                 list crush weight sets
osd crush weight-set reweight <pool>    set weight for an item (bucket or osd)
 <item> <weight:float>...                in a pool's weight-set
osd crush weight-set reweight-compat    set weight for an item (bucket or osd)
 <item> <weight:float>...                in the backward-compatible weight-set
osd crush weight-set rm <pool>          remove the weight-set for a given pool
osd crush weight-set rm-compat          remove the backward-compatible weight-
                                         set
osd deep-scrub <who>                    initiate deep scrub on osd <who>, or
                                         use <all|any> to deep scrub all
osd destroy <id|osd.id> [--force] [--   mark osd as being destroyed. Keeps the
 yes-i-really-mean-it]                   ID intact (allowing reuse), but
                                         removes cephx keys, config-key data
                                         and lockbox keys, rendering data
                                         permanently unreadable.
osd df [plain|tree] [class|name]        show OSD utilization
 [<filter>]
osd down <ids>... [--definitely-dead]   set osd(s) <id> [<id>...] down, or use
                                         <any|all> to set all osds down
osd dump [<epoch:int>]                  print summary of OSD map
osd erasure-code-profile get <name>     get erasure code profile <name>
osd erasure-code-profile ls             list all erasure code profiles
osd erasure-code-profile rm <name>      remove erasure code profile <name>
osd erasure-code-profile set <name>     create erasure code profile <name> with
 [<profile>...] [--force]                [<key[=value]> ...] pairs. Add a --
                                         force at the end to override an
                                         existing profile (VERY DANGEROUS)
osd find <id|osd.id>                    find osd <id> in the CRUSH map and show
                                         its location
osd force-create-pg <pgid> [--yes-i-    force creation of pg <pgid>
 really-mean-it]
osd force_healthy_stretch_mode [--yes-  force a healthy stretch mode, requiring
 i-really-mean-it]                       the full number of CRUSH buckets to
                                         peer and letting all non-tiebreaker
                                         monitors be elected leader
osd force_recovery_stretch_mode [--yes- try and force a recovery stretch mode,
 i-really-mean-it]                       increasing the pool size to its non-
                                         failure value if currently degraded
                                         and all monitor buckets are up
osd get-require-min-compat-client       get the minimum client version we will
                                         maintain compatibility with
osd getcrushmap [<epoch:int>]           get CRUSH map
osd getmap [<epoch:int>]                get OSD map
osd getmaxosd                           show largest OSD id
osd in <ids>...                         set osd(s) <id> [<id>...] in, can use
                                         <any|all> to automatically set all
                                         previously out osds in
osd info [<id|osd.id>]                  print osd's {id} information (instead
                                         of all osds from map)
osd last-stat-seq <id|osd.id>           get the last pg stats sequence number
                                         reported for this osd
osd lost <id|osd.id> [--yes-i-really-   mark osd as permanently lost. THIS
 mean-it]                                DESTROYS DATA IF NO MORE REPLICAS
                                         EXIST, BE CAREFUL
osd ls [<epoch:int>]                    show all OSD ids
osd ls-tree [<epoch:int>] <name>        show OSD ids under bucket <name> in the
                                         CRUSH map
osd map <pool> <object> [<nspace>]      find pg for <object> in <pool> with
                                         [namespace]
osd metadata [<id|osd.id>]              fetch metadata for osd {id} (default
                                         all)
osd new <uuid> [<id|osd.id>]            Create a new OSD. If supplied, the `id`
                                         to be replaced needs to exist and have
                                         been previously destroyed. Reads
                                         secrets from JSON file via `-i <file>`
                                         (see man page).
osd numa-status                         show NUMA status of OSDs
osd ok-to-stop <ids>... [<max:int>]     check whether osd(s) can be safely
                                         stopped without reducing immediate
                                         data availability
osd out <ids>...                        set osd(s) <id> [<id>...] out, or use
                                         <any|all> to set all osds out
osd pause                               pause osd
osd perf                                print dump of OSD perf summary stats
osd pg-temp <pgid> [<id|osd.id>...]     set pg_temp mapping pgid:[<id> [<id>...
                                         ]] (developers only)
osd pg-upmap <pgid> <id|osd.id>...      set pg_upmap mapping <pgid>:[<id> [<id>.
                                         ..]] (developers only)
osd pg-upmap-items <pgid> <id|osd.id>.. set pg_upmap_items mapping <pgid>:{<id>
 .                                       to <id>, [...]} (developers only)
osd pool application disable <pool> <app> [--yes-i-really-mean-it]     disables use of an application <app> on pool <poolname>
osd pool application enable <pool>      enable use of an application <app>
 <app> [--yes-i-really-mean-it]          [cephfs,rbd,rgw] on pool <poolname>
osd pool application get [<pool>]       get value of key <key> of application
 [<app>] [<key>]                         <app> on pool <poolname>
osd pool application rm <pool> <app>    removes application <app> metadata key
 <key>                                   <key> on pool <poolname>
osd pool application set <pool> <app>   sets application <app> metadata key
 <key> <value>                           <key> to <value> on pool <poolname>
osd pool autoscale-status [<format>]    report on pool pg_num sizing
                                         recommendation and intent
osd pool cancel-force-backfill <who>... restore normal recovery priority of
                                         specified pool <who>
osd pool cancel-force-recovery <who>... restore normal recovery priority of
                                         specified pool <who>
osd pool create <pool> [<pg_num:int>]   create pool
 [<pgp_num:int>] [replicated|erasure]
 [<erasure_code_profile>] [<rule>]
 [<expected_num_objects:int>] [<size:
 int>] [<pg_num_min:int>] [on|off|
 warn] [<target_size_bytes:int>]
 [<target_size_ratio:float>]
osd pool deep-scrub <who>...            initiate deep-scrub on pool <who>
osd pool force-backfill <who>...        force backfill of specified pool <who>
                                         first
osd pool force-recovery <who>...        force recovery of specified pool <who>
                                         first
osd pool get <pool> size|min_size|pg_   get pool parameter <var>
 num|pgp_num|crush_rule|hashpspool|
 nodelete|nopgchange|nosizechange|
 write_fadvise_dontneed|noscrub|nodeep-
 scrub|hit_set_type|hit_set_period|hit_
 set_count|hit_set_fpp|use_gmt_hitset|  
 target_max_objects|target_max_bytes|
 cache_target_dirty_ratio|cache_target_
 dirty_high_ratio|cache_target_full_
 ratio|cache_min_flush_age|cache_min_
 evict_age|erasure_code_profile|min_
 read_recency_for_promote|all|min_
 write_recency_for_promote|fast_read|
 hit_set_grade_decay_rate|hit_set_
 search_last_n|scrub_min_interval|
 scrub_max_interval|deep_scrub_
 interval|recovery_priority|recovery_
 op_priority|scrub_priority|
 compression_mode|compression_
 algorithm|compression_required_ratio|  
 compression_max_blob_size|compression_
 min_blob_size|csum_type|csum_min_
 block|csum_max_block|allow_ec_
 overwrites|fingerprint_algorithm|pg_
 autoscale_mode|pg_autoscale_bias|pg_
 num_min|target_size_bytes|target_size_
 ratio|dedup_tier|dedup_chunk_
 algorithm|dedup_cdc_chunk_size
osd pool get-quota <pool>               obtain object or byte limits for pool
osd pool ls [detail]                    list pools
osd pool mksnap <pool> <snap>           make snapshot <snap> in <pool>
osd pool rename <srcpool> <destpool>    rename <srcpool> to <destpool>
osd pool repair <who>...                initiate repair on pool <who>
osd pool rm <pool> [<pool2>] [--yes-i-really-really-mean-it] [--yes-i-really-really-mean-it-not-faking]  remove pool
osd pool rmsnap <pool> <snap>           remove snapshot <snap> from <pool>
osd pool scrub <who>...                 initiate scrub on pool <who>
osd pool set <pool> size|min_size|pg_   set pool parameter <var> to <val>
 num|pgp_num|pgp_num_actual|crush_rule|
 hashpspool|nodelete|nopgchange|
 nosizechange|write_fadvise_dontneed|
 noscrub|nodeep-scrub|hit_set_type|hit_
 set_period|hit_set_count|hit_set_fpp|  
 use_gmt_hitset|target_max_bytes|
 target_max_objects|cache_target_dirty_
 ratio|cache_target_dirty_high_ratio|
 cache_target_full_ratio|cache_min_
 flush_age|cache_min_evict_age|min_
 read_recency_for_promote|min_write_
 recency_for_promote|fast_read|hit_set_
 grade_decay_rate|hit_set_search_last_  
 n|scrub_min_interval|scrub_max_
 interval|deep_scrub_interval|recovery_
 priority|recovery_op_priority|scrub_
 priority|compression_mode|compression_
 algorithm|compression_required_ratio|  
 compression_max_blob_size|compression_
 min_blob_size|csum_type|csum_min_
 block|csum_max_block|allow_ec_
 overwrites|fingerprint_algorithm|pg_
 autoscale_mode|pg_autoscale_bias|pg_
 num_min|target_size_bytes|target_size_
 ratio|dedup_tier|dedup_chunk_
 algorithm|dedup_cdc_chunk_size <val>
 [--yes-i-really-mean-it]
osd pool set autoscale-profile scale-   set the autoscaler behavior to start
 down                                    out with full pgs and scales down when
                                         there is pressure
osd pool set autoscale-profile scale-up set the autoscaler behavior to start
                                         out with minimum pgs and scales up
                                         when there is pressure
osd pool set-quota <pool> max_objects|  set object or byte limit on pool
 max_bytes <val>
osd pool stats [<pool_name>]            obtain stats from all pools, or from
                                         specified pool
osd primary-affinity <id|osd.id>        adjust osd primary-affinity from 0.0 <=
 <weight:float>                          <weight> <= 1.0
osd primary-temp <pgid> <id|osd.id>     set primary_temp mapping pgid:<id>|-1 (
                                         developers only)
osd purge <id|osd.id> [--force] [--yes- purge all osd data from the monitors
 i-really-mean-it]                       including the OSD id and CRUSH position
osd purge-new <id|osd.id> [--yes-i-     purge all traces of an OSD that was
 really-mean-it]                         partially created but never started
osd repair <who>                        initiate repair on osd <who>, or use
                                         <all|any> to repair all
osd require-osd-release luminous|mimic| set the minimum allowed OSD release to
 nautilus|octopus|pacific [--yes-i-      participate in the cluster
 really-mean-it]
osd reweight <id|osd.id> <weight:float> reweight osd to 0.0 < <weight> < 1.0
osd reweight-by-pg [<oload:int>] [<max_ reweight OSDs by PG distribution
 change:float>] [<max_osds:int>]         [overload-percentage-for-consideration,
 [<pools>...]                             default 120]
osd reweight-by-utilization [<oload:    reweight OSDs by utilization [overload-
 int>] [<max_change:float>] [<max_osds:  percentage-for-consideration, default 1
 int>] [--no-increasing]                 20]
osd reweightn <weights>                 reweight osds with {<id>: <weight>,...}
osd rm-pg-upmap <pgid>                  clear pg_upmap mapping for <pgid> (
                                         developers only)
osd rm-pg-upmap-items <pgid>            clear pg_upmap_items mapping for <pgid>
                                         (developers only)
osd safe-to-destroy <ids>...            check whether osd(s) can be safely
                                         destroyed without reducing data
                                         durability
osd scrub <who>                         initiate scrub on osd <who>, or use
                                         <all|any> to scrub all
osd set full|pause|noup|nodown|noout|   set <key>
 noin|nobackfill|norebalance|norecover|
 noscrub|nodeep-scrub|notieragent|
 nosnaptrim|pglog_hardlimit [--yes-i-
 really-mean-it]
osd set-backfillfull-ratio <ratio:      set usage ratio at which OSDs are
 float>                                  marked too full to backfill
osd set-full-ratio <ratio:float>        set usage ratio at which OSDs are
                                         marked full
osd set-group <flags> <who>...          set <flags> for batch osds or crush
                                         nodes, <flags> must be a comma-
                                         separated subset of {noup,nodown,noin,
                                         noout}
osd set-nearfull-ratio <ratio:float>    set usage ratio at which OSDs are
                                         marked near-full
osd set-require-min-compat-client       set the minimum client version we will
 <version> [--yes-i-really-mean-it]      maintain compatibility with
osd setcrushmap [<prior_version:int>]   set crush map from input file
osd setmaxosd <newmax:int>              set new maximum osd value
osd stat                                print summary of OSD map
osd status [<bucket>]                   Show the status of OSDs within a bucket,
                                          or all
osd stop <ids>...                       stop the corresponding osd daemons and
                                         mark them as down
osd test-reweight-by-pg [<oload:int>]   dry run of reweight OSDs by PG
 [<max_change:float>] [<max_osds:int>]   distribution [overload-percentage-for-
 [<pools>...]                            consideration, default 120]
osd test-reweight-by-utilization        dry run of reweight OSDs by utilization
 [<oload:int>] [<max_change:float>]      [overload-percentage-for-consideration,
 [<max_osds:int>] [--no-increasing]       default 120]
osd tier add <pool> <tierpool> [--      add the tier <tierpool> (the second one)
 force-nonempty]                          to base pool <pool> (the first one)
osd tier add-cache <pool> <tierpool>    add a cache <tierpool> (the second one)
 <size:int>                              of size <size> to existing pool <pool>
                                         (the first one)
osd tier cache-mode <pool> writeback|   specify the caching mode for cache tier
 readproxy|readonly|none [--yes-i-       <pool>
 really-mean-it]
osd tier rm <pool> <tierpool>           remove the tier <tierpool> (the second
                                         one) from base pool <pool> (the first
                                         one)
osd tier rm-overlay <pool>              remove the overlay pool for base pool
                                         <pool>
osd tier set-overlay <pool>             set the overlay pool for base pool
 <overlaypool>                           <pool> to be <overlaypool>
osd tree [<epoch:int>] [up|down|in|out| print OSD tree
 destroyed...]
osd tree-from [<epoch:int>] <bucket>    print OSD tree in bucket
 [up|down|in|out|destroyed...]
osd unpause                             unpause osd
osd unset full|pause|noup|nodown|noout| unset <key>
 noin|nobackfill|norebalance|norecover|
 noscrub|nodeep-scrub|notieragent|
 nosnaptrim
osd unset-group <flags> <who>...        unset <flags> for batch osds or crush
                                         nodes, <flags> must be a comma-
                                         separated subset of {noup,nodown,noin,
                                         noout}
osd utilization                         get basic pg distribution stats
osd versions                            check running versions of OSDs
pg cancel-force-backfill <pgid>...      restore normal backfill priority of
                                         <pgid>
pg cancel-force-recovery <pgid>...      restore normal recovery priority of
                                         <pgid>
pg debug unfound_objects_exist|         show debug info about pgs
 degraded_pgs_exist
pg deep-scrub <pgid>                    start deep-scrub on <pgid>
pg dump [all|summary|sum|delta|pools|   show human-readable versions of pg map (
 osds|pgs|pgs_brief...]                  only 'all' valid with plain)
pg dump_json [all|summary|sum|pools|    show human-readable version of pg map
 osds|pgs...]                            in json only
pg dump_pools_json                      show pg pools info in json only
pg dump_stuck [inactive|unclean|stale|  show information about stuck pgs
 undersized|degraded...] [<threshold:
 int>]
pg force-backfill <pgid>...             force backfill of <pgid> first
pg force-recovery <pgid>...             force recovery of <pgid> first
pg getmap                               get binary pg map to -o/stdout
pg ls [<pool:int>] [<states>...]        list pg with specific pool, osd, state
pg ls-by-osd <id|osd.id> [<pool:int>]   list pg on osd [osd]
 [<states>...]
pg ls-by-pool <poolstr> [<states>...]   list pg with pool = [poolname]
pg ls-by-primary <id|osd.id> [<pool:    list pg with primary = [osd]
 int>] [<states>...]
pg map <pgid>                           show mapping of pg to osds
pg repair <pgid>                        start repair on <pgid>
pg repeer <pgid>                        force a PG to repeer
pg scrub <pgid>                         start scrub on <pgid>
pg stat                                 show placement group status.
progress                                Show progress of recovery operations
progress clear                          Reset progress tracking
progress json                           Show machine readable progress
                                         information
progress off                            Disable progress tracking
progress on                             Enable progress tracking
prometheus file_sd_config               Return file_sd compatible prometheus
                                         config for mgr cluster
quorum_status                           report status of monitor quorum
rbd mirror snapshot schedule add        Add rbd mirror snapshot schedule
 <level_spec> <interval> [<start_time>]
rbd mirror snapshot schedule list       List rbd mirror snapshot schedule
 [<level_spec>]
rbd mirror snapshot schedule remove  <level_spec> [<interval>] [<start_time>]    Remove rbd mirror snapshot schedule
rbd mirror snapshot schedule status [<level_spec>]    Show rbd mirror snapshot schedule status
rbd perf image counters [<pool_spec>]   Retrieve current RBD IO performance counters
 [write_ops|write_bytes|write_latency|   
 read_ops|read_bytes|read_latency]
rbd perf image stats [<pool_spec>]      Retrieve current RBD IO performance
 [write_ops|write_bytes|write_latency|   stats
 read_ops|read_bytes|read_latency]
rbd task add flatten <image_spec>       Flatten a cloned image asynchronously
                                         in the background
rbd task add migration abort <image_Abort a prepared migration
 spec>                                   asynchronously in the background
rbd task add migration commit <image_   Commit an executed migration
 spec>                                   asynchronously in the background
rbd task add migration execute <image_Execute an image migration
 spec>                                   asynchronously in the background
rbd task add remove <image_spec>        Remove an image asynchronously in the
                                         background
rbd task add trash remove <image_id_    Remove an image from the trash
 spec>                                   asynchronously in the background
rbd task cancel <task_id>               Cancel a pending or running
                                         asynchronous task
rbd task list [<task_id>]               List pending or running asynchronous
                                         tasks
rbd trash purge schedule add <level_    Add rbd trash purge schedule
 spec> <interval> [<start_time>]
rbd trash purge schedule list [<level_List rbd trash purge schedule
 spec>]
rbd trash purge schedule remove <level_ Remove rbd trash purge schedule
 spec> [<interval>] [<start_time>]
rbd trash purge schedule status         Show rbd trash purge schedule status
 [<level_spec>]
report [<tags>...]                      report full status of cluster, optional
                                         title tag strings
restful create-key <key_name>           Create an API key with this name
restful create-self-signed-cert         Create localized self signed certificate
restful delete-key <key_name>           Delete an API key with this name
restful list-keys                       List all API keys
restful restart                         Restart API server
service dump                            dump service map
service status                          dump service state
status                                  show cluster status
telegraf config-set <key> <value>       Set a configuration value
telegraf config-show                    Show current configuration
telegraf send                           Force sending data to Telegraf
telemetry off                           Disable telemetry reports from this
                                         cluster
telemetry on [<license>]                Enable telemetry reports from this
                                         cluster
telemetry send [ceph|device...]         Force sending data to Ceph telemetry
 [<license>]
telemetry show [<channels>...]          Show last report or report to be sent
telemetry show-all                      Show report of all channels
telemetry show-device                   Show last device report or device
                                         report to be sent
telemetry status                        Show current configuration
tell <type.id> <args>...                send a command to a specific daemon
test_orchestrator load_data             load dummy data into test orchestrator
time-sync-status                        show time sync status
versions                                check running versions of ceph daemons
zabbix config-set <key> <value>         Set a configuration value
zabbix config-show                      Show current configuration
zabbix discovery                        Discovering Zabbix data
zabbix send                             Force sending data to Zabbix
```

## rbd 命令

```sh
usage: rbd <command> ...

Command-line interface for managing Ceph RBD images.
```

Positional arguments: 
|   command                                   |          reference
| ---                                         | --- |
| bench                                       | Simple benchmark.
| children                                    | Display children of an image or its snapshot.
| clone                                       | Clone a snapshot into a CoW child image.
|  |
| config global get                           | Get a global-level configuration override.
| config global list (... ls)                 | List global-level configuration overrides.
| config global remove (... rm)               | Remove a global-level configuration override.
| config global set                           | Set a global-level configuration override.
| config image get                            | Get an image-level configuration override.
| config image list (... ls)                  | List image-level configuration overrides.
| config image remove (... rm)                | Remove an image-level configuration override.
| config image set                            | Set an image-level configuration override.
| config pool get                             | Get a pool-level configuration override.
| config pool list (... ls)                   | List pool-level configuration overrides.
| config pool remove (... rm)                 | Remove a pool-level configuration override.
| config pool set                             | Set a pool-level configuration override.
|   |
| copy (cp)                                   | Copy src image to dest.
| create                                      | Create an empty image.
| deep copy (deep cp)                         | Deep copy src image to dest.
| device list (showmapped)                    | List mapped rbd images.
| device map (map)                            | Map an image to a block device.
| device unmap (unmap)                        | Unmap a rbd device.
| diff                                        | Print extents that differ since a previous snap, or image creation.
| disk-usage (du)                             | Show disk usage stats for pool, image or snapshot.
| encryption format                           | Format image to an encrypted format.
| export                                      | Export image to file.
| export-diff                                 | Export incremental diff to file.
| feature disable                             | Disable the specified image feature.
| feature enable                              | Enable the specified image feature.
| flatten                                     | Fill clone with parent data (make it independent).
|  |
| group create                                | Create a group.
| group image add                             | Add an image to a group.
| group image list (... ls)                   | List images in a group.
| group image remove (... rm)                 | Remove an image from a group.
| group list (group ls)                       | List rbd groups.
| group remove (group rm)                     | Delete a group.
| group rename                                | Rename a group within pool.
| group snap create                           | Make a snapshot of a group.
| group snap list (... ls)                    | List snapshots of a group.
| group snap remove (... rm)                  | Remove a snapshot from a group.
| group snap rename                           | Rename group's snapshot.
| group snap rollback                         | Rollback group to snapshot.
|  |
| image-cache invalidate                      | Discard existing / dirty image cache
| image-meta get                              | Image metadata get the value associated with the key.
| image-meta list (image-meta ls)             | Image metadata list keys with values.
| image-meta remove (image-meta rm)           | Image metadata remove the key and value associated.
| image-meta set                              | Image metadata set key with value.
|  |
| import                                      | Import image from file.
| import-diff                                 | Import an incremental diff.
| info                                        | Show information about image size,striping, etc.
|  |
| journal client disconnect                   | Flag image journal client as disconnected.
| journal export                              | Export image journal.
| journal import                              | Import image journal.
| journal info                                | Show information about image journal.
| journal inspect                             | Inspect image journal for structural errors.
| journal reset                               | Reset image journal.
| journal status                              | Show status of image journal.
|  |
| list (ls)                                   | List rbd images.
| lock add                                    | Take a lock on an image.
| lock list (lock ls)                         | Show locks held on an image.
| lock remove (lock rm)                       | Release a lock on an image.
| merge-diff                                  | Merge two diff exports together.
|  |
| migration abort                             | Cancel interrupted image migration.
| migration commit                            | Commit image migration.
| migration execute                           | Execute image migration.
| migration prepare                           | Prepare image migration.
|  |
| mirror image demote                         | Demote an image to non-primary for RBD mirroring.
| mirror image disable                        | Disable RBD mirroring for an image.
| mirror image enable                         | Enable RBD mirroring for an image.
| mirror image promote                        | Promote an image to primary for RBD mirroring.
| mirror image resync                         | Force resync to primary image for RBD mirroring.
| mirror image snapshot                       | Create RBD mirroring image snapshot.
| mirror image status                         | Show RBD mirroring status for an image.
| mirror pool demote                          | Demote all primary images in the pool.
| mirror pool disable                         | Disable RBD mirroring by default within a pool.
| mirror pool enable                          | Enable RBD mirroring by default within a pool.
| mirror pool info                            | Show information about the pool mirroring configuration.
| mirror pool peer add                        | Add a mirroring peer to a pool.
| mirror pool peer bootstrap create           | Create a peer bootstrap token to import in a remote cluster
| mirror pool peer bootstrap import           | Import a peer bootstrap token created from a remote cluster
| mirror pool peer remove                     | Remove a mirroring peer from a pool.
| mirror pool peer set                        | Update mirroring peer settings.
| mirror pool promote                         | Promote all non-primary images in the pool.
| mirror pool status                          | Show status for all mirrored images in the pool.
| mirror snapshot schedule add                | Add mirror snapshot schedule.
| mirror snapshot schedule list (... ls)      | List mirror snapshot schedule.
| mirror snapshot schedule remove (... rm)    | Remove mirror snapshot schedule.
| mirror snapshot schedule status             | Show mirror snapshot schedule status.
|  |
| namespace create                            | Create an RBD image namespace.
| namespace list (namespace ls)               | List RBD image namespaces.
| namespace remove (namespace rm)             | Remove an RBD image namespace.
|  |
| object-map check                            | Verify the object map is correct.
| object-map rebuild                          | Rebuild an invalid object map.
| perf image iostat                           | Display image IO statistics.
| perf image iotop                            | Display a top-like IO monitor.
| pool init                                   | Initialize pool for use by RBD.
| pool stats                                  | Display pool statistics.
| remove (rm)                                 | Delete an image.
| rename (mv)                                 | Rename image within pool.
| resize                                      | Resize (expand or shrink) image.
| |
| snap create (snap add)                      | Create a snapshot.
| snap limit clear                            | Remove snapshot limit.
| snap limit set                              | Limit the number of snapshots.
| snap list (snap ls)                         | Dump list of image snapshots.
| snap protect                                | Prevent a snapshot from being deleted.
| snap purge                                  | Delete all unprotected snapshots.
| snap remove (snap rm)                       | Delete a snapshot.
| snap rename                                 | Rename a snapshot.
| snap rollback (snap revert)                 | Rollback image to snapshot.
| snap unprotect                              | Allow a snapshot to be deleted.
| sparsify                                    | Reclaim space for zeroed image extents.
| status                                      | Show the status of this image.
| |
| trash list (trash ls)                       | List trash images.
| trash move (trash mv)                       | Move an image to the trash.
| trash purge                                 | Remove all expired images from trash.
| trash purge schedule add                    | Add trash purge schedule.
| trash purge schedule list (... ls)          | List trash purge schedule.
| trash purge schedule remove (... rm)        | Remove trash purge schedule.
| trash purge schedule status                 | Show trash purge schedule status.
| trash remove (trash rm)                     | Remove an image from trash.
| trash restore                               | Restore an image from trash.
| watch                                       | Watch events on image.
| Optional arguments:                         |
|  -c [ --conf ] arg                          | path to cluster configuration
|  --cluster arg                              | cluster name
|  --id arg                                   | client id (without 'client.' prefix)
|  -n [ --name ] arg                          | client name
|  -m [ --mon_host ] arg                      | monitor host
|  -K [ --keyfile ] arg                       | path to secret key
|  -k [ --keyring ] arg                       | path to keyring
