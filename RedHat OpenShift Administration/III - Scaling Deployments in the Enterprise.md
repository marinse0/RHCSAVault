## LDAP Authentication

### Objectives

- Configure an LDAP identity provider.
    

### The LDAP Service

After the installation of a new Red Hat OpenShift Container Platform (RHOCP) cluster, only a `kubeadmin` user exists. A cluster administrator must next create users and groups, or more commonly, integrate a business system that both provides these entities and manages authentication. In doing so, an OpenShift cluster adopts a consistent corporate network authentication approach with other business systems, and inherits the existing security implementations in place, such as the company access restrictions that are available through a VPN-based connection.

Integrating an existing authentication method, and available users and groups in your organization, by using a Lightweight Directory Access Protocol (LDAP) server, is a common approach for many businesses. An LDAP directory server defines a standard application protocol for querying the system to authenticate users and groups. Additionally, LDAP integrations enable the use of Red Hat Identity Management (IdM) and Microsoft Active Directory (AD) services that many organizations use for managing authentication.

An LDAP server can be configured as an Identity Provider (IdP) for authentication bindings between the LDAP identity entries and cluster users through the OpenShift OAuth server. When users enter credentials for the cluster, the OAuth server initiates a connection and a corresponding query to the LDAP server, and creates bindings when matching unique entries are found.

Although this implementation is sufficient for authenticating the user, it is still necessary to define the access for the user, based on group memberships and roles. Furthermore, configuration within the OpenShift cluster is required to synchronize groups from the LDAP database, as well as to define the appropriate access and permissions for each group.

### Note

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

### Note

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

[user@host ~]$ mkdir -p ocp4.redhat.com/users
[user@host ~]$ touch ocp4.redhat.com/users/ptomchek
[user@host ~]$ tree ocp4.redhat.com/
ocp4.redhat.com/
  └── users
    └── ptomchek

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

apiVersion: v1
kind: Secret
metadata:
  name: ldap-secret
  namespace: openshift-config
type: Opaque
data:
  bindPassword: _`base64-encoded-bind-password`_

>Note
Consider that cluster administrators can read the password that is stored in the secret. Red Hat recommends using a service account with limited permissions as the bindDN within LDAP to avoid any security breaches.

#### Create the Certificate Configuration Map

LDAP configurations for OAuth IdPs use an OpenShift configuration map resource in the `openshift-config` namespace to define the certificate authority bundles.

Create an OpenShift configuration map resource that contains the certificate authority. You must store the certificate authority in the `ca.crt` key of the configuration map resource.

The following example is a YAML file that defines how to create a configuration map resource for the certificate bundle:

apiVersion: v1
kind: ConfigMap
metadata:
  name: ca-config-map
  namespace: openshift-config
data:
  ca.crt: |
    _`CA_certificate_PEM`_

#### Update the OAuth CR

The following example shows the configuration for defining an LDAP IdP in the OAuth CR YAML file:

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

### Note

The OAuth CR contains all configured IdPs. Any additional IdP that is configured is appended to the CR. Replacing or removing any entries can result in the loss of access to your cluster.

### Log in by Using the Added LDAP IdP

After you update the OAuth CR, the new LDAP IdP is available to the cluster. The IdP `Name` is shown among any other configured IdPs on the web console login page, as well as in the list of configured OAuth IdPs.

Log in with this added LDAP IdP by using the `oc login` command or the added selection from the OpenShift web console, and by providing a username and password that are available through the LDAP server. During authentication, OpenShift generates a search filter by combining the attribute and filter in the configured OAuth CR `url` parameter, with the provided username. Then, OpenShift applies this filter to the LDAP directory to find a unique entry.

From the preceding configuration example, if the user enters `paula.tomcheck` as a username, then a query for that value is sent to the LDAP server. This query attempts to find a unique match in the `uid` LDAP field that the OAuth CR `url` value `"ldaps://ldaps.example.com/ou=users,dc=acme,dc=com?uid"` specifies.

If you configure the `mappingMethod` parameter for the LDAP IdP entry in the OAuth CR as `claim` or `add`, then OpenShift uses the resulting LDAP identity to create a cluster `User` resource on the first attempt to access the cluster.

Authentication fails if the LDAP lookup does not return a unique match. In this case, no binding is created, no user is added to the cluster, and access is denied.