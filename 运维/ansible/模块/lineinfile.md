# lineinfile
按行修改文件

1. path：指定要修改的文件路径
2. regexp：正则表达式，匹配到的行将会被整行替换
3. line：新的行
4. state：absent时代表删除行

```yaml
- name: Manage lines in text files
  lineinfile:
      attributes:            # The attributes the resulting file or directory should have. To get supported flags look
                               at the man page for `chattr' on the target system. This
                               string should contain the attributes in the same order as
                               the one displayed by `lsattr'. The `=' operator is
                               assumed as default, otherwise `+' or `-' operators need
                               to be included in the string.
      backrefs:              # Used with `state=present'. If set, `line' can contain backreferences (both positional
                               and named) that will get populated if the `regexp'
                               matches. This parameter changes the operation of the
                               module slightly; `insertbefore' and `insertafter' will be
                               ignored, and if the `regexp' does not match anywhere in
                               the file, the file will be left unchanged. If the
                               `regexp' does match, the last matching line will be
                               replaced by the expanded line parameter.
      backup:                # Create a backup file including the timestamp information so you can get the original
                               file back if you somehow clobbered it incorrectly.
      create:                # Used with `state=present'. If specified, the file will be created if it does not already
                               exist. By default it will fail if the file is missing.
      firstmatch:            # Used with `insertafter' or `insertbefore'. If set, `insertafter' and `insertbefore' will
                               work with the first line that matches the given regular
                               expression.
      group:                 # Name of the group that should own the file/directory, as would be fed to `chown'.
      insertafter:           # Used with `state=present'. If specified, the line will be inserted after the last match
                               of specified regular expression. If the first match is
                               required, use(firstmatch=yes). A special value is
                               available; `EOF' for inserting the line at the end of the
                               file. If specified regular expression has no matches, EOF
                               will be used instead. If `insertbefore' is set, default
                               value `EOF' will be ignored. If regular expressions are
                               passed to both `regexp' and `insertafter', `insertafter'
                               is only honored if no match for `regexp' is found. May
                               not be used with `backrefs' or `insertbefore'.
      insertbefore:          # Used with `state=present'. If specified, the line will be inserted before the last match
                               of specified regular expression. If the first match is
                               required, use `firstmatch=yes'. A value is available;
                               `BOF' for inserting the line at the beginning of the
                               file. If specified regular expression has no matches, the
                               line will be inserted at the end of the file. If regular
                               expressions are passed to both `regexp' and
                               `insertbefore', `insertbefore' is only honored if no
                               match for `regexp' is found. May not be used with
                               `backrefs' or `insertafter'.
      line:                  # The line to insert/replace into the file. Required for `state=present'. If `backrefs' is
                               set, may contain backreferences that will get expanded
                               with the `regexp' capture groups if the regexp matches.
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
      regexp:                # The regular expression to look for in every line of the file. For `state=present', the
                               pattern to replace if found. Only the last line found
                               will be replaced. For `state=absent', the pattern of the
                               line(s) to remove. If the regular expression is not
                               matched, the line will be added to the file in keeping
                               with `insertbefore' or `insertafter' settings. When
                               modifying a line the regexp should typically match both
                               the initial state of the line as well as its state after
                               replacement by `line' to ensure idempotence. Uses Python
                               regular expressions. See
                               http://docs.python.org/2/library/re.html.
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
      state:                 # Whether the line should be there or not.
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