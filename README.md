[![Build Status](https://travis-ci.org/aalaesar/manage-bind.svg?branch=primary)](https://travis-ci.org/aalaesar/manage-bind)
# Ansible Role: manage-bind 2.0
 This role is built as an abtraction layer to configure bind and create DNS zones using YAML syntax.
- [x] Install and manage your bind9 server on Debian/Ubuntu servers.
- [x] Use YAML syntax/files to configure Bind options, zones, etc.

## Requirements
Ansible 2.4+

**Note:** this role requires root access to the bind server
> playbook.yml:
```yaml
- hosts: dnsserver
  roles:
    - role: aalaesar.manage-bind
      become: yes
```

## Role Variables
```yaml
# host configuration
bind_user:  bind
bind_group: bind
host_dns_srv: self
# log directory
bind_log_dir: /var/log/bind
# Installation configuration
bind_pkg_state: present
bind_pkgs: ['bind9', 'dnsutils']
bind_service_state: started
bind_service_enabled: yes
# Configs
bind_configs_dir: /etc/bind
bind_config_primary_zones: []
bind_config_default_zones: 'yes'
RFC1918: no
# Zones files
bind_zones_dir: /var/lib/bind
# Zones default configuration
zones_config_ttl: 38400 #10h45m
zones_config_refresh: 10800 #3h
zones_config_retry: 3600 #1h
zones_config_expire: 604800 #1w
zones_config_minimum: 38400 #10h45m

# remove unmanaged files to keep the server clean
remove_unmanaged_files: true
# initialise list of managed zone files :
list_zone_files: []
```

## Configuring Bind
### Introduction to Bind's configuration
Bind uses **clause's statements inheritance** mechanism to allow precise configuration.
- A **_clause_** is a class with its own **set** of statements.
  - They can have specific or common statements.
  - A clause defined inside another clause will implicitly inherit its mother's common statements.
- A **_statement_** is a clause's property.
  - It describes the server's behavior on how to perform a task, whether, when, etc.
  - It may be explicitly or implicitly defined.
- ***Some inheritance rules***:
 - **Options** is the _top_ clause that include all others.
 - **zone** is the _lowest_ clause. It can't have any child. It also contains **zone records**.
 - a clause can override its parent's statement and pass on the new value to its child(ren) by redefining explicitly the statement.
> Here is an ASCII example of this rules:
```
|##########|  
|  zone1   |  |#########|
|==========|  |  zone2  |
|statement1|  |=========|
|  =john   |  |         |
|##########|  |#########|
     /\            /\
     ||            ||
|#######################|   |##############|   |##############|
|         view1         |   |   zone3      |   |    zone4     |
|=======================|   |==============|   |              |
|  statement2=kangaroo  |   |statement1=bar|   |##############|
|#######################|   |##############|          /\
           /\                     /\                  ||
           ||                     ||                  ||
|#############################################################|
|                          Options                            |
|=============================================================|
|                       statement1=foo                        |
|                      statement2=koala                       |
|#############################################################|
Final result:
According statement1 & statement2 are common to all clauses
zone1: statement1=john, statement2=kangaroo
zone2: statement1=foo, statement2=kangaroo
zone3: statement1=bar, statement2=koala
zone4: statement1=foo, statement2=koala
```

**manage-bind** support the following `clauses`:
- _acl_
- _dnssec-policy_
- _options_
- _zone_
- _key_

The list of supported `statements`:
See `./vars/main.yml`

**TODO**:  _views_


### _Caution_ when defining a statement !
- some statements are defined with complex mappings while others requires just a simple value.
  Just in case, each statement have its own template self documented.
- **manage-bind** uses bind's tools **named-checkconf** and **named-checkzone** for configuration and zone validation.
  However, thoses tools are limited to **syntax** and **light coherence** verification. _This role do **not** provide advanced validation methods._
- Escape special chars like @ with quotes
- Some statements requires "yes|no" string values: Escape **yes** and **no** with quotes as Ansible evaluates them as boolean.

### The ACL clause

`acl` clauses are optional and define Access Control Lists that can be used in `options`. ACLs are list of IP addresses or subnet ranges. See Bind9 documentation ([acl](https://bind9.readthedocs.io/en/v9.18.26/reference.html#acl-block-grammar), [address match lists](https://bind9.readthedocs.io/en/v9.18.26/reference.html#address-match-lists)) for details. 

> playbook.yml
```yaml
- name: Install Bind9 DNS server
  hosts: dns
  roles:
    - role: aalaesar.bind
      become: false
      acls:
        intranets:
          - '192.168.88.0/24'
          - '192.168.89.0/24'
        containers:
          - '192.168.100.0/24'
      options:
        allow_query:
          - localhost   # 'localhost' is predefined ACL in Bind9
          - intranets   # defined ACL
          - containers  # also a defined ACL
```

### The DNSSEC Policy clause

**Note:** The Bind9's default DNSSEC policy (`dnssec-policy: default`) is most likely what you want unless you *really* know what you are doing. Use `none` to disable DNSSEC. Please **do not** use the example below as any guidance for DNSSEC configuration. 

> playbook.yml
```yaml
- name: Install Bind9 DNS server
  hosts: dns
  roles:
    - role: aalaesar.bind
      become: false
      options:
        dnssec_validation: auto
        dnssec_policy: test-dnssec-policy  
      dnssec_policies:
        test-dnssec-policy:
          cdnskey: "yes"
          cds_digest_types:
            - "SHA-256"
            - "SHA-512"
          dnskey_ttl: 7200
          inline_signing: "yes"
          keys:
            - type: csk
              key_directory: /var/lib/bind/keys
              lifetime: 3600
              algorithm: rsasha256 
              algorithm_value: 2048
            - type: zsk
              key_directory: /var/lib/bind/keys
              lifetime: P30D
              algorithm: ecdsa256
          max_zone_ttl: 10000
          nsec3param:
            iterations: 10
            optout: "no"
            salt_length: 64
```

### The Options clause
**Note:** The role comes with some default options.
#### Defining options' statements
When calling **manage-bind**, you can pass options statements:
 - in an external YAML file declared with `options_file`.
   - it must contain a mapping of statements called `options`.
 - in a mapping called `options` inside your playbook

**Note:** _First method excludes the second:_ the role will load only the statements in the file if you declare it in your playbook.
> playbook.yml:
```yaml
- hosts: dnsserver
  become: yes
  roles:
    - role: aalaesar.manage-bind
      options_file: ./files/options.yml # the role will only load those options
      options: # the next lines are useless is this case
        statement1: ...
        statement2: ...
```
> ./files/options.yml:
```yaml
---
options:
  statement1: ...
  statement2: ...
```
The list of all the statements available for the options is in **./tests/bind_options.yml**

#### Changing the role's default options :
They are defined in **./defaults/default_options.yml**

You may use this file to share a common policy over your infrastructure and override specific options easily

### Zones clauses
Zone clauses are defined with **statement** _and_ **zone records**
#### zone declaration
Each zone is declared as an element of the list named `zones`.

**'zones'** have to be defined in the playbook and is _mandatory_.
> playbook.yml:
```yaml
- hosts: dnsserver
  become: yes
  roles:
    - role: aalaesar.manage-bind
      zones:
        "zone 1":
          statements ...
        "zone 2":
          statements ...
```

#### Defining zone's statements
A zone is a mapping where the zone name is its main key  and its statements are keys:value
- `[statement]:value`
- `type` is mandatory statements
- `force_file` [boolean]: Tell ansible to rewrite the records database file. Useful if your zone is dynamically populated by DNS. - _false for secondary zones_ - _true for others_
- Some zone types have their own mandatory statements

> playbook.yml:
```yaml
zones:
  example.com: # The domain's name
    type: primary # Mandatory. The type of the zone : primary|secondary|forward|stub
    recursion: "no" # statement overriding global option "recursion" for this zone.
    ... # etc
```


The list of all zone's statements available is in **./tests/zone_statements.md**

#### Defining zone's records
the **zone's** records are defined in a mapping named **'records'**

This mapping **'records'** can be declared:
- in an YAML file specified in the **_yamlfile_** key.
- _as a mapping inside its zone's mapping_

**Note:** _the content of the Yaml files is combined with the zone config:_ So, in case of duplicates records, __the file content will have precedence.__

> playbook.yml:
```yaml
zones:
  example.com:
    records:
      SOA: ...
      NS: ...
      ... # etc
    ... # etc
  test.tld:
    yamlfile: "./files/test.tld.yml"
    records:
      SOA: ...
      NS: ...
      A:
       localhost: 127.0.0.1
       test1: 9.8.7.6 # will be overriden by the yamlfile
    ... # etc
```

In the yaml file, **records** must be top level mapping:
> ./files/test.tld.yml:
```yaml
---
records:
  SOA: ...
  NS: ...
  A:
   test1: 1.2.3.4
   test2: 5.6.7.8
  ... # etc
```

#### Adding records in the zone
Zone records have different types, they are declared by type inside `records`.

_each records type is different and follow its own YAML structure._

Manage-bind support the following records
- **SOA**
- **NS**
- **A**
- **AAAA**
- **MX**
- **SRV**
- **PTR**
- **CNAME**
- **DNAME**
- **TXT**
- ttl can be declared along the zone's records

## Dependencies

None.

## Example Playbooks
### Exemple configuration :
- You own the zones example.tld, example.com and example.org
- You have 2 name servers : dnserver1 (11.22.33.44) & dnserver2 (55.66.77.88)
- dnserver1 is primary of example.tld and secondary of example.com.
- dnserver2 is primary of example.com and secondary of example.tld
- example.tld is dynamically populated by a DHCP server

### dnserver1's playbook :
```yaml
---
- hosts: dnserver1
  roles:
    - role: aalaesar.manage-bind
      options:
        allow_recursion: '55.66.77.88'
        allow_transfer: '55.66.77.88'
      zones:
        example.tld:
          type: primary
          force_file: no
          notify: '55.66.77.88'
          allow_update:
            - key dhcp_updater
          records:
            - SOA:
                serial: 2016080401
                ns: dnserver1.example.tld.
                email: admin.example.tld.
            - NS:
              - dnserver1.example.tld.
              - dnserver2.example.tld.
            - A:
                dnserver1: 11.22.33.44
                dnserver2: 55.66.77.88
        example.com:
          type: secondary
          primaries: '55.66.77.88'
      keys:
      - name: dhcp_updater
        algorithm: "hmac-md5"
        secret: "{{myvault_dhcp_key}}"
```
### dnserver2's playbook :
```yaml
---
- hosts: dnserver2
  roles:
    - role: aalaesar.manage-bind
      options:
        allow_recursion: '11.22.33.44'
        allow_transfer: '11.22.33.44'
      zones:
        example.com:
          type: secondary
          notify: '11.22.33.44'
        example.tld:
         type: secondary
         primaries: '11.22.33.44'
         ymlfile: example.com.yml
```
### YAML file for example.com.yml on dnserver2
```yaml
---
records:
  ttl: 3d
  SOA:
    serial: 2016080401
    ns: dnserver2.example.com.
    email: admin.example.com.
  NS:
    - srvdns01.example.com.
  A:
    127.0.0.1:
      - '@'
      - dnserver2.example.com.
    host1: 12.34.56.78
    mailsrv: 98.76.54.32
    'ftp.domain.tld.': 95.38.94.196
  MX:
    '@':
      - target: backup.fqdn.
        priority: 20
  CNAME:
    host1: ftp
    '@':
      - www
      - webmail
  TXT:
    - text: '"v=spf1 mx -all"'
    - text: '( "v=DKIM1; k=rsa; t=s; n=core; p=someverylongstringbecausethisisakeyformailsecurity" )  ; ----- DKIM key mail for example.com'
      label: mail._domainkey
```

More examples are available in the tests folder
## License
BSD
