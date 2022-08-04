A simple yet powerful role to install/configure HAProxy
=======================================================


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

This role:
- Allows HAProxy sections to be templated from user variables, as simply as possible.
- Allows HAProxy sections to be passed-through from user configuration and user files.
- Ensures the configuration from the templated variables can't produce dumb configuration:
  If some variables are not matching HAProxy expected behaviour, an error has
  to be raised before the file is sent to the remote node for final validation.
- Is close to the default OS specific globals. If a variable is different than the OS
  standard global, it will be documented in a git commit history, and easy to follow.

## Variables

### Variables related to OS

* When ```haproxy_bind_on_non_local``` is set to True, the role will change
  the HAProxy host sysctls to allow HAProxy to bind on non-local IPs.
  It is possible to override it per address family (IPv4/IPv6) too.

* ```haproxy_packages``` is the list of packages that will be installed on
  the deployed system. By default, only HAProxy is installed.
  Extra packages might be useful to manage HAProxy, for example ``nc``,
  ``socat``, ``hatop``, or ``haproxyctl``.

  To install additional packages, set ``haproxy_extrapackages`` like this:
  ```
  haproxy_extrapackages:
    - "nc"
  ```
  This will install nc on top of haproxy.

  Note: We already provide a convenience variable named ``_haproxy_extrapackages``
  containing ``hatop`` and ``socat``. To install them, set
  ``` haproxy_role_extrapackages: true ```.
  This will automatically add socat and hatop to the packages to install on
  haproxy nodes.

* ``haproxy_packages_state`` clarifies the intent behind the package installation.
  By default it is only intended to have HAProxy installed. It is possible to ensure
  always the latest version of the package is installed by setting:
  ``haproxy_packages_state: "latest"``

* Distribution specific variables (for example package names, paths to binaries,
  binary names, service names) are stored in vars/ folder (one file per distro).
  If this doesn't work for a case, please submit a PR.

### Variables related to HAProxy configuration

* ```haproxy_globals```: This is an optional dict containing HAProxy's
  "global" configuration directive. This will be merged with:
  * the "per OS" standard global section. You can find it in code in vars, under the variable `haproxy_globals_osdefaults`
  * the global overrides recommended in this role (OS specitic) are under the variable `haproxy_globals_roledefaults`.

  This merge behaviour is defined in the role default variable: ```haproxy_globals_merged```.
  Override it as ``haproxy_globals_merged: "{{ haproxy_globals }}"`` to ignore the OS/role recommendations.

* Same behaviour happens for ```haproxy_defaults{,_osdefaults,_roledefaults,_merged}```.
  Please note that it is possible to have multiple default sections. This only applies for the last unnamed "default" section
  from the assembled fragments.

* HAProxy named defaults will be generated from ```haproxy_named_defaults```

* HAProxy proxies are split in variables based on their top level configuration items:
  * ```haproxy_frontends```
  * ```haproxy_backends```
  * ```haproxy_listens```

### Variables related to fragments handling for HAProxy configuration

The HAProxy "running" configuration is a generated as an assembly of fragments.
At the opposite of other roles, the fragments are stored **locally** in ``haproxy_fragments_folder``. This will allow full overrides.
The assembled file is validated on a temp file on the remote machine before overriding the production configuration.

Each fragement is generated based on the dicts explained above. (See also low level view of variables for more details).
To be more precise, each fragment corresponds to a single top level configuration section (i.e. one fragment per frontend, backend, listen, ...)

Here are a few variables linked to fragment handling:

* As HAProxy order configuration matters, fragments filename will be defined by default as the following:
  * 001_global for globals
  * 002_defaults for unnamed defaults
  * 002_defaults_<name> for named defaults
  * 003_listen_<name> for listens
  * 004_frontend_<name> for frontends
  * 005_backend_<name> for backends
  * 006_<type>_<name> for the following types:
    * userlist
    * cache
    * http-errors
    * mailers
    * resolvers
    * fcgi-app
    * peers
    * aggregations
    * program
    * dynamic-update
    * ring

* The HAProxy configuration files (fragments) are stored in a local folder on the ansible node before
  all the fragments are merged together. This local folder location is stored in ``haproxy_fragments_folder``.
  This defaults to ``/var/cache/ansible/haproxy-role/``. This folder has been chosen to respect the FHS 3.0.
  See also: https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s05.html .

* A user can bring its own files by defining their path (they will be copied using the copy module) or their
  content. For that, the user needs to define ``haproxy_user_fragments``.
  Example:
  ```
  haproxy_user_fragments:
    - src: <filename>
    - content: "# haproxy valid content"
  ```
  Of course content can be a longer string (use the ``|`` sign for example) or a variable (use a lookup plugin for example).

  Destination fragment filename will always be defined as 000_user_<index>.
  Changing the order of your list in user_fragments will change the haproxy configuration by changing its order.

  Side notes about user fragments:
  * They are not validated by this role configuration, only validated by haproxy
  * haproxy doesn't allow two "global" sections.
    To have a user fragment for those sections, make sure to override the role generation of globals
    by overriding ``haproxy_globals_merged`` as follow:  ``haproxy_globals_merged: {}``
  * haproxy allows multiple defaults, including multiple unnamed ones. The last one before any configuration
    will be applied. This is NOT SUPPPORTED BY THIS ROLE.
    This role choses to work in an explicit manner, rather than purely implicit. As
    the order of the defaults is always on before the proxies, it is impossible to have two unnamed defaults
    for a specific proxy. To fix this, be explicit in your defaults by naming them. If you still want to
    avoid the role provided unnamed defaults, make sure to override the role generation of unnamed defaults
    by overriding the variable ``haproxy_defaults_merged`` as follow:  ``haproxy_defaults_merged: {}``.

* A default variable, named ``haproxy_fragments``, will list the fragments to assemble together.
  This list will be by default populated as following:
  * The path to the 001_global file (as explained above, this file is templated from `haproxy_globals_merged`)
  * The path to the 002_defaults file (as explained above, this file is templated from `haproxy_defaults_merged`)
  * The paths to each frontend/backend/listen/... fragments (as explained above, generated from)
    ``haproxy_{frontends,backends,listens}`` and from convenience variables.
  * The paths to each user fragment 000_user_*

  The ``haproxy_fragments`` ensures that HAProxy configuration is always up to date,
  the generation is idempotent, and the user doesn't need to define a variable to "clean the state".
  This is more robust than assembling all the files from the ``haproxy_fragments_folder``, as
  the latter might is not defined as desired state/might contain outdated files.
  
  CAUTION: Any configuration file in the `haproxy_fragments_folder` AND not in the
  `haproxy_fragments` list will be deleted in the `haproxy_fragments_folder` on
   the local ansible machine before sending to the haproxy node.

### Variables related to convenience: Useful HAProxy fragments

Here are some variables which will automatically create some frontend/backend/listen fragments.

#### Stats website

TODO

## Dependencies

None

## Example Playbook

TODO, particularly if other repos are necessary.

## License

Apache2

## Author Information

Jean-Philippe Evrard <open-source@a.spamming.party>
