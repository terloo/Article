# yum
管理软件包

1. name：要安装的软件包名
2. state：表示安装或者卸载软件，默认安装

```yaml
- name: Manages packages with the `yum' package manager
  yum:
      allow_downgrade:       # Specify if the named package and version is allowed to downgrade a maybe already
                               installed higher version of that package. Note that
                               setting allow_downgrade=True can make this module behave
                               in a non-idempotent way. The task could end up with a set
                               of packages that does not match the complete list of
                               specified packages to install (because dependencies
                               between the downgraded package and others can cause
                               changes to the packages which were in the earlier
                               transaction).
      autoremove:            # If `yes', removes all "leaf" packages from the system that were originally installed as
                               dependencies of user-installed packages but which are no
                               longer required by any such package. Should be used alone
                             # or when state is `absent' NOTE: This feature requires yum >= 3.4.3 (RHEL/CentOS 7+)
      bugfix:                # If set to `yes', and `state=latest' then only installs updates that have been marked
                               bugfix related.
      conf_file:             # The remote yum configuration file to use for the transaction.
      disable_excludes:      # Disable the excludes defined in YUM config files. If set to `all', disables all
                               excludes. If set to `main', disable excludes defined in
                               [main] in yum.conf. If set to `repoid', disable excludes
                               defined for given repo id.
      disable_gpg_check:     # Whether to disable the GPG checking of signatures of packages being installed. Has an
                               effect only if state is `present' or `latest'.
      disable_plugin:        # `Plugin' name to disable for the install/update operation. The disabled plugins will not
                               persist beyond the transaction.
      disablerepo:           # `Repoid' of repositories to disable for the install/update operation. These repos will
                               not persist beyond the transaction. When specifying
                               multiple repos, separate them with a `","'. As of Ansible
                               2.7, this can alternatively be a list instead of `","'
                               separated string
      download_dir:          # Specifies an alternate directory to store packages. Has an effect only if
                               `download_only' is specified.
      download_only:         # Only download the packages, do not install them.
      enable_plugin:         # `Plugin' name to enable for the install/update operation. The enabled plugin will not
                               persist beyond the transaction.
      enablerepo:            # `Repoid' of repositories to enable for the install/update operation. These repos will
                               not persist beyond the transaction. When specifying
                               multiple repos, separate them with a `","'. As of Ansible
                               2.7, this can alternatively be a list instead of `","'
                               separated string
      exclude:               # Package name(s) to exclude when state=present, or latest
      install_weak_deps:     # Will also install all packages linked by a weak dependency relation. NOTE: This feature
                               requires yum >= 4 (RHEL/CentOS 8+)
      installroot:           # Specifies an alternative installroot, relative to which all packages will be installed.
      list:                  # Package name to run the equivalent of yum list --show-duplicates <package> against. In
                               addition to listing packages, use can also list the
                               following: `installed', `updates', `available' and
                               `repos'. This parameter is mutually exclusive with
                               `name'.
      lock_timeout:          # Amount of time to wait for the yum lockfile to be freed.
      name:                  # A package name or package specifier with version, like `name-1.0'. If a previous version
                               is specified, the task also needs to turn
                               `allow_downgrade' on. See the `allow_downgrade'
                               documentation for caveats with downgrading packages. When
                               using state=latest, this can be `'*'' which means run
                               `yum -y update'. You can also pass a url or a local path
                               to a rpm file (using state=present). To operate on
                               several packages this can accept a comma separated string
                               of packages or (as of 2.0) a list of packages.
      releasever:            # Specifies an alternative release from which all packages will be installed.
      security:              # If set to `yes', and `state=latest' then only installs updates that have been marked
                               security related.
      skip_broken:           # Skip packages with broken dependencies(devsolve) and are causing problems.
      state:                 # Whether to install (`present' or `installed', `latest'), or remove (`absent' or
                               `removed') a package. `present' and `installed' will
                               simply ensure that a desired package is installed.
                               `latest' will update the specified package if it's not of
                               the latest available version. `absent' and `removed' will
                               remove the specified package. Default is `None', however
                               in effect the default action is `present' unless the
                               `autoremove' option is enabled for this module, then
                               `absent' is inferred.
      update_cache:          # Force yum to check if cache is out of date and redownload if needed. Has an effect only
                               if state is `present' or `latest'.
      update_only:           # When using latest, only update installed packages. Do not install packages. Has an
                               effect only if state is `latest'
      use_backend:           # This module supports `yum' (as it always has), this is known as `yum3'/`YUM3'/`yum-
                               deprecated' by upstream yum developers. As of Ansible
                               2.7+, this module also supports `YUM4', which is the "new
                               yum" and it has an `dnf' backend. By default, this module
                               will select the backend based on the `ansible_pkg_mgr'
                               fact.
      validate_certs:        # This only applies if using a https url as the source of the rpm. e.g. for localinstall.
                               If set to `no', the SSL certificates will not be
                               validated. This should only set to `no' used on
                               personally controlled sites using self-signed
                               certificates as it avoids verifying the source site.
                               Prior to 2.1 the code worked as if this was set to `yes'.
```