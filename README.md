Ansible HAProxy
===============

[![Build Status](https://semaphoreci.com/api/v1/projects/67cc3c0d-550d-4d99-99ff-b01c40f899cc/552340/badge.svg)](https://semaphoreci.com/evrardjp/ansible-haproxy)

This role installs haproxy based on your configuration, with a simple dict.

Requirements
------------

openssl if haproxy_ssl is set to True.

Role Variables
--------------

* ```haproxy_packagenames``` is the list of packages that will be installed on
  the deployed system. By default, socat hatop and haproxy are installed.
  The first two are useful for managing haproxy.
* ```haproxy_default_backend_options_http``` is a list of the options you can
  use in your configuration file for http backend, for convenience
* ```haproxy_default_backend_options_https``` is the same as its http
   counterpart, for https use.
* When ```haproxy_bind_on_non_local``` is set to True, the role will change
  your host sysctls to allow haproxy to bind on non-local ips.
* When ```haproxy_use_extra_config_folder``` is set to True, the role will
  deploy its configuration files into subfolders (in a conf.d folder inside
  the haproxy default configuration folder of your OS).
* When ```haproxy_webstats``` is set to True, webstats will be enabled for
  haproxy. You'll have to define the IP/port webstats will bind on
  (with ```haproxy_webstats_bind```) and the authentication credentials
  (with ```haproxy_webstats_auth```)
* Still for the webstats, you can define ```haproxy_webstats_hide_version```,
  ```haproxy_webstats_uri``` and ```haproxy_webstats_realm```.
  The use of these variables should be self explanatory. You can still check
  on HAProxy upstream documentation, if needed.
* ```haproxy_localstats``` is a way to enable/disable local socket for
  managing haproxy. It can be used with hatop and/or socat. By default, the
  local stats socket is enabled, with admin mode. The level of the access
  can be modified by replacing 'level admin' with the level you need
  in the variable ```haproxy_localstats_level```.
* A variable ```haproxy_localstats_timeout``` exists if you want to
  define a timeout on the localstats socket. Cf. upstream documentation.
* ```haproxy_ssl``` enables HAProxy ssl handling. By default to False, it
  allows HAProxy to terminate ssl connections.
* ```haproxy_ssl_default_dh_param``` is the variable defining the parameter
  used in the diffie-hellman key exchange algorithm. Upstream default is
  1024, which is insecure, therefore it's defaulted here to 4096 (which
  may be too much for your environement).
* When ```haproxy_ssl_self_signed_regen``` is set to True, haproxy will
  regenerate a new self-signed certificate on each role run.
* To change the self-signed certificate subject, edit the
  ```haproxy_ssl_self_signed_subject``` variable. An example can be
  found in the defaults/main.yml file
* ```haproxy_ssl_cert``` is the path to your public certificate in the
  deployed host
* ```haproxy_ssl_key``` is the path to your private key in the deployed
  host
* ```haproxy_ssl_ca_cert``` is the path to your ca certificate in the
  deployed host
* ```haproxy_ssl_pem``` is the path to the combined certificate in the
  deployed host (combination of certificate - ca - private key)
* These certificates files can be uploaded from the deployement host
  by setting their path on the deployment host in the according user
  variables:
    - ```haproxy_user_ssl_cert```
    - ```haproxy_user_ssl_key```
    - ```haproxy_user_ssl_ca_cert```
* ```haproxy_ssl_bind_cipher_suite``` is the default cipher suite
  used when doing ssl termination. You can override it by setting
  it to your own value.
* ```haproxy_ssl_bind_options``` is a string holding the default
  ssl protocols allowed on the bind. You can define your own string.
  Default is ```no-sslv3 no-tls-tickets force-tlsv12```


Dependencies
------------

None

Example Playbook
----------------

    ---
    - hosts: all
      vars:
        haproxy_use_extra_config_folder: True
        haproxy_ssl: True
        haproxy_ssl_self_signed_regen: False
        haproxy_services:
          galera:
            frontends:
              - name: galera-external
                binds:
                  - ip: '127.0.0.1:3400'
                  - ip: '127.0.0.1:3401'
                balance_type: 'tcp'
              - name: galera-internal
                binds:
                  - ip: '127.0.0.1:3402'
                balance_type: 'tcp'
            backends:
              - name: galera-backend-servers
                servers:
                  - name: galera-1
                    ip: '127.0.0.1:3307'
                  - name: galera-2
                    ip: '127.0.0.1:3308'
                backups:
                  - name: galera-3
                    ip: '127.0.0.1:3309'
          horizon:
            frontends:
              - name: external
                binds:
                  - ip: '127.0.0.1:9443'
                    ssl_termination: True
                    #ssl_certificate: "/etc/ssl/certs/mycert.pem"
                    ssl_ciphers: "ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS"
                  - ip: '127.0.0.1:9444'
                    ssl_termination: False
                balance_type: 'http'
              - name: internal
                binds:
                  - ip: '127.0.0.1:7443'
                    ssl_termination: True
                  - ip: '127.0.0.1:7444'
                    ssl_termination: False
                balance_type: 'http'
            backends:
              - name: horizon-backends
                servers:
                  - name: horizon-1
                    ip: '127.0.0.1:80'
                  - name: horizon-2
                    ip: '127.0.0.1:81'
                backups:
                  - name: horizon-3
                    ip: '127.0.0.1:82'
      roles:
        - ansible-haproxy


License
-------

Apache2

Author Information
------------------

Jean-Philippe Evrard <jean-philippe@evrard.me>
