# user
管理用户

1. name：用户名
2. home：是否创建家目录

```yaml
- name: Manage user accounts
  user:
      append:                # If `yes', add the user to the groups specified in `groups'. If `no', user will only be
                               added to the groups specified in `groups', removing them
                               from all other groups. Mutually exclusive with `local'
      authorization:         # Sets the authorization of the user. Does nothing when used with other platforms. Can set
                               multiple authorizations using comma separation. To delete
                               all authorizations, use `authorization='''. Currently
                               supported on Illumos/Solaris.
      comment:               # Optionally sets the description (aka `GECOS') of user account.
      create_home:           # Unless set to `no', a home directory will be made for the user when the account is
                               created or if the home directory does not exist. Changed
                               from `createhome' to `create_home' in Ansible 2.5.
      expires:               # An expiry time for the user in epoch, it will be ignored on platforms that do not
                               support this. Currently supported on GNU/Linux, FreeBSD,
                               and DragonFlyBSD. Since Ansible 2.6 you can remove the
                               expiry time specify a negative value. Currently supported
                               on GNU/Linux and FreeBSD.
      force:                 # This only affects `state=absent', it forces removal of the user and associated
                               directories on supported platforms. The behavior is the
                               same as `userdel --force', check the man page for
                               `userdel' on your system for details and support. When
                               used with `generate_ssh_key=yes' this forces an existing
                               key to be overwritten.
      generate_ssh_key:      # Whether to generate a SSH key for the user in question. This will *not* overwrite an
                               existing SSH key unless used with `force=yes'.
      group:                 # Optionally sets the user's primary group (takes a group name).
      groups:                # List of groups user will be added to. When set to an empty string `''', the user is
                               removed from all groups except the primary group. Before
                               Ansible 2.3, the only input format allowed was a comma
                               separated string. Mutually exclusive with `local'
      hidden:                # macOS only, optionally hide the user from the login window and system preferences. The
                               default will be `yes' if the `system' option is used.
      home:                  # Optionally set the user's home directory.
      local:                 # Forces the use of "local" command alternatives on platforms that implement it. This is
                               useful in environments that use centralized
                               authentification when you want to manipulate the local
                               users (i.e. it uses `luseradd' instead of `useradd').
                               This will check `/etc/passwd' for an existing account
                               before invoking commands. If the local account database
                               exists somewhere other than `/etc/passwd', this setting
                               will not work properly. This requires that the above
                               commands as well as `/etc/passwd' must exist on the
                               target host, otherwise it will be a fatal error. Mutually
                               exclusive with `groups' and `append'
      login_class:           # Optionally sets the user's login class, a feature of most BSD OSs.
      move_home:             # If set to `yes' when used with `home: ', attempt to move the user's old home directory
                               to the specified directory if it isn't there already and
                               the old home exists.
      name:                  # (required) Name of the user to create, remove or modify.
      non_unique:            # Optionally when used with the -u option, this option allows to change the user ID to a
                               non-unique value.
      password:              # Optionally set the user's password to this crypted value. On macOS systems, this value
                               has to be cleartext. Beware of security issues. To create
                               a disabled account on Linux systems, set this to `'!'' or
                               `'*''. To create a disabled account on OpenBSD, set this
                               to `'*************''. See
                               https://docs.ansible.com/ansible/faq.html#how-do-i
                               -generate-encrypted-passwords-for-the-user-module for
                               details on various ways to generate these password
                               values.
      password_lock:         # Lock the password (`usermod -L', `usermod -U', `pw lock'). Implementation differs by
                               platform. This option does not always mean the user
                               cannot login using other methods. This option does not
                               disable the user, only lock the password. This must be
                               set to `False' in order to unlock a currently locked
                               password. The absence of this parameter will not unlock a
                               password. Currently supported on Linux, FreeBSD,
                               DragonFlyBSD, NetBSD, OpenBSD.
      profile:               # Sets the profile of the user. Does nothing when used with other platforms. Can set
                               multiple profiles using comma separation. To delete all
                               the profiles, use `profile='''. Currently supported on
                               Illumos/Solaris.
      remove:                # This only affects `state=absent', it attempts to remove directories associated with the
                               user. The behavior is the same as `userdel --remove',
                               check the man page for details and support.
      role:                  # Sets the role of the user. Does nothing when used with other platforms. Can set multiple
                               roles using comma separation. To delete all roles, use
                               `role='''. Currently supported on Illumos/Solaris.
      seuser:                # Optionally sets the seuser type (user_u) on selinux enabled systems.
      shell:                 # Optionally set the user's shell. On macOS, before Ansible 2.5, the default shell for
                               non-system users was `/usr/bin/false'. Since Ansible 2.5,
                               the default shell for non-system users on macOS is
                               `/bin/bash'. On other operating systems, the default
                               shell is determined by the underlying tool being used.
                               See Notes for details.
      skeleton:              # Optionally set a home skeleton directory. Requires `create_home' option!
      ssh_key_bits:          # Optionally specify number of bits in SSH key to create.
      ssh_key_comment:       # Optionally define the comment for the SSH key.
      ssh_key_file:          # Optionally specify the SSH key filename. If this is a relative filename then it will be
                               relative to the user's home directory. This parameter
                               defaults to `.ssh/id_rsa'.
      ssh_key_passphrase:    # Set a passphrase for the SSH key. If no passphrase is provided, the SSH key will default
                               to having no passphrase.
      ssh_key_type:          # Optionally specify the type of SSH key to generate. Available SSH key types will depend
                               on implementation present on target host.
      state:                 # Whether the account should exist or not, taking action if the state is different from
                               what is stated.
      system:                # When creating an account `state=present', setting this to `yes' makes the user a system
                               account. This setting cannot be changed on existing
                               users.
      uid:                   # Optionally sets the `UID' of the user.
      update_password:       # `always' will update passwords if they differ. `on_create' will only set the password
                               for newly created users.
```