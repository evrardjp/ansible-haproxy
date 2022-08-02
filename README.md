There are multiple roles to configure HAProxy.

Here is what I bring this role back from the dead:
- While HAProxy can be configured with simple `listen: bind: server: `, HAProxy is far more than that.
  It can, for example, allow environment variables, conditionals, and other directives in its config.
  That syntax is generally not supported in the templates I have seen on galaxy.ansible.com
- To support many features, some roles went to the option to simply passing a larg string and copy that to the HAProxy node.
  While this is very flexible, it requires people to know exactly what to pass,
  and is very delicate to override or merge with different variables.
- Some roles simply considered that templating should be limited to a certain set of features, and do not allow further customization.
  These are too simplistic for my need.
- Some roles did the opposite approach, and defined all the variables HAProxy can support.
  These are the best roles for flexibility, but still fail to deal with my first point.
  On top of that, every feature requires an update of the role, and the defaults are not usually simple (too many knobs to turn).
- Contributions are slow, due to lack of focus, and lack of community building around the role.
- Many roles are not providing the necessary validation of HAProxy
- Some roles are not actively maintained

This gave me a reason to restart this role.

## How will this role behave?

HAProxy configuration is split into the following sections:

- Globals:
  - Process management and security
  - Performance tuning
  - Debugging
- Proxies:
  - defaults
  - frontend
  - backend
  - listen
- Extra sections:
  - Userlist
  - Resolvers
  - Cache
  - fcgi-app

Extra sections are very situational and optional, but will be supported.
The lambda user of this role will most likely care about globals and proxies.

From the HAProxy configuration guide:
"Parameters in the "global" section are process-wide and often OS-specific. They
are generally set once for all and do not need being changed once correct. Some
of them have command-line equivalents."

This role will:
- Ensure the role is close to the default OS specific globals.
  If a variable is different than the OS standard global, it will be documented in a git commit history.
- Allow these sections to be templated from user configuration, as simply as possible.
- Allow these sections to be passed-through from user configuration and user files.
- Ensure the configuration from the templated variables can't produce dumb configuration:
  If some variables are not matching HAProxy expected behaviour, an error has to be raised before the file is sent to the remote node for final validation.

## High level view of variables

* When ```haproxy_bind_on_non_local``` is set to True, the role will change
  the HAProxy host sysctls to allow HAProxy to bind on non-local ips.

* ```haproxy_globals```: This is an optional dict containing HAProxy's
  "global" configuration directive. This will be merged with:
  * the "per OS" standard global section. You can find it in code in vars, under the variable `haproxy_globals_osdefaults`
  * the global overrides recommended in this role (OS specitic) are under the variable `haproxy_globals_roledefaults`.

  This merge behaviour is defined in the role default variable: ```haproxy_globals_merged```.
  Override it as ``haproxy_globals_merged: "{{ haproxy_globals }}"`` to ignore the OS/role recommendations.

* Same behaviour happens for ```haproxy_defaults{,_osdefaults,_roledefaults,_merged}```

* HAProxy proxies are split in variables based on their top level configuration items:
  * ```haproxy_frontends```
  * ```haproxy_backends```
  * ```haproxy_listens```

## Fragments handling

The HAProxy "running" configuration is a generated as an assembly of fragments.
At the opposite of other roles, the fragments are stored _local_. This will allow full overrides.
The assembled file is validated on a temp file on the remote machine before overriding the production configuration.

Each fragement is generated based on the dicts explained above. (See also low level view of variables for more details).
To be more precise, each fragment corresponds to a single top level configuration section (i.e. one fragment per frontend, backend, listen, ...)

Here are a few variables linked to fragment handling:
* As HAProxy order configuration matters, fragments filename will be defined by default as <type>-<name>-<index>.
  This can be overriden by the deployer, by defining `fragment_name` (see below)
* 
  The HAProxy configuration files (fragments) are stored in a local folder on the ansible node before
  all the fragments are merged together. This local folder location is stored in ``haproxy_fragments_folder``.

* A default variable, named ``haproxy_fragments``, will list the default fragments to assemble together.
  This list will be automatically populated from the variables
  ``haproxy_frontends_all``, ``haproxy_backends_all``, ``haproxy_listens_all``, which are
  indirectly populated from respectively ``haproxy_frontends``, ``haproxy_backends``, and ``haproxy_listens``.
  (the ``_all`` are there for convenience, see section below).

  The ``haproxy_fragments`` ensures that HAProxy configuration is always up to date,
  the generation is idempotent, and the user doesn't need to define a variable to "clean the state".
  At the opposite of copying all the files in the assembly folder, which might contain outdated files.

  This feature is useful to bring an external file fragment, and assemble it with the rest.
  
  CAUTION: Any configuration file not appearing in the `haproxy_fragments` list will be deleted on the
  `haproxy_fragments_folder` on the local ansible machine!

## Low level view of variables

TODO

## Convenience tools

Here are some variables which will automatically create some frontend/backend/listen fragments.

### Stats

HAProxy has an internal way of gathering statistics, and can show it to the
user/deployer. Here are variables altering the stats behaviour.

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

## Distributions variables

These variables are defined per distribution, and define package names, paths
to binaries, binaries names, etc. They are stored in vars/ folder (one file
per distro).

## Other ansible default variables

* ```haproxy_packages``` is the list of packages that will be installed on
  the deployed system. By default, socat hatop and haproxy are installed
  on Ubuntu. The first two are useful for managing haproxy. You can override
  this list with your own packages.
  To extend the list, can define in your vars:
  ``haproxy_packages: "{{ _haproxy_packages + [ "myextrapackage" ] }}"

* ```haproxy_server_repo``` is the repo url used for installing a recent haproxy
  with packages.

* ```haproxy_server_configuration_folder``` is a variable holding the path
  to the haproxy configuration.

## Dependencies

None

## Example Playbook

    ---
    - hosts: all
      vars:
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


## License

Apache2

## Author Information

Jean-Philippe Evrard <open-source@a.spamming.party>
