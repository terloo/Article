# copy
复制ansible主机文件/目录到远程主机

1. content：直接将参数值新建为一个文件
2. backup：如果文件存在，先进行备份
3. dest：目标文件目录
4. remote_src：yes时将src视作远程主机的目录而不是ansible主机

```yaml
- name: Copy files to remote locations
  copy:
      attributes:            # The attributes the resulting file or directory should have. To get supported flags look
                               at the man page for `chattr' on the target system. This
                               string should contain the attributes in the same order as
                               the one displayed by `lsattr'. The `=' operator is
                               assumed as default, otherwise `+' or `-' operators need
                               to be included in the string.
      backup:                # Create a backup file including the timestamp information so you can get the original
                               file back if you somehow clobbered it incorrectly.
      checksum:              # SHA1 checksum of the file being transferred. Used to validate that the copy of the file
                               was successful. If this is not provided, ansible will use
                               the local calculated checksum of the src file.
      content:               # When used instead of `src', sets the contents of a file directly to the specified value.
                               Works only when `dest' is a file. Creates the file if it
                               does not exist. For advanced formatting or if `content'
                               contains a variable, use the [template] module.
      decrypt:               # This option controls the autodecryption of source files using vault.
      dest:                  # (required) Remote absolute path where the file should be copied to. If `src' is a
                               directory, this must be a directory too. If `dest' is a
                               non-existent path and if either `dest' ends with "/" or
                               `src' is a directory, `dest' is created. If `dest' is a
                               relative path, the starting directory is determined by
                               the remote host. If `src' and `dest' are files, the
                               parent directory of `dest' is not created and the task
                               fails if it does not already exist.
      directory_mode:        # When doing a recursive copy set the mode for the directories. If this is not set we will
                               use the system defaults. The mode is only set on
                               directories which are newly created, and will not affect
                               those that already existed.
      follow:                # This flag indicates that filesystem links in the destination, if they exist, should be
                               followed.
      force:                 # Influence whether the remote file must always be replaced. If `yes', the remote file
                               will be replaced when contents are different than the
                               source. If `no', the file will only be transferred if the
                               destination does not exist. Alias `thirsty' has been
                               deprecated and will be removed in 2.13.
      group:                 # Name of the group that should own the file/directory, as would be fed to `chown'.
      local_follow:          # This flag indicates that filesystem links in the source tree, if they exist, should be
                               followed.
      mode:                  # The permissions of the destination file or directory. For those used to `/usr/bin/chmod'
                               remember that modes are actually octal numbers. You must
                               either add a leading zero so that Ansible's YAML parser
                               knows it is an octal number (like `0644' or `01777')or
                               quote it (like `'644'' or `'1777'') so Ansible receives a
                               string and can do its own conversion from string into
                               number. Giving Ansible a number without following one of
                               these rules will end up with a decimal number which will
                               have unexpected results. As of Ansible 1.8, the mode may
                               be specified as a symbolic mode (for example, `u+rwx' or
                               `u=rw,g=r,o=r'). As of Ansible 2.3, the mode may also be
                               the special string `preserve'. `preserve' means that the
                               file will be given the same permissions as the source
                               file.
      owner:                 # Name of the user that should own the file/directory, as would be fed to `chown'.
      remote_src:            # Influence whether `src' needs to be transferred or already is present remotely. If `no',
                               it will search for `src' at originating/master machine.
                               If `yes' it will go to the remote/target machine for the
                               `src'. `remote_src' supports recursive copying as of
                               version 2.8. `remote_src' only works with `mode=preserve'
                               as of version 2.6.
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
      src:                   # Local path to a file to copy to the remote server. This can be absolute or relative. If
                               path is a directory, it is copied recursively. In this
                               case, if path ends with "/", only inside contents of that
                               directory are copied to destination. Otherwise, if it
                               does not end with "/", the directory itself with all
                               contents is copied. This behavior is similar to the
                               `rsync' command line tool.
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
      validate:              # The validation command to run before copying into place. The path to the file to
                               validate is passed in via '%s' which must be present as
                               in the examples below. The command is passed securely so
                               shell features like expansion and pipes will not work.
```