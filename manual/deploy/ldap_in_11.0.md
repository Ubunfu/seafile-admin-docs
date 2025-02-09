# Configure Seafile to use LDAP

Note: This documentation is for the Community Edition. If you're using Pro Edition, please refer to [the Seafile Pro documentation](../deploy_pro/ldap_in_11.0.md).

## How does LDAP User Management work in Seafile

When Seafile is integrated with LDAP, users in the system can be divided into two tiers:

- Users within Seafile's internal user database. Some attributes are attached to these users, such as whether it's a system admin user, whether it's activated.

- Users in LDAP server. These are all the intended users of Seafile inside the LDAP server. Seafile doesn't manipulate these users directly. It has to import them into its internal database before setting attributes on them.

When Seafile counts the number of users in the system, it only counts the **activated** users in its internal database.

## Basic LDAP Integration

The only requirement for Seafile to use LDAP for authentication is that there must be a unique identifier for each user in the LDAP server. Seafile can only use email-address-format user identifiers. So there are usually only two options for this unique identifier:

- Email address: this is the most common choice. Most organizations assign unique email address for each member.

- UserPrincipalName: this is a user attribute only available in Active Directory. It's format is `user-login-name@domain-name`, e.g. `john@example.com`. It's not a real email address, but it works fine as the unique identifier.

### Basic configuration items

Add the following options to `seahub_settings.py`. Examples are as follows:

```python
ENABLE_LDAP = True
LDAP_SERVER_URL = 'ldap://192.168.0.1'     # The URL of LDAP server
LDAP_BASE_DN = 'ou=test,dc=seafile,dc=ren'   # The root node of users who can 
                                             # log in to Seafile in the LDAP server
LDAP_ADMIN_DN = 'administrator@example.com'  # DN of the administrator used 
                                             # to query the LDAP server for information.
                                             # For OpenLDAP, it maybe cn=admin,dc=example,dc=com
LDAP_ADMIN_PASSWORD = 'Hello@123'            # Password of LDAP_ADMIN_DN
LDAP_PROVIDER = 'ldap'                       # Identify the source of the user, used in 
                                             # the table social_auth_usersocialauth, defaults to 'ldap'
LDAP_LOGIN_ATTR = 'userPrincipalName'        # User's attribute used to log in to Seafile, 
                                             # can be mail or userPrincipalName, cannot be changed
LDAP_CONTACT_EMAIL_ATTR = ''                 # LDAP user's contact_email attribute
LDAP_USER_ROLE_ATTR = ''                     # LDAP user's role attribute
LDAP_USER_FIRST_NAME_ATTR = 'givenName'      # For sync user's first name
LDAP_USER_LAST_NAME_ATTR = 'sn'              # For sync user's last name
LDAP_USER_NAME_REVERSE = False               # Whether to reverse the user's first and last name
LDAP_FILTER = 'memberOf=CN=testgroup,OU=test,DC=seafile,DC=ren'  # Additional filter conditions,
                                             # users who meet the filter conditions can log in, otherwise they cannot log in

```

Tips for choosing `LDAP_BASE_DN` and `LDAP_ADMIN_DN`:

* To determine the `LDAP_BASE_DN`, you first have to navigate your organization hierachy on the domain controller GUI.

    * If you want to allow all users to use Seafile, you can use `cn=users,dc=yourdomain,dc=com` as `LDAP_BASE_DN` (with proper adjustment for your own needs).

    * If you want to limit users to a certain OU (Organization Unit), you run `dsquery` command on the domain controller to find out the DN for this OU. For example, if the OU is `staffs`, you can run `dsquery ou -name staff`. More information can be found [here](https://technet.microsoft.com/en-us/library/cc770509.aspx).

* AD supports `user@domain.name` format for the `LDAP_ADMIN_DN` option. For example you can use administrator@example.com for `LDAP_ADMIN_DN`. Sometime the domain controller doesn't recognize this format. You can still use `dsquery` command to find out user's DN. For example, if the user name is 'seafileuser', run `dsquery user -name seafileuser`. More information [here](https://technet.microsoft.com/en-us/library/cc725702.aspx).

## Advanced LDAP Integration Options

### Multiple BASE

Multiple base DN is useful when your company has more than one OUs to use Seafile. You can specify a list of base DN in the `LDAP_BASE_DN` option. The DNs are separated by ";", e.g.

```python
LDAP_BASE_DN = 'ou=developers,dc=example,dc=com;ou=marketing,dc=example,dc=com'
```

### Additional Search Filter

Search filter is very useful when you have a large organization but only a portion of people want to use Seafile. The filter can be given by setting `LDAP_FILTER` option. The value of this option follows standard LDAP search filter syntax (https://msdn.microsoft.com/en-us/library/aa746475(v=vs.85).aspx).

The final filter used for searching for users is `(&($LOGIN_ATTR=*)($LDAP_FILTER))`. `$LOGIN_ATTR` and `$LDAP_FILTER` will be replaced by your option values.

For example, add below option to `seahub_settings.py`:

```python
LDAP_FILTER = 'memberOf=CN=group,CN=developers,DC=example,DC=com'
```

The final search filter would be `(&(mail=*)(memberOf=CN=group,CN=developers,DC=example,DC=com))`

Note that the case of attribute names in the above example is significant. The `memberOf` attribute is only available in Active Directory.

### Limiting Seafile Users to a Group in Active Directory

You can use the `LDAP_FILTER` option to limit user scope to a certain AD group.

1. First, you should find out the DN for the group. Again, we'll use the `dsquery` command on the domain controller. For example, if group name is 'seafilegroup', run `dsquery group -name seafilegroup`.

2. Add below option to `seahub_settings.py`:

```python
LDAP_FILTER = 'memberOf={output of dsquery command}'
```

### Using TLS connection to LDAP server

If your LDAP service supports TLS connections, you can configure `LDAP_SERVER_URL` as the access address of the ldaps protocol to use TLS to connect to the LDAP service, for example:

```python
LDAP_SERVER_URL = 'ldaps://192.168.0.1:636/'
```
