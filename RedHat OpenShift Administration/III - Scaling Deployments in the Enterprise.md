## LDAP Authentication

### Objectives

- Configure an LDAP identity provider.
    

### The LDAP Service

After the installation of a new Red Hat OpenShift Container Platform (RHOCP) cluster, only a `kubeadmin` user exists. A cluster administrator must next create users and groups, or more commonly, integrate a business system that both provides these entities and manages authentication. In doing so, an OpenShift cluster adopts a consistent corporate network authentication approach with other business systems, and inherits the existing security implementations in place, such as the company access restrictions that are available through a VPN-based connection.

Integrating an existing authentication method, and available users and groups in your organization, by using a Lightweight Directory Access Protocol (LDAP) server, is a common approach for many businesses. An LDAP directory server defines a standard application protocol for querying the system to authenticate users and groups. Additionally, LDAP integrations enable the use of Red Hat Identity Management (IdM) and Microsoft Active Directory (AD) services that many organizations use for managing authentication.

An LDAP server can be configured as an Identity Provider (IdP) for authentication bindings between the LDAP identity entries and cluster users through the OpenShift OAuth server. When users enter credentials for the cluster, the OAuth server initiates a connection and a corresponding query to the LDAP server, and creates bindings when matching unique entries are found.

Although this implementation is sufficient for authenticating the user, it is still necessary to define the access for the user, based on group memberships and roles. Furthermore, configuration within the OpenShift cluster is required to synchronize groups from the LDAP database, as well as to define the appropriate access and permissions for each group.

>Note
This section does not cover LDAP administration in depth, but only the necessary information for configuring LDAP integrations in an RHOCP cluster.

### Authentication by Using LDAP

The LDAP protocol defines a query language to retrieve information from database entries of a remote LDAP server, which can contain entries for users and groups within the organization. When implemented as an OpenShift OAuth IdP, the cluster can rely on the LDAP server to validate user-provided login credentials for the cluster.

An LDAP URI exists and uses a standard syntax for connection parameters to an LDAP server, and is necessary when configuring client connections to an LDAP server. To perform queries, the URI includes the search criteria to match with entries in the LDAP database. Many LDAP servers also require the use of an administrative `bind DN`, which is a set of privileged credentials with access to perform queries. This set of privileged credentials is included in the URI, when implemented.

A correctly formed search string describes the LDAP endpoint and the specific set of credentials, or a set of search criteria, such as user or group names, along with the bind Distinguished Name (bindDN) credentials when required for the interaction.

An LDAP URI is formed with the following standardized syntax:

ldap://host:port/basedn?attribute?scope?filter

`ldap`

The LDAP protocol designation. For LDAP over SSL, use the `ldaps` protocol. Red Hat recommends using StartTLS over regular `ldap`.

`host:port`

The LDAP server name and listening port. The default for `ldap` is `localhost:389`, and for `ldaps` is `localhost:636`.

`basedn`

The DN of the directory where a query begins. This directory is the root location for the search.

`attributes`

The target for the search, with the default as `uid`, which the LDAP lookup is intended to return. Multiple attributes are provided in a comma-separated list.

`scope`

The scope of the LDAP lookup. The scope is either `one` or `sub`. The default is `sub` if unspecified.

`filter`

An LDAP filter refines the results for the query. If omitted, the filter defaults to `(objectClass=*)`.

The following table provides sample LDAP lookup information for a specific username:

|**URL object**|**Value**|
|:--|:--|
|Connection protocol|`ldap`|
|Server name and port|`ldap.example.com:389`|
|Base DN|`dc=example,dc=com`|
|Attributes|`givenName,sn,cn`|
|Query filter|`uid=paula.tomcheck`|

The preceding information would result in the following URL for an LDAP lookup:

ldap://ldap.example.com:389/dc=example,dc=com?givenName,sn,cn?(uid=paula.tomcheck)

The result of this lookup returns one or more usernames from directory entries that match the given `uid` value in the LDAP server.

In an OpenShift cluster, when a new user provides login credentials for the cluster, the OAuth LDAP IdP searches for the identity by using the configured LDAP search account. This LDAP search account is configured during the OAuth IdP configuration, and contains a Bind Distinguished Name (Bind DN) and a Bind Password. On a unique match from the search, the provided credentials are submitted to the LDAP server in a second interaction, and an authentication binding is created between an OpenShift user resource and the returned LDAP `id` and `preferredUsername` values.

If an authentication request has no matching result, then the query fails and no bind occurs.

Whereas the LDAP server provides the validation to create an authentication binding for the identity, a cluster administrator must also grant appropriate cluster roles for the identity through the OpenShift Role Based Access Controls (RBAC) settings. A new user configuration is complete when the binding and role are in place.

An administrator must synchronize groups from an LDAP server as an additional cluster-side configuration before a full LDAP IdP configuration is complete. It is necessary to create a cluster cron job to routinely query the LDAP server and to update the OpenShift groups to ensure consistency within the cluster to any organizational updates to the LDAP group entries.

> Note
Configuring and running the LDAP group synchronization are covered in the next section.

### LDAP CLI Queries

Although this course does not provide guidelines for LDAP administration, a tool is needed to inspect and validate available LDAP connectivity during cluster configuration.

The `ldapsearch` command is available from the `openldap-clients` RPM package, and provides a command-line query utility for LDAP interactions. The following example illustrates the format for an LDAP query that uses the `ldapsearch` command:

[user@host ~]$ **``ldapsearch -x -D _`bind_dn`_ -H _`ldap_host_URI`_ -b _`search_base`_ \   --filter _`filter`_ --requestedAttribute _`attributes`_ -W``**

In this example, the following parameters are used:

- The `-x` option specifies using simple authentication.
    
- The `-W` option prompts you to provide the `bindPassword` secret for the specific ``-D _`bind_dn`_`` option.
    
- The ``-b _`search_base`_`` option denotes the root location for the lookup.
    
- The `--filter` and `--requestedAttribute` options provide further criteria to refine the search.
    

This approach provides a method for testing credentials for the LDAP server from the CLI, as well as retrieving listings for various LDAP entries, such as any defined groups. Although the previous example is too broad to provide meaningful results or detailed information, this approach establishes the validity of the credentials. Specifying additional attributes and filters returns more specific information, such as refining the query to search for an account with `uid=1001`, as shown in the following example:
```
[user@host ~]$ **`ldapsearch -x -D "cn=administrator,dc=example,dc=com" \    -H ldap://ldap.example.com -b "uid=1001" -W "objectclass=account" uid`**
Enter Password:
# extended LDIF
#

# LDAPv3
# base <uid=1001> (default) with scope subtree
# filter: (objectclass=account)
# requesting: ALL

# example.com
dn: dc=example,dc=com

_...output omitted..._
```

The options for `ldapsearch` queries, and their resulting output, are consistent with the needed information for the configuration of an LDAP server as an OAuth IdP.

The organizational units that describe the path to the correct location for an information lookup within the LDAP system are similar to the structure of directories and files within a file system.

The following example DN shows the `uid`, `ou`, and `dc` values for information within the LDAP server:

dn: uid=ptomchek,ou=users,dc=ocp4,dc=redhat,dc=com

The following commands illustrate a corresponding organizational structure for the same information within a file system:

```
[user@host ~]$ mkdir -p ocp4.redhat.com/users
[user@host ~]$ touch ocp4.redhat.com/users/ptomchek
[user@host ~]$ tree ocp4.redhat.com/
ocp4.redhat.com/
  └── users
    └── ptomchek
```

>Note
For more information about LDAP administration and the `ldapsearch` command, refer to the Red Hat Directory Server documentation in the references section.

### Configuring an LDAP Server as an OpenShift IdP

Integrating an LDAP server as an IdP for an OpenShift cluster requires the following process:

- Create an OpenShift secret resource to store the password for LDAP queries. This OpenShift secret resource contains the Base64-encoded secret in the `bindPassword` field.
    
- Create an OpenShift configuration map resource that contains the certificate authority bundle (if the LDAP server uses encryption).
    

>Note
Create this configuration map resource only if your LDAP IdP has a CA certificate that is not present in the default system keystore.

This OpenShift `ConfigMap` resource belongs in the `openshift-config` namespace.

- Update the CR to add the LDAP configuration to the existing OpenShift OAuth IdP entries.
    
- Redeploy the IdP CR to the cluster with the added LDAP IdP.
    

#### Create the `bindPassword` Secret Resource

The `bindPassword` for the IdP configuration is stored in an OpenShift secret resource. The following example is a YAML file for creating the `bindPassword` secret for an LDAP IdP:

```
apiVersion: v1
kind: Secret
metadata:
  name: ldap-secret
  namespace: openshift-config
type: Opaque
data:
  bindPassword: _`base64-encoded-bind-password
```

>Note
Consider that cluster administrators can read the password that is stored in the secret. Red Hat recommends using a service account with limited permissions as the bindDN within LDAP to avoid any security breaches.

#### Create the Certificate Configuration Map

LDAP configurations for OAuth IdPs use an OpenShift configuration map resource in the `openshift-config` namespace to define the certificate authority bundles.

Create an OpenShift configuration map resource that contains the certificate authority. You must store the certificate authority in the `ca.crt` key of the configuration map resource.

The following example is a YAML file that defines how to create a configuration map resource for the certificate bundle:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: ca-config-map
  namespace: openshift-config
data:
  ca.crt: |
    _`CA_certificate_PEM`_
```

#### Update the OAuth CR

The following example shows the configuration for defining an LDAP IdP in the OAuth CR YAML file:

```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: `ldapidp` ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    mappingMethod: `claim` ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
    type: LDAP
    ldap:
      attributes:
        id: ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
        - `dn`
        email: ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
        - `mail`
        name: ![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
        - `cn`
        preferredUsername: ![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)
        - `uid`
      bindDN: `'cn=administrator,dc=example,dc=com'` ![7](https://rol.redhat.com/rol/static/roc/Common_Content/images/7.svg)
      bindPassword: ![8](https://rol.redhat.com/rol/static/roc/Common_Content/images/8.svg)
        name: `ldap-secret`
      `ca`: ![9](https://rol.redhat.com/rol/static/roc/Common_Content/images/9.svg)
        name: ca-config-map
      insecure: `false` ![10](https://rol.redhat.com/rol/static/roc/Common_Content/images/10.svg)
      url: "`ldaps://ldaps.example.com/ou=users,dc=acme,dc=com?uid`" ![11](https://rol.redhat.com/rol/static/roc/Common_Content/images/11.svg)
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-1)|Provider names are shown on the web console login screen and in the list of configured IdPs.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-2)|Controls how mappings are established between this provider's identities and user objects.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-3)|List of attributes where the first non-empty attribute is used. At least one attribute is required. If none of the listed attributes has a value, then authentication fails.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-4)|List of attributes to use as the email address. The first non-empty attribute is used.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-5)|List of attributes to use as the display name. The first non-empty attribute is used.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-6)|List of attributes to use as the preferred username when provisioning a user for this identity. The first non-empty attribute is used.|
|[![7](https://rol.redhat.com/rol/static/roc/Common_Content/images/7.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-7)|Required DN that is used during the search phase, unless anonymous searches are allowed. Must be set if the `bindPassword` parameter is defined.|
|[![8](https://rol.redhat.com/rol/static/roc/Common_Content/images/8.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-8)|Required OpenShift secret resource that contains the bind password, unless anonymous searches are allowed. Must be set if the `bindDN` parameter is defined.|
|[![9](https://rol.redhat.com/rol/static/roc/Common_Content/images/9.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-9)|Optional: Reference to an OpenShift configuration map resource that contains the privacy enhanced mail (PEM) encoded certificate authority bundle to validate server certificates for the configured URL. Used only when the insecure option is false.|
|[![10](https://rol.redhat.com/rol/static/roc/Common_Content/images/10.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-10)|When true, no TLS connection is made to the server. When false, `ldaps://` URLs connect by using TLS, and `ldap://` URLs are upgraded to TLS. The `insecure` option must be set to false when `ldaps://` URLs are in use, because these URLs always attempt to connect by using TLS.|
|[![11](https://rol.redhat.com/rol/static/roc/Common_Content/images/11.svg)](https://rol.redhat.com/rol/app/#_update_the_oauth_cr-CO5-11)|An RFC 2255 URL, which specifies the LDAP host and maps the identity parameters in the LDAP database schema. This `url` value associates the user-supplied login credential with the specific entries in the LDAP server for identity lookups.|

>Note
The OAuth CR contains all configured IdPs. Any additional IdP that is configured is appended to the CR. Replacing or removing any entries can result in the loss of access to your cluster.

### Log in by Using the Added LDAP IdP

After you update the OAuth CR, the new LDAP IdP is available to the cluster. The IdP `Name` is shown among any other configured IdPs on the web console login page, as well as in the list of configured OAuth IdPs.

Log in with this added LDAP IdP by using the `oc login` command or the added selection from the OpenShift web console, and by providing a username and password that are available through the LDAP server. During authentication, OpenShift generates a search filter by combining the attribute and filter in the configured OAuth CR `url` parameter, with the provided username. Then, OpenShift applies this filter to the LDAP directory to find a unique entry.

From the preceding configuration example, if the user enters `paula.tomcheck` as a username, then a query for that value is sent to the LDAP server. This query attempts to find a unique match in the `uid` LDAP field that the OAuth CR `url` value `"ldaps://ldaps.example.com/ou=users,dc=acme,dc=com?uid"` specifies.

If you configure the `mappingMethod` parameter for the LDAP IdP entry in the OAuth CR as `claim` or `add`, then OpenShift uses the resulting LDAP identity to create a cluster `User` resource on the first attempt to access the cluster.

Authentication fails if the LDAP lookup does not return a unique match. In this case, no binding is created, no user is added to the cluster, and access is denied.

## LDAP Group Synchronization

### Objectives

- Automate group synchronization between OpenShift OAuth and an LDAP server.
    

### OpenShift Group Synchronization with LDAP

Configuring an OAuth IdP for an LDAP server provides only a mechanism to validate and create users in the cluster, based on entries that are provided through the LDAP instance. Businesses also rely on LDAP to provide the groups that are used throughout the organization and to define appropriate RBAC configurations. By configuring LDAP group synchronization, a business can manage group memberships in a single location, and OpenShift can use these groups within the cluster.

OpenShift can synchronize groups from an LDAP server, which enables you to mirror the available LDAP groups within the cluster. LDAP group synchronization requires a sync configuration file with a client configuration and a query definition. Additionally, user-defined mappings can optionally provide granular details for establishing the custom relationships between group entries in the LDAP server and the corresponding OpenShift groups that you require.

Group synchronization requires a project where the pods run, a service account to perform the work, a cluster role where the right permissions are defined, and cluster role binding to associate the cluster role with the service account. When the synchronization configuration is in place, the synchronization of groups is automated by using a cron job to schedule the recurring execution of the synchronization task.

#### LDAP Sync Configuration File

To synchronize LDAP groups, create an `LDAPSyncConfig` resource that contains the client LDAP configuration details and query definition for the LDAP group entities that are needed within OpenShift.

The LDAP client configuration part defines the connection parameters for the LDAP server, as well as the `bindDN` account and `bindPassword` secret for performing queries. This configuration uses the same connection and credentials that are used during the OAuth LDAP IdP implementation.

The following YAML code is an example of LDAP client configuration details:

```yaml
url: ldap://1.2.3.4:389 #![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
bindDN: cn=administrator,dc=example,dc=com #![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
bindPassword: `password` #![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
insecure: false #![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
ca: /path/to/ca-bundle.crt #![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO6-1)|The LDAP server URL.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO6-2) [![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO6-3)|The account and password of the LDAP user that OpenShift uses to retrieve group information from the LDAP server.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO6-4)|When true, no TLS connection is made to the server. When false, `ldaps://` URLs connect by using TLS, and `ldap://` URLs are upgraded to TLS. The `insecure` option must be set to false when `ldaps://` URLs are in use, because these URLs always attempt to connect by using TLS.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO6-5)|OpenShift uses the platform trusted CA bundle to validate the LDAP server certificate by default. If this CA bundle does not trust the LDAP server certificate, then you must provide the path to the certificate authority bundle in the `ca` parameter.|

The LDAP query definition part describes the format of entries within the LDAP server that provide the groups to mirror to the OpenShift cluster.

The following YAML code is an example of an LDAP query definition:

```yaml
baseDN: ou=groups,dc=example,dc=com #![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
scope: sub #![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
derefAliases: never #![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
timeout: 0 #![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
filter: (objectClass=group) #![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
pageSize: 0 #![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO7-1)|The base DN of the LDAP server where the groups are located.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO7-2)|The scope of the search. The `sub` scope means that the entire subtree of the base DN is searched.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO7-3)|The alias dereferencing mode. The `never` mode means that aliases are not followed for faster search results.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO7-4)|The timeout for the search. If set to `0`, then no timeout is applied.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO7-5)|The filter for the search. When not specified, the search returns all entries. In this example, the search returns only entries where the `objectClass` attribute is set to `group`.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO7-6)|The number of entries to return per response from the LDAP server. If set to `0`, then the LDAP server returns all entries.|

By assembling the client configuration details together with the query definition, you can create an `LDAPSyncConfig` resource by using the parameters for your LDAP server:

```yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://example.com:389
insecure: false
rfc2307:
    groupsQuery:
        baseDN: "ou=groups,dc=example,dc=com"
        scope: sub
        derefAliases: never
        pageSize: 0
    groupUIDAttribute: dn #![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    groupNameAttributes: [ cn ] #![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
    groupMembershipAttributes: [ member ]
    usersQuery:
        baseDN: "ou=users,dc=example,dc=com"
        scope: sub
        derefAliases: never
        pageSize: 0
    userUIDAttribute: dn #![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
    userNameAttributes: [ mail ] #![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
    tolerateMemberNotFoundErrors: false
    tolerateMemberOutOfScopeErrors: false
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO8-1)|The field that corresponds to the unique identifier field on the LDAP server for a group.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO8-2)|The attribute for the name of the group.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO8-3)|The field that corresponds to the unique identifier field of the LDAP server for a user.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_ldap_sync_configuration_file-CO8-4)|The attribute for the name of the user.|

>Note
For more information about the LDAPSyncConfig resource and its parameters, refer to [https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/authentication_and_authorization/index#ldap-syncing](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/authentication_and_authorization/index#ldap-syncing)

#### Example Active Directory Configuration

Active Directory (AD) is another source for providing group definitions for an OpenShift cluster. The AD configuration requires an LDAP query definition and attributes to represent users within the OpenShift group definitions.

The following YAML code is an example LDAP synchronization configuration for an AD schema:

```yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://ad.example.com:389
activeDirectory:
    usersQuery:
        baseDN: "ou=users,dc=example,dc=com"
        scope: sub
        derefAliases: never
        filter: (objectclass=person)
        pageSize: 0
    userNameAttributes: [ mail ] #![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    groupMembershipAttributes: [ memberOf ] #![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_example_active_directory_configuration-CO9-1)|The attribute to use as the username in the OpenShift group record.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_example_active_directory_configuration-CO9-2)|The attribute on the user that stores the group membership information.|

Note the schema that AD uses to map users to the corresponding group membership through the `memberOf` attribute.

### Running an LDAP Synchronization

After the `LDAPSyncConfig` file is created, an administrator can use the file to synchronize groups from the LDAP server to the OpenShift cluster. The following example command uses the `oc adm groups sync` command to synchronize all groups from the LDAP server to the cluster by using the `config.yaml` file that contains the `LDAPSyncConfig` details:

[user@host ~]$ **`oc adm groups sync --sync-config=config.yaml --confirm`**

If the `--confirm` parameter is omitted, then the command performs a dry run of the operation. An initial dry run is advised when testing a new configuration, and to review the resulting groups that the cluster will inherit.

### Automating LDAP Group Synchronizations

LDAP group synchronizations are automated through the use of a cron job. Configure this cron job from within a designated OpenShift project for this purpose. Execute the job by using a service account with administrative access to the cluster to manage groups.

Start by creating the project for the cron job to run. The following command creates a project named `ldap-group-sync` for this purpose:

[user@host ~]$ **`oc new-project ldap-group-sync`**

The client `bindPassword` secret resource and the configuration map resource that are used during the configuration of the LDAP IdP are also needed for this task. Additionally, create a service account with an appropriate cluster role and cluster role binding for the synchronization. The following YAML code shows an example definition for creating a service account named `ldap-group-sync-acct`:

kind: ServiceAccount
apiVersion: v1
metadata:
  name: ldap-group-sync-acct
  namespace: ldap-group-sync

Next, define a cluster role, as shown in the following example:

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ldap-group-sync-acct
rules:
  - apiGroups:
      - ''  ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
      - user.openshift.io
    resources:
      - groups
    verbs:
      - get
      - list
      - create
      - update

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_automating_ldap_group_synchronizations-CO10-1)|The core API group is identified by an empty string.|

Then, define a cluster role binding to bind the previous cluster role to the existing service account.

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ldap-group-sync-acct
subjects:
  - kind: ServiceAccount
    name: ldap-group-sync-acct
    namespace: ldap-group-sync
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ldap-group-sync-acct

The cluster is configured to communicate with the LDAP IdP, and a service account is available to configure OpenShift groups from the LDAP group entities. The following YAML code is an example configuration map file to specify the sync configuration:

kind: ConfigMap
apiVersion: v1
metadata:
  name: ldap-group-sync-acct
  namespace: ldap-group-sync
data:
  sync.yaml: | ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    kind: LDAPSyncConfig
    apiVersion: v1
    url: ldaps://1.2.3.4:389
    insecure: false
    bindDN: cn=administrator,dc=example,dc=com
    bindPassword:
      file: "/etc/secrets/bindPassword"
    ca: /etc/ldap-ca/ca.crt
    rfc2307:
      groupsQuery:
        baseDN: "ou=groups,dc=example,dc=com"
        scope: sub
        filter: "(objectClass=groupOfMembers)"
        derefAliases: never
        pageSize: 0
      groupUIDAttribute: dn
      groupNameAttributes: [ cn ]
      groupMembershipAttributes: [ member ]
      usersQuery:
        baseDN: "ou=users,dc=example,dc=com"
        scope: sub
        derefAliases: never
        pageSize: 0
      userUIDAttribute: dn
      userNameAttributes: [ uid ]
      tolerateMemberNotFoundErrors: false
      tolerateMemberOutOfScopeErrors: false

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_automating_ldap_group_synchronizations-CO11-1)|The YAML file contains the `LDAPSyncConfig` resource that you use with the `oc adm groups sync` command.|

Finally, define and deploy a cron job for the LDAP group synchronization. The following YAML code is an example cron job definition that could be customized, such as by adapting the cron-specified schedule to meet business needs:

```yaml
kind: CronJob
apiVersion: batch/v1
metadata:
  name: ldap-group-sync-acct
  namespace: ldap-group-sync
spec:
  schedule: "​*/30 * * * *​" #![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      backoffLimit: 0
      ttlSecondsAfterFinished: 1800 #![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
      template:
        spec:
          containers:
            - name: ldap-group-sync
              image: "registry.redhat.io/openshift4/ose-cli:latest"
              command:
                - "/bin/bash"
                - "-c"
                - "oc adm groups sync --sync-config=/etc/config/sync.yaml --confirm" #![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
              volumeMounts:
                - mountPath: "/etc/config"
                  name: "ldap-sync-volume"
                - mountPath: "/etc/secrets"
                  name: "ldap-bind-password"
                - mountPath: "/etc/ldap-ca"
                  name: "ldap-ca"
          volumes:
            - name: "ldap-sync-volume"
              configMap:
                name: "ldap-group-sync-acct"
            - name: "ldap-bind-password"
              secret:
                secretName: "ldap-secret" #![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
            - name: "ldap-ca"
              configMap:
                name: "ca-config-map"
          restartPolicy: "Never"
          terminationGracePeriodSeconds: 30
          activeDeadlineSeconds: 500
          dnsPolicy: "ClusterFirst"
          serviceAccountName: "ldap-group-sync-acct"
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_automating_ldap_group_synchronizations-CO12-1)|The defined schedule for the job in `cron` format, to show every 30 minutes.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_automating_ldap_group_synchronizations-CO12-2)|The time, in seconds, to keep completed sync information, to correspond to the preceding cron schedule.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_automating_ldap_group_synchronizations-CO12-3)|The LDAP group sync command to execute by using the sync configuration file that is defined in the configuration map. The `oc adm groups sync` command is defined on a single line.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_automating_ldap_group_synchronizations-CO12-4)|The LDAP IdP `bindDN` and `bindPassword`.|

After the file is created, the configured cron job executes the group synchronization command during the specified schedule to ensure that consistent group configurations that are available from the LDAP IdP are added to the OpenShift cluster. This job runs within the `project` namespace, by using the service account, at an interval that the cron time spec defines in the `CronJob` definition.

## OIDC Authentication and Group Claims

### Objectives

- Configure an OIDC identity provider and automate group synchronization between OpenShift OAuth and an OIDC server.

### OpenID Connect

You can use OpenShift to configure OpenID Connect (OIDC) identity providers (IdPs) for synchronizing users and groups.

OIDC is a set of standards for delegating the authentication of a user who accesses a protected resource. OIDC provides a way for applications to verify the identity of users and to obtain user profile information.

OIDC is the standard for cloud authentication, social authentication, single sign-on (SSO), and two-factor authentication (2FA). OIDC removes the responsibility of setting, storing, and managing passwords locally. This approach reduces credential-based data breaches and requires fewer administrative resources.

Examples of OIDC providers that are tested and supported on OpenShift include Google, Microsoft identity platform, and Keycloak. Red Hat provides Red Hat build of Keycloak to extend the capabilities of the OpenShift internal OAuth, and to serve as a solution for an OIDC identity infrastructure. Red Hat build of Keycloak replaces Red Hat SSO. Red Hat SSO is in the Maintenance Support phase. Red Hat build of Keycloak can run on bare-metal or virtualized environments, or as pods on OpenShift.

>Note
For more information about Red Hat SSO, refer to the _Product Documentation for Red Hat Single Sign-On 7.6_ at [https://docs.redhat.com/en/documentation/red_hat_single_sign-on/7.6](https://docs.redhat.com/en/documentation/red_hat_single_sign-on/7.6)
For more information about the Red Hat build of Keycloak, refer to the _Product Documentation for Red Hat build of Keycloak_ at [https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/)
To see the full list of supported OIDC providers on OpenShift, refer to the _Supported Identity Providers_ section in the Red Hat OpenShift Container Platform 4.18 _Authentication and Authorization_ documentation at [https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/authentication_and_authorization/index#identity-provider-oidc-supported_configuring-oidc-identity-provider](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/authentication_and_authorization/index#identity-provider-oidc-supported_configuring-oidc-identity-provider)

#### OIDC Tokens

OIDC relies on the JavaScript Object Signing and Encryption (JOSE) set of standards. The primary standard within JOSE is JSON Web Token (JWT), which serves as a standardized format for representing information in a readable piece of text data. The following list shows some JWT advantages:

- Can parse in many programming languages
- Can propagate across networks
- Validates message integrity without relying on external validators or resource-intensive checks

Three parts, which are separated by a period, form the JWT token: the header, the payload, and the hash signature.

The header contains the type of the token and the signing algorithm.

The payload contains the user claims. User claims are user attributes with details about their identity, profile, privileges, and group membership. Examples of user claims are the `sub`, `iat`, `exp`, or `iss` parameters, which are the identity for the user, the issue date, the expiration date, and the token issuer, respectively.

Finally, the signature is composed of the concatenation of the encoded header, the encoded payload, and the result of applying a signing algorithm.

#### OIDC in OpenShift

An _identity broker_ is a service that connects clients with IdPs. The identity broker delegates the authentication of the user to the external IdP. OpenShift includes a built-in OAuth server that you can configure to determine the user's identity from the configured IdPs. Thus, the OpenShift built-in OAuth server acts as an identity broker.

When a user tries to log in to OpenShift, the OpenShift built-in OAuth server redirects the user to a login screen to choose from the configured IdPs. For OIDC IdPs, you must first configure the OIDC client in the IdP that authenticates the user to OpenShift. Then, use the connection parameters and credentials from the OIDC client to configure the OIDC IdP in the OpenShift built-in OAuth server. After authenticating the user in the configured IdP, the internal OAuth server creates an access token for the request and returns the token. The built-in OAuth server, as for any other IdP, also creates or updates the user resources. If you define OIDC group claims, then the OAuth server also creates or updates the groups.

#### OIDC Claims

OIDC claims are key/value pairs that contain information about the user. OpenShift reads user claims from the JWT token that the IdP issues, and uses these user claims to populate the user, identity, and group resources. You must configure one claim to use as the user's identity. The default identity claim is `sub`, which stands for _subject identifier_.

You can configure additional user parameters in other standard claims, such as the preferred username, display name, or email address. For any of those user parameters, you can specify multiple claims. OpenShift uses the first claim with a non-empty value.

The following list includes some standard claims that are defined in OIDC:

`sub`

The remote identity for the user at the IdP

`preferred_username`

The preferred username when provisioning a user, which typically corresponds to the username in the authentication system for the user

`email`

Email address

`name`

Display name

>Note
To see the full list of standard claims that are defined in OIDC, refer to [https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims)

#### OIDC Group Claims

You can also define additional claims for parameters that OIDC does not define as standard claims. One important additional claim for users is the `group` claim. OpenShift uses the `group` claim to map the group membership of a user in the IdP to a group object. Then, you can use role-based access control (RBAC) objects in OpenShift to assign permissions to that group, instead of assigning permissions to individual users.

> Note
The GitHub repository for the Red Hat Communities of Practice Group Sync Operator provides an unsupported operator for synchronizing OIDC groups with external providers that cannot provide group claims as part of their tokens. You can find more information in [https://github.com/redhat-cop/group-sync-operator](https://github.com/redhat-cop/group-sync-operator)

### Configuring OIDC IdP

To integrate an OIDC IdP into the OpenShift cluster, complete the following steps:

1. Obtain the client ID and the client secret from the OIDC IdP client for the OpenShift integration.
2. Create an OpenShift secret object, which contains the client secret that is obtained from the OIDC client configuration, in the `openshift-config` namespace.
3. Create an OpenShift configuration map object, which contains the certificate authority bundle in the `ca.crt` file parameter, in the `openshift-config` namespace (this step is required only if the CA certificate is not configured as a system-wide CA).
4. Create the OAuth CR YAML file to include the OIDC IdP.
5. Apply the configuration file to the OAuth CR.

#### Creating the OAuth CR YAML File

After creating the OpenShift secret object that contains the client secret, and the configuration object that contains the certificate authority bundle (if necessary), you can create the OAuth CR YAML file that contains the information to configure the OIDC IdP. If you add an OIDC IdP to the OAuth CR, then you must include the OIDC IdP information in the `identityProviders` array. You can add multiple OIDC IdPs to the OAuth CR YAML file.

### Important

The `identityProviders` array in the OAuth CR must not be empty. If you remove other IdPs when adding your OIDC IdP, then you cannot log in to the cluster through those IdPs.

The following example shows a minimal configuration file for OIDC integration to OpenShift. The settings might differ for other OIDC providers, and you must work with your vendor or identity administrator to get all the necessary attributes for your specific setup.

apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: _`oidc_provider_name`_ ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    mappingMethod: claim
    type: OpenID
    openID:
      clientID: _`oidc_clientid`_ ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
      clientSecret: ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
        name: _`secret_name`_
      ca: ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
        name: _`config_map_name`_
      claims: ![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
        preferredUsername:
        - preferred_username
        - email
        name:
        - nickname
        - given_name
        - name
        email:
        - custom_email_claim
        - email
        groups:
        - groups
      issuer: _`https://external_idp_url.com`_ ![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_creating_the_oauth_cr_yaml_file-CO14-1)|OpenShift prefixes the value of the identity claim with the provider name to form an identity name, and uses the identity name to build the redirect URL.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_creating_the_oauth_cr_yaml_file-CO14-2)|The client ID for the existing client in the IdP. You must enable the client to redirect to ``https://oauth-openshift.apps._`cluster_name`_._`cluster_domain`_/oauth2callback/_`idp_provider_name`_``.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_creating_the_oauth_cr_yaml_file-CO14-3)|The name for the OpenShift secret object that contains the client secret.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_creating_the_oauth_cr_yaml_file-CO14-4)|(Optional) The name for the OpenShift configuration map object that contains the certificate authority bundle.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_creating_the_oauth_cr_yaml_file-CO14-5)|The list of claims to use as the identity, such as the preferred username, email address, or groups.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_creating_the_oauth_cr_yaml_file-CO14-6)|The URL for the IdP. OpenShift accepts only HTTPS URLs.|

#### Log in to the Cluster Through the OIDC IdP

After adding the OIDC IdP to the cluster, you can log in through the IdP by using the OpenShift web console, and providing a username and password.

If your OIDC IdP supports the grant flow for resource owner password credentials (ROPC), then you can also log in through the IdP by using the `oc login` command with a username and password.

If your OIDC IdP does not support the ROPC grant flow, then you receive the `You must obtain an API token` login error when you use the `oc login` command with a username and password. Then, you must get an OAuth access token and use it to log in by using the ``oc login --token=_`access_token`_`` command. You can get the OAuth access token by logging in through the OpenShift web console and clicking Help → Command line tools → Copy login command. You can also request the OAuth access token through the OpenShift REST API.

> Note
Requesting an OAuth access token through the OpenShift REST API is out of the scope of this course. For more information about this topic, refer to [https://access.redhat.com/solutions/6610781](https://access.redhat.com/solutions/6610781)

#### IdP Mapping Methods

IdP mapping methods apply to any IdP. If you configure the `mappingMethod` parameter for the OIDC IdP as `claim` or `add`, then OpenShift establishes mappings between the provider's identity and the `User` object the first time that a user logs in to the cluster.

#### OAuth Access Tokens

The first time that a user logs in to the cluster, the OpenShift built-in OAuth server creates an OAuth token to authenticate to the API. OAuth access tokens are common to any IdP. OpenShift renews the API authentication token every time that the user logs in to the cluster. As a user, you can list all your user-owned OAuth access tokens by using the following command. This command lists all the user-owned OAuth access tokens from any IdP that is configured in the OpenShift built-in OAuth server.

```sh
[user@host ~]$ oc get useroauthaccesstokens
NAME            CLIENT NAME    CREATED   EXPIRES                        ...
sha256~9BZ3...  openshift-...  6m        2025-12-17 13:24:42 +0000 UTC  ...
sha256~lm1O...  console        7m        2025-12-17 13:19:20 +0000 UTC  ...
sha256~xmpC...  openshift-...  8m        2025-12-17 13:11:24 +0000 UTC  ...
```

If you modify a user parameter in the OIDC IdP, then the changes do not automatically synchronize with OpenShift. For example, if you remove a user account from the OIDC IdP and the user is logged in to OpenShift, then the user can still perform tasks in OpenShift until they log out, because they still have a valid token that the OpenShift built-in OAuth server issued.

For this reason, after you modify a parameter in the OIDC IdP, you must remove all the user access tokens from OpenShift to force a user logout. Then, after the user logs in again, OpenShift synchronizes the new user parameters.

You can use the following command to remove all access tokens for that user from OpenShift:

```sh
[user@host ~]$ `oc delete oauthaccesstoken $(oc get oauthaccesstoken -o \   jsonpath='{.items[?(@.userName=="_`username`_")].metadata.name}')`
```

## Token and Client Certificate Authentication with `kubeconfig` Files

### Objectives

- Generate a token and a client certificate and add them to a `kubeconfig` file.

### External Client Authentication

In certain scenarios, you might authenticate external clients, such as CI/CD pipelines or monitoring tools, to the Kubernetes cluster. Kubernetes enables external clients to authenticate to the Kubernetes API by embedding either a client certificate or an authentication token into a `kubeconfig` configuration file. This feature ensures that only authorized external clients can access the Kubernetes cluster.

### Service Accounts

A service account (SA) is a Kubernetes resource that provides an identity for a component to authenticate to the Kubernetes API server. SAs provide a flexible way to control API access without sharing a regular user's credentials. SAs use signed JSON Web Tokens (JWTs) to authenticate to the API server, as well as with external resources.

For example, you can use SAs in the following scenarios:

- A replication controller makes API calls to create or delete pods.
- A pod that collects logs might need access to certain log storage resources.
- A backup and restore tool might require access to the cluster's configuration and data to back up and restore data.
- An external application makes monitoring or integration API calls.

SAs are specific to a particular project and cannot be directly shared across projects. Each SA user name is derived from its project and name; for example, the following SA named `test-sa` belongs to the `test` project:

system:serviceaccount:test:test-sa

Every SA is also a member of the following two groups:

`system:serviceaccounts`

This group includes all service accounts in the system.

``system:serviceaccounts:_`project`_``

This group includes all service accounts in the specified project.

#### Manage SAs

To manage SAs, you use the standard `oc` commands, such as `create`, `get`, or `describe`. You can grant roles to SAs the same as for regular users. You can also use the `-z` option to avoid using the long ``system:serviceaccount:_`project`_:_`name`_`` SA name, and use instead the short SA ``_`name`_`` name.

#### Default Service Accounts

OpenShift contains default service accounts for cluster management, and generates the following service accounts for each project:

`builder`

OpenShift uses this SA to build pods. By default, this SA has the `system:image-builder` role, so the resource can push images to any image stream in the project by using the internal Docker registry.

`deployer`

OpenShift uses this SA in deployment pods. By default, this SA has the `system:deployer` role, so the resource can view and modify replication controllers and pods in the project. This SA exists only for applications that use OpenShift deployment configuration resources.

`default`

OpenShift assigns this default SA to pods if you do not specify a different SA when you create the pods.

#### Bound Service Account Tokens

Bound service account tokens are a more secure type of JWTs. Unlike the legacy secret-based tokens that never expired, bound service account tokens are time-limited, and are bound to specific objects or audiences. The bound tokens are invalidated when the bound object is removed. You can request bound service account tokens by using volume projection and the `TokenRequest` API.

>Warning
You can still manually create legacy long-lived API tokens for SAs. However, Red Hat recommends using long-lived API tokens only if you cannot use the `TokenRequest` API and if the security exposure of a non-expiring token is acceptable to you. To manually create long-lived API tokens for SAs, refer to [https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/nodes/index#nodes-pods-secrets-creating-sa_nodes-pods-secrets](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/nodes/index#nodes-pods-secrets-creating-sa_nodes-pods-secrets)

##### Configuring Bound Service Account Tokens by Using Volume Projection

You can configure pods to request bound service account tokens by using volume projection. The following diagram summarizes the authentication flow by using the `TokenRequest` API:

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do380-4.18/images/auth/tls/assets/tokenapi.svg)|

_Authentication flow for bound service account tokens_

|   |   |
|---|---|
|![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)|An application that is running in a pod needs to authenticate to the Kubernetes API. The `kubelet` agent requests a bound SA account token from the `TokenRequest` API.|
|![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)|The `kubelet` agent receives the SA token from the `TokenRequest` API and mounts it to the pod.|
|![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)|The application reads the SA token and uses it to authenticate to the Kubernetes API.|

The following pod YAML definition shows how to mount a bound SA token by using volume projection:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
_...output omitted..._
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
_...output omitted..._
  `serviceAccountName`: build-robot #![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          `path`: vault-token  #![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
          `expirationSeconds`: 7200  #![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
          `audience`: vault #![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_configuring_bound_service_account_tokens_by_using_volume_projection-CO18-1)|The name of the service account.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_configuring_bound_service_account_tokens_by_using_volume_projection-CO18-2)|The path relative to the mount point of the file to project the token to.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_configuring_bound_service_account_tokens_by_using_volume_projection-CO18-3)|Optional, the expiration time in seconds of the service account token. The default value is 3600 seconds. The kubelet starts trying to rotate the token if the token is older than 80 percent of its time to live, or if the token is older than 24 hours.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_configuring_bound_service_account_tokens_by_using_volume_projection-CO18-4)|Optional, the intended audience of the token. The recipient of a token verifies that the recipient identity matches the audience claim of the token, and should otherwise reject the token. The default value is the identifier of the API server.|

##### Creating Bound Service Account Tokens Outside a Pod

For external applications, you can manually generate a bound SA token by using the `TokenRequest` API, and use it as the bearer token for authentication to the Kubernetes API. The client includes the `Authorization: Bearer <token>` header with the HTTP request.

The following command creates a bound SA token for the `my-sa` SA by using the `TokenRequest` API:

[user@host ~]$ **`oc create token my-sa`**

You can set the lifetime for the SA token by using the --duration option. The default lifetime for the SA token is one hour. You must manually refresh the manually generated SA token before it expires, so that your application can continue authenticating to the Kubernetes API.

The following example shows a decoded JWT SA token:

{
  "aud": [
    "https://kubernetes.default.svc"
  ],
  `"exp"`: _`1765671322`_, ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
  `"iat"`: _`1765667722`_, ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
  "iss": "https://kubernetes.default.svc",
  "jti": "516a902a-c59c-4f3b-a24c-329dde511782",
  "kubernetes.io": {
    `"namespace"`: "_`test`_", ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
    "serviceaccount": {
      `"name"`: "_`my-sa`_", ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
      "uid": "e0dda498-399b-470c-84df-3813ac469fb2"
    }
  },
  "nbf": 1765667722,
  "sub": "system:serviceaccount:test:my-sa"
}

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_creating_bound_service_account_tokens_outside_a_pod-CO19-1)|The SA token expiration date in Unix epoch format|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_creating_bound_service_account_tokens_outside_a_pod-CO19-2)|The SA token issued date in Unix epoch format|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_creating_bound_service_account_tokens_outside_a_pod-CO19-3)|The project for the SA token|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_creating_bound_service_account_tokens_outside_a_pod-CO19-4)|The name for the SA|

### Client Certificate Authentication

Client certificate authentication in Kubernetes clusters refers to the process of authenticating clients, such as users or services, which access the Kubernetes cluster by using TLS client certificates.

By default, OpenShift provides an internal certificate authority (CA). The OpenShift internal CA is a built-in component of the OpenShift cluster that manages and issues digital certificates in the cluster. The internal CA provides a trusted source for generating X.509 certificates for secure communication, authentication, and encryption in the OpenShift cluster.

The Kubernetes API server requires client authentication by using client certificates. OpenShift is preconfigured to trust the client certificates that the OpenShift internal CA signs.

>Note
You can also configure additional client CAs for the Kubernetes API server. For more information about configuring additional client CAs for the Kubernetes API server, refer to [https://access.redhat.com/solutions/6054271](https://access.redhat.com/solutions/6054271)

This feature is mainly used to generate a client certificate for an administrator user, such as the predefined `system:admin` user, to use this client certificate as a backdoor for cluster administrators in the event of failure of the IdP that provides the administrator credentials.

Although Red Hat does not recommend doing so, you can also use client certificates in certain scenarios when running outside the cluster, such as CI/CD pipelines, automation playbooks, or monitoring tools. These clients need to present valid client certificates during the TLS handshake to establish a secure connection with the API server. This mechanism ensures that only authorized clients can access the API server from outside the cluster.

>Warning
Red Hat recommends using SAs to run automation outside the cluster whenever you can, instead of using client certificates, because a certificate cannot be revoked in OpenShift. Revoking a client certificate would require invalidating all client certificates that the current CA ever signed, and creating replacement certificates for all client users and applications. For more information about this topic, refer to [https://github.com/kubernetes/kubernetes/issues/18982](https://github.com/kubernetes/kubernetes/issues/18982)

#### Create a Client Certificate

OpenShift assigns the username for the user by using the common name (CN) field from the certificate, and assigns the user's groups by using the organization (O) fields. You can use role-based access control (RBAC) rules to provide the minimal rights to the client account to perform the job. For example, you could provide view permissions for the entire cluster to a monitoring tool, or edit permissions on selected projects to a CI/CD pipeline.

To create client certificates by using the internal OpenShift CA, you must follow these steps:

1. Create a certificate signing request (CSR): The first step is to create a CSR for the client. You can use the OpenSSL tool to create the CSR. This request includes the client's information and the public key. You can use more than one group for the user if you require it.
    
```sh
    [user@host ~]$ openssl req -noenc -newkey rsa:4096 -keyout key_filename \
    -subj "/O=group1/O=_`group2`_/CN=username" -out _csr_filename
    
```
1. Create the CSR YAML resource manifest. The following example shows the parameters for a CSR:
    
```yaml
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      `name`: csr_name #![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    spec:
      `signerName`: kubernetes.io/kube-apiserver-client #![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
      `expirationSeconds`: 604800  # one week ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
      `request`: $(base64 -w0 csr_filename) #![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
      `usages`: #![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
      - client auth
    
```

   |  |  | 
   |---|---|
    |[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_create_a_client_certificate-CO20-1)|The CSR name.|
    |[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_create_a_client_certificate-CO20-2)|The `kube-apiserver-client` built-in signer of certificates that the API server accepts. OpenShift provides different built-in signers that you can use to sign certificates. For more information, refer to [https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers)|
    |[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_create_a_client_certificate-CO20-3)|The expiration time for the certificate in seconds. If you do not specify a value, then the default value is 30 days.|
    |[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_create_a_client_certificate-CO20-4)|The OpenSSL X.509 CSR, which is encoded in Base64 format.|
    |[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_create_a_client_certificate-CO20-5)|The use cases for the client certificate. It must include the `client auth` usage, which indicates that the certificate is intended for client authentication.|
    
2. Submit the CSR to the Kubernetes API server. The API server interacts with the internal CA to process the CSR.
    
    [user@host ~]$ **`oc apply -f csr.yaml`**
    
3. Review, approve, and sign the CSR: You can review the CSR details by using the ``oc describe csr _`csr_name`_`` command. After reviewing the CSR details, use the following command so the internal CA signs the CSR to generate the client certificate:
    
    [user@host ~]$ **``oc adm certificate approve _`csr_name`_``**
    
4. Retrieve the client certificate: After you approve the CSR and the OpenShift internal CA generates the certificate, you can retrieve the signed certificate from the API server.
    
    [user@host ~]$ **``oc get csr _`csr_name`_ -o jsonpath='{.status.certificate}' \   | base64 -d > _`certificate_filename`_``**
    
5. Distribute the certificate: The signed certificate, together with the private key that generates the CSR, enables the client to authenticate to the cluster.
    

### OpenShift CLI Configuration Files

You can use a `kubeconfig` file as a CLI configuration file to set up profiles for use with the Kubernetes `kubectl` and OpenShift `oc` CLI tools. Moreover, most Kubernetes client libraries use `kubeconfig` files in the same way as the `kubectl` and `oc` CLI tools. Use `kubeconfig` files to authenticate external applications to the cluster by storing tokens and client certificates inside the `kubeconfig` files.

The `kubeconfig` file is defined as a YAML file that contains clusters, users, and contexts.

`clusters`

The `clusters` parameter in the `kubeconfig` file contains information about the OpenShift clusters, such as the IP or fully qualified domain name (FQDN), or the CA.

`users`

The `users` parameter contains the user credentials to interact with the Kubernetes API. This parameter contains information such as the username, the user password, or the user token.

`contexts`

The `contexts` parameter contains information about the combination of a cluster and a user to interact with the Kubernetes API. Whenever you run an `oc` command, you reference a context inside the `kubeconfig` file.

#### `kubeconfig` File Details

When you run any `oc` command, OpenShift reads a `kubeconfig` file in the following ways in turn:

1. The specified file in the `--kubeconfig` option, if you use it
    
2. The specified file in the `KUBECONFIG` environment variable, if it is set
    
3. The default `~/.kube/config` file
    

When you log in to the OpenShift cluster through the `oc login` command for the first time, OpenShift creates a `kubeconfig` file in the default `~/.kube/config` location, if the file does not exist. You can add more authentication and connection details to the `kubeconfig` file automatically by using the `oc login` and `oc project` commands, or by manually editing the `kubeconfig` file.

You can read the details from your `kubeconfig` file by using the `oc config view` command or by opening the file with a text editor. The following example shows the parameters for a `kubeconfig` file from an OpenShift cluster:

apiVersion: v1
`clusters`: ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
- cluster:
    server: https://api.ocp4.prod.com:6443
  name: production
- cluster:
    server: https://api.ocp4.stage.com:6443
    certificate-authority: ocp-apiserver-cert.crt
  name: stage
`users`: ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
- name: admin-production
  user:
    token: REDACTED
- name: admin-stage
  user:
    client-certificate: admin-stage.crt
    client-key: tls.key
`contexts`: ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
- context:
    cluster: production
    namespace: prod-app
    user: admin-production
  name: prod-app/api-ocp4-prod-com:6443/admin-production
- context:
    cluster: stage
    namespace: demo-app
    user: admin-stage
  name: demo-app/api-ocp4-stage-com:6443/admin-stage
current-context: demo-app/api-ocp4-stage-com:6443/admin-stage
kind: Config
preferences: {}

|                                                                                                                                            |                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| [![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_kubeconfig_file_details-CO21-1) | The list of all the clusters that you already connected to.              |
| [![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_kubeconfig_file_details-CO21-2) | The list of all the users that you already connected to the cluster.     |
| [![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_kubeconfig_file_details-CO21-3) | The list of contexts that you can reference when using the `oc` command. |

The previous `kubeconfig` example file defines two clusters, `production` and `stage`, with their FQDNs that are defined in the `server` parameter. The `stage` cluster definition also contains the public certificate from the API server in the `certificate-authority` parameter.

The file also defines two users, `admin-production` and `admin-stage`. The `admin-production` user definition uses the token that is defined in the `token` parameter to authenticate to the API server. OpenShift truncates the token information, to prevent the configuration file from becoming too long.

You can use the `--raw` option in the `oc config view` command to show the token information. The `admin-stage` user authenticates to the cluster by using a client certificate that the internal OpenShift CA previously signed, as defined in the `client-certificate` parameter, and the key that signed the client certificate, as defined in the `client-key` parameter.

Every context in the ``kubeconfig file specifies a combination of a user and a cluster that are already defined in the `kubeconfig`` file, through the `cluster` and `user` parameters, and a project through the `namespace` parameter.

#### Configure CLI Profiles

Although you can set most `kubeconfig` file parameters by using the `oc login` and `oc project` OpenShift commands, you can use the `oc config` command to manually configure these parameters if needed, instead of directly modifying the file. You can use the `oc config` command with the ``--kubeconfig=_`file`_`` option to create and store `kubeconfig` files with different parameters that you can use when required. The `oc config` command comes from the Kubernetes `kubectl` CLI tool, with no modifications from OpenShift.

Use the following `oc config` subcommands to manually configure your `kubeconfig` files:

`set-cluster`

Creates a cluster entry in the `kubeconfig` file. This subcommand accepts different cluster parameters as the server IP address or the CA file.

`set-credentials`

Creates a user entry in the `kubeconfig` file. This subcommand accepts different user parameters as the username, the user password, or the token.

`set-context`

Creates a context entry in the `kubeconfig` file. Use this subcommand to define a context to specify a combination of a user and a cluster. You can also set the OpenShift project.

`use-context`

Sets the current context.

>Note
For more information about manually configuring the `kubeconfig` files by using the `oc config` command, consult the references section.

### User Impersonation

User impersonation in OpenShift enables certain users or SAs to act on behalf of other users or SAs with different permissions and roles. This feature is useful when administrators or privileged users need to perform actions on behalf of regular users, or when SAs need to execute tasks on behalf of other SAs, or for regular users to perform actions with administrative permissions. For example, a system administrator can debug an RBAC rule by impersonating another user and verifying whether OpenShift accepts or denies a request. As another example, you want system administrators to escalate privileges when it is necessary instead of them logging in as cluster administrators.

#### System Administrator That Impersonates a Regular User

As a system administrator, use the `--as` and `--as-group` options when using the `oc` command to impersonate a user or group. Use the `oc auth can-i` command to test the user access to a particular resource in the cluster. Use the `-n` option to specify a project for the request, or use the `-A` option to verify the action in all the projects. Thus, use the following command to test the RBAC rules for a particular user or group by impersonating them:

[user@host ~]$ **``oc auth can-i _`command`_ --as _`user_to_impersonate`_ \   --as-group _`group_to_impersonate`_``**

You can also list all the permissions for a specific user or group by using the `oc auth can-i --list` command.

#### Regular User That Impersonates Another User

To grant a regular user permissions to impersonate another user, you must create a custom role with the appropriate permissions, and then create a role binding to assign the role to the user.

For example, to allow a regular user to run commands as an administrator, you can create the following cluster role, which enables anyone to impersonate the administrator user:

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sudo-admin
rules:
- `apiGroups`: [""]  ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
  resources: ["users"]
  verbs: ["impersonate"]
  resourceNames: ["admin"]

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_regular_user_that_impersonates_another_user-CO22-1)|The core API group is identified with an empty string.|

Then, bind the role to the user:

[user@host ~]$ **``oc create clusterrolebinding _`binding_name`_ \   --clusterrole sudo-admin --user _`regularuser`_``**

The user can impersonate the `admin` user by using the `--as=admin` option. The following example shows how the user cannot retrieve the node information when using their account, but can retrieve that information when impersonating the `admin` user:

[user@host ~]$ **`oc get nodes`**
Error from server (Forbidden): nodes is forbidden:
User "regularuser" cannot list resource "nodes" in API group "" at the cluster scope

[user@host ~]$ **`oc get nodes --as admin`**
NAME       STATUS   ROLES                         AGE     VERSION
master01   Ready    control-plane,master,worker   201d    v1.31.6
master02   Ready    control-plane,master,worker   201d    v1.31.6
master03   Ready    control-plane,master,worker   201d    v1.31.6
worker01   Ready    worker                        2d12h   v1.31.6
worker02   Ready    worker                        2d12h   v1.31.6
worker03   Ready    worker                        2d12h   v1.31.6

