# file

在远程主机上删除、创建、修改属性文件或目录

1. path：目标路径
2. state：对path进行的操作类型 absent表示删除

```yaml
- name: Manage files and file properties
  file:
      access_time:           # This parameter indicates the time the file's access time should be set to. Should be
                               `preserve' when no modification is required,
                               `YYYYMMDDHHMM.SS' when using default time format, or
                               `now'. Default is `None' meaning that `preserve' is the
                               default for `state=[file,directory,link,hard]' and `now'
                               is default for `state=touch'.
      access_time_format:    # When used with `access_time', indicates the time format that must be used. Based on
                               default Python format (see time.strftime doc).
      attributes:            # The attributes the resulting file or directory should have. To get supported flags look
                               at the man page for `chattr' on the target system. This
                               string should contain the attributes in the same order as
                               the one displayed by `lsattr'. The `=' operator is
                               assumed as default, otherwise `+' or `-' operators need
                               to be included in the string.
      follow:                # This flag indicates that filesystem links, if they exist, should be followed. Previous
                               to Ansible 2.5, this was `no' by default.
      force:                 # Force the creation of the symlinks in two cases: the source file does not exist (but
                               will appear later); the destination exists and is a file
                               (so, we need to unlink the `path' file and create symlink
                               to the `src' file in place of it).
      group:                 # Name of the group that should own the file/directory, as would be fed to `chown'.
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
      modification_time:     # This parameter indicates the time the file's modification time should be set to. Should
                               be `preserve' when no modification is required,
                               `YYYYMMDDHHMM.SS' when using default time format, or
                               `now'. Default is None meaning that `preserve' is the
                               default for `state=[file,directory,link,hard]' and `now'
                               is default for `state=touch'.
      modification_time_format:   # When used with `modification_time', indicates the time format that must be used. Bas
                               on default Python format (see time.strftime doc).
      owner:                 # Name of the user that should own the file/directory, as would be fed to `chown'.
      path:                  # (required) Path to the file being managed.
      recurse:               # Recursively set the specified file attributes on directory contents. This applies only
                               when `state' is set to `directory'.
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
      src:                   # Path of the file to link to. This applies only to `state=link' and `state=hard'. For
                               `state=link', this will also accept a non-existing path.
                               Relative paths are relative to the file being created
                               (`path') which is how the Unix command `ln -s SRC DEST'
                               treats relative paths.
      state:                 # If `absent', directories will be recursively deleted, and files or symlinks will be
                               unlinked. In the case of a directory, if `diff' is
                               declared, you will see the files and folders deleted
                               listed under `path_contents'. Note that `absent' will not
                               cause `file' to fail if the `path' does not exist as the
                               state did not change. If `directory', all intermediate
                               subdirectories will be created if they do not exist.
                               Since Ansible 1.7 they will be created with the supplied
                               permissions. If `file', without any other options this
                               works mostly as a 'stat' and will return the current
                               state of `path'. Even with other options (i.e `mode'),
                               the file will be modified but will NOT be created if it
                               does not exist; see the `touch' value or the [copy] or
                               [template] module if you want that behavior. If `hard',
                               the hard link will be created or changed. If `link', the
                               symbolic link will be created or changed. If `touch' (new
                               in 1.4), an empty file will be created if the `path' does
                               not exist, while an existing file or directory will
                               receive updated file access and modification times
                               (similar to the way `touch' works from the command line).
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
```