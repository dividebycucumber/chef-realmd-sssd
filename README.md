# realmd-sssd

[realmd](http://www.freedesktop.org/software/realmd/) is a DBus service that configures network authentication and domain membership in a standard way. It provides automatic realm or domain discovery and configures SSSD or winbind to do the actual network authentication and user account lookups.

[SSSD (System Security Services Daemon)](https://fedorahosted.org/sssd/) allows a local service to check with a local access/authentication cache in SSSD, but that cache may be taken from any variety of remote identity providers -- an LDAP directory, an Identity Management domain, Active Directory, possibly even a Kerberos realm.

An integrated realm furnished with the appropriate schema and attribute values provides a single management point from which to control authentication and authorization policies for an organization. See the [Integration](#integration) section below for examples.

## Supported Platforms

- Debian 8
- CentOS 7
- Fedora 23
- Ubuntu 14

## Attributes

| Attribute | Type | Default | Description |
| ---       | ---  | ---         | ---     |
| `['realmd-sssd']['join']` | Boolean |  false | Whether or not to join the domain. |
| `['realmd-sssd']['debian-mkhomedir-umask']` | String |  '0022' | Octal representation of the home directory creation mask (Debian platform family specific). |
| `['realmd-sssd']['packages']` | Array | *varies by platform* | List of packages required for `realmd` to join the realm. |
| `['realmd-sssd']['host-spn']` | String | `node['fqdn']` given proper DNS, or `node['machinename']` | Alternate computer name to use when joining the domain (i.e.: ec2 instance ID) |
| `['realmd-sssd']['password-auth']` | Boolean | false | Enable OpenSSH server password authentication system-wide. Note that a value of `false` __does not__ explicitly disable password authentication -- for that, refer to  `['realmd-sssd']['ldap-key-auth']` attributes. A value of `true` will override `['openssh']['server']['password_authentication']` with 'yes' |
| `['realmd-sssd']['ldap-key-auth']['enable']` | Boolean | false | Enable OpenSSH server to retrieve public key information from LDAP only for listed networks. Enables password authentication on these networks as a fallback. Disables password authentication system-wide via a `Match *` directive. |
| `['realmd-sssd']['ldap-key-auth']['cidr']` | Array | empty Array | List of IPv4 or IPv6 CIDR blocks from which to allow SSH connections. |
| `['realmd-sssd']['config']` | Hash | *default SSSD configuration Hash* | Default SSSD configuration hash. You likely won't need to change this. |
| `['realmd-sssd']['extra-config']` | Hash | emtpy Hash | Extra configuration which will be merged and override the default SSSD configuration Hash. See Usage for an example. |
| `['realmd-sssd']['vault-name']` | String | 'realmd-sssd' | Name of the chef-vault with the vault item holding realm information required to join. |
| `['realmd-sssd']['vault-item']` | String | 'realm' | Name of the chef-vault item containing values for `realm`, `username`, `password`, and optionally `computer-ou`. |

## Usage

### Default Recipe

Include `realmd-sssd` in your node's `run_list`:

```json
{
  "run_list": [
    "recipe[realmd-sssd]"
  ]
}
```

### Attributes

The example Chef role default attributes below demonstrate how to add and override `sssd.conf` values via the `node['realmd-sssd']['extra-config']` attribute.

```json
{
  "default_attributes": {
    "realmd-sssd": {
      "join": "true",
      "extra-config": {
        "[domain/example.org]": {
          "realmd_tags": "managed by chef",
          "ad_access_filter": "(&(memberOf=CN=linux-users,OU=Groups,DC=example,DC=org)(objectCategory=user)(sAMAccountName=*))"
        }
      },
      "debian-mkhomedir-umask": "0077",
      "ldap-key-auth": {
        "enable": "true",
        "cidr": [
          "192.0.2.0/24"
        ]
      }
    }
  }
}

```
### Merge Behavior
Note that values supplied in the the `extra-config` Hash will merge if data types allow, and overwrite if not. For example, when given a String: `"realmd_tags": "managed by chef"` the default realmd_tags will be replaced, but given an Array `"realmd_tags": [ "managed by chef" ]` the default realmd_tags are appended.

## Testing

See .kitchen.yml and .kitchen.local.yml.EXAMPLE.

To create a local data bag required by the Test Kitchen with-registration suite:

```bash
# Assuming ChefDK
chef exec bundle install --with development
knife solo data bag create realmd-sssd realm -c .chef/solo.rb

# example Chef Vault item content:
{
  "id": "realm",
  "realm": "example.org",
  "username": "testuser",
  "password": "<<REDACTED>>",
  "computer-ou": "OU=Linux-Hosts,DC=example,DC=org"
}

# run integration tests
kitchen test --destroy never
```

Integration testing SUTs will not automagically leave the domain and clean up after themselves when destroyed.

For the purposes of integration testing, Chef Vault will fall back to reading plaintext data bag contents. In a production deployment, please consider setting `['chef-vault']['databag_fallback']` to false.

## Integration

### LDAP  SSH public keys

Public SSH keys must be present via [`sshPublicKey`](https://github.com/jirutka/ssh-ldap-pubkey/blob/master/etc/openssh-lpk.schema) or another attribute in LDAP. The key must begin with the key-type prefix or a custom SSH `AuthorizedKeysCommand` must be specified (i.e. if querying the attribute returns "SSHKey: ssh-rsa AAAA..." instead of "ssh-rsa AAAA...").

The following key must be present for the integration tests to succeed:
```
testuser@example.org@sssd-ubuntu14:~$ ldapsearch -LLL -H ldap://dc1.example.org -Y GSSAPI -b 'DC=example,DC=org' 'CN=Test User' altSecurityIdentities
SASL/GSSAPI authentication started
SASL username: testuser@EXAMPLE.ORG
SASL SSF: 56
SASL data security layer installed.
dn: CN=Test User,CN=Users,DC=example,DC=org
altSecurityIdentities: ecdsa-sha2-nistp521 AAAAE2VjZHNhLXNoYTItbmlzdHA1MjEAAAA
 IbmlzdHA1MjEAAACFBAHJ3sWSWZfjEw3mGCqIksxa8mRl5X3LEvLuLuMtIv/U9Efaku/lsLNNsmUi
 Q2x/8+k4TummCCR37vTnmsdB+BljdQFOOrq7FXJjaAQrHqIXDc/B2X5HIWveG6KbOnPluSLdenrrz
 m1CpZn5WHS2HePyS1+2OEalX+JZsStCVwZKlVTHJw== Test Kitchen realmd-sssd integrat
 ion key
 ```

### LDAP sudoers

The [sudo source code](https://www.sudo.ws/download.html#source) has relevant LDAP schemas under `doc/`. LDAP `sudoRole` objects can define sudoers policy for users and groups.

```
testuser@example.org@sssd-ubuntu14:~$ ldapsearch -LLL -H ldap://dc1.example.org -Y GSSAPI -b 'DC=example,DC=org' 'CN=linux-admins' member name object{Category,Class} sudo{Command,Host,Option,User}
SASL/GSSAPI authentication started
SASL username: testuser@EXAMPLE.ORG
SASL SSF: 56
SASL data security layer installed.
dn: CN=Linux-Admins,OU=SSSD-Integration,DC=example,DC=org
objectClass: top
objectClass: group
member: CN=Test User,CN=Users,DC=example,DC=org
name: Linux-Admins
objectCategory: CN=Group,CN=Schema,CN=Configuration,DC=example,DC=org

dn: CN=Linux-Admins,OU=Sudoers,OU=SSSD-Integration,DC=example,DC=org
objectClass: top
objectClass: sudoRole
name: Linux-Admins
objectCategory: CN=sudoRole,CN=Schema,CN=Configuration,DC=example,DC=org
sudoHost: ALL
sudoUser: %Linux-Admins
sudoOption: !authenticate
sudoCommand: ALL
```

## License and Authors

Author:: John Bartko (jbartko@gmail.com)

```text
Copyright:: 2016 John Bartko

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
