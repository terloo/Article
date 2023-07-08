# unarchive
将ansible主机上的压缩包传到远程主机后解压到特定目录(copy=yes)或将远程主机上某个目录的压缩包解压到指定路径下(copy=no)

1. copy：默认为yes
2. remote_src：与copy功能一致且互斥
3. src：源路径，ansible主机路径或者远程主机路径
4. dest：远程主机的绝对路径
5. mod：解压后文件的权限


```yaml
- name: Unpacks an archive after (optionally) copying it from the local machine.
  unarchive:
      attributes:            # The attributes the resulting file or directory should have. To get supported flags look
                               at the man page for `chattr' on the target system. This
                               string should contain the attributes in the same order as
                               the one displayed by `lsattr'. The `=' operator is
                               assumed as default, otherwise `+' or `-' operators need
                               to be included in the string.
      copy:                  # If true, the file is copied from local 'master' to the target machine, otherwise, the
                               plugin will look for src archive at the target machine.
                               This option has been deprecated in favor of `remote_src'.
                               This option is mutually exclusive with `remote_src'.
      creates:               # If the specified absolute path (file or directory) already exists, this step will *not*
                               be run.
      decrypt:               # This option controls the autodecryption of source files using vault.
      dest:                  # (required) Remote absolute path where the archive should be unpacked.
      exclude:               # List the directory and file entries that you would like to exclude from the unarchive
                               action.
      extra_opts:            # Specify additional options by passing in an array. Each space-separated command-line
                               option should be a new element of the array. See
                               examples. Command-line options with multiple elements
                               must use multiple lines in the array, one for each
                               element.
      group:                 # Name of the group that should own the file/directory, as would be fed to `chown'.
      keep_newer:            # Do not replace existing files that are newer than files from the archive.
      list_files:            # If set to True, return the list of files that are contained in the tarball.
      mode:                  # The permissions the resulting file or directory should have. For those used to
                               `/usr/bin/chmod' remember that modes are actually octal
                               numbers. You must either add a leading zero so that
                               Ansible's YAML parser knows it is an octal number (like
                               `0644' or `01777') or quote it (like `'644'' or `'1777'')
                               so Ansible receives a string and can do its own
                               conversion from string into number. Giving Ansible a
                               number without following one of these rules will end up
                               with a decimal number which will have unexpected results.
                               As of Ansible 1.8, the mode may be specified as a
                               symbolic mode (for example, `u+rwx' or `u=rw,g=r,o=r').
      owner:                 # Name of the user that should own the file/directory, as would be fed to `chown'.
      remote_src:            # Set to `yes' to indicate the archived file is already on the remote system and not local
                               to the Ansible controller. This option is mutually
                               exclusive with `copy'.
      selevel:               # The level part of the SELinux file context. This is the MLS/MCS attribute, sometimes
                               known as the `range'. When set to `_default', it will use
                               the `level' portion of the policy if available.
      serole:                # The role part of the SELinux file context. When set to `_default', it will use the
                               `role' portion of the policy if available.
      setype:                # The type part of the SELinux file context. When set to `_default', it will use the
                               `type' portion of the policy if available.
      seuser:                # The user part of the SELinux file context. By default it uses the `system' policy, where
                               applicable. When set to `_default', it will use the
                               `user' portion of the policy if available.
      src:                   # (required) If `remote_src=no' (default), local path to archive file to copy to the
                               target server; can be absolute or relative. If
                               `remote_src=yes', path on the target server to existing
                               archive file to unpack. If `remote_src=yes' and `src'
                               contains `://', the remote machine will download the file
                               from the URL first. (version_added 2.0). This is only for
                               simple cases, for full download support use the [get_url]
                               module.
      unsafe_writes:         # Influence when to use atomic operation to prevent data corruption or inconsistent reads
                               from the target file. By default this module uses atomic
                               operations to prevent data corruption or inconsistent
                               reads from the target files, but sometimes systems are
                               configured or just broken in ways that prevent this. One
                               example is docker mounted files, which cannot be updated
                               atomically from inside the container and can only be
                               written in an unsafe manner. This option allows Ansible
                               to fall back to unsafe methods of updating files when
                               atomic operations fail (however, it doesn't force Ansible
                               to perform unsafe writes). IMPORTANT! Unsafe writes are
                               subject to race conditions and can lead to data
                               corruption.
      validate_certs:        # This only applies if using a https URL as the source of the file. This should only set
                               to `no' used on personally controlled sites using self-
                               signed certificate. Prior to 2.2 the code worked as if
                               this was set to `yes'.
```