# archive
压缩远程主机上的文件或目录

```yaml
- name: Creates a compressed archive of one or more files or trees
  archive:
      attributes:            # The attributes the resulting file or directory should have. To get supported flags look
                               at the man page for `chattr' on the target system. This
                               string should contain the attributes in the same order as
                               the one displayed by `lsattr'. The `=' operator is
                               assumed as default, otherwise `+' or `-' operators need
                               to be included in the string.
      dest:                  # The file name of the destination archive. This is required when `path' refers to
                               multiple files by either specifying a glob, a directory
                               or multiple paths in a list.
      exclude_path:          # Remote absolute path, glob, or list of paths or globs for the file or files to exclude
                               from the archive.
      force_archive:         # Allow you to force the module to treat this as an archive even if only a single file is
                               specified. By default behaviour is maintained. i.e A when
                               a single file is specified it is compressed only (not
                               archived).
      format:                # The type of compression to use. Support for xz was added in Ansible 2.5.
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
      owner:                 # Name of the user that should own the file/directory, as would be fed to `chown'.
      path:                  # (required) Remote absolute path, glob, or list of paths or globs for the file or files
                               to compress or archive.
      remove:                # Remove any added source files and trees after adding to archive.
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