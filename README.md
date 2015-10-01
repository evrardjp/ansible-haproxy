Role Name
=========

A brief description of the role goes here.

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well.

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

TO BE IMPROVED...

    - hosts: all
      vars:
        haproxy_use_extra_config_folder: False
        haproxy_ssl: True
        haproxy_ssl_self_signed_regen: False
        haproxy_services:
          - name: galera
            frontends:
              - external:
                  - ip: '127.0.0.1:6969'
                    ssl_termination: True
                  - ip: '127.0.0.1:6968'
                    ssl_termination: False
              - internal:
                  - ip: '127.0.0.1:5555'
                    ssl_termination: True
                  - ip: '127.0.0.1:5556'
                    ssl_termination: False
            backends:
              - servers:
                  - ip: '127.0.0.1:3307'
                  - ip: '127.0.0.1:3308'
              - backups:
                  - ip: '127.0.0.1:3309'
          - name: horizon
            frontends:
              - external:
                  - ip: '127.0.0.1:6970'
                    ssl_termination: False
                  - ip: '127.0.0.1:6971'
                    ssl_termination: False
              - internal:
                  - ip: '127.0.0.1:5557'
                    ssl_termination: True
                  - ip: '127.0.0.1:5558'
                    ssl_termination: False
            backends:
              - servers:
                  - ip: '127.0.0.1:3310'
                  - ip: '127.0.0.1:3311'
              - backups:
                  - ip: '127.0.0.1:3312'
      roles:
        - ansible-haproxy

License
-------

Apache2

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
