# replace
类似于sed，使用正则表达式修改文件

1. path：需要修改的文件路径
2. regexp：正则表达式
3. replace：将正则表达式匹配到的文本替换为该文本

```yaml
- name: Replace all instances of a particular string in a file using a back-referenced regular expression
  replace:
      after:                 # If specified, only content after this match will be replaced/removed. Can be used in
                               combination with `before'. Uses Python regular
                               expressions; see
                               http://docs.python.org/2/library/re.html. Uses DOTALL,
                               which means the `.' special character `can match
                               newlines'.
      attributes:            # The attributes the resulting file or directory should have. To get supported flags look
                               at the man page for `chattr' on the target system. This
                               string should contain the attributes in the same order as
                               the one displayed by `lsattr'. The `=' operator is
                               assumed as default, otherwise `+' or `-' operators need
                               to be included in the string.
      backup:                # Create a backup file including the timestamp information so you can get the original
                               file back if you somehow clobbered it incorrectly.
      before:                # If specified, only content before this match will be replaced/removed. Can be used in
                               combination with `after'. Uses Python regular
                               expressions; see
                               http://docs.python.org/2/library/re.html. Uses DOTALL,
                               which means the `.' special character `can match
                               newlines'.
      encoding:              # The character encoding for reading and writing the file.
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
      others:                # All arguments accepted by the [file] module also work here.
      owner:                 # Name of the user that should own the file/directory, as would be fed to `chown'.
      path:                  # (required) The file to modify. Before Ansible 2.3 this option was only usable as `dest',
                               `destfile' and `name'.
      regexp:                # (required) The regular expression to look for in the contents of the file. Uses Python
                               regular expressions; see
                               http://docs.python.org/2/library/re.html. Uses MULTILINE
                               mode, which means `^' and `$' match the beginning and end
                               of the file, as well as the beginning and end
                               respectively of `each line' of the file. Does not use
                               DOTALL, which means the `.' special character matches any
                               character `except newlines'. A common mistake is to
                               assume that a negated character set like `[^#]' will also
                               not match newlines. In order to exclude newlines, they
                               must be added to the set like `[^#\n]'. Note that, as of
                               Ansible 2.0, short form tasks should have any escape
                               sequences backslash-escaped in order to prevent them
                               being parsed as string literal escapes. See the examples.
      replace:               # The string to replace regexp matches. May contain backreferences that will get expanded
                               with the regexp capture groups if the regexp matches. If
                               not set, matches are removed entirely. Backreferences can
                               be used ambiguously like `\1', or explicitly like
                               `\g<1>'.
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
      validate:              # The validation command to run before copying into place. The path to the file to
                               validate is passed in via '%s' which must be present as
                               in the examples below. The command is passed securely so
                               shell features like expansion and pipes will not work.
```