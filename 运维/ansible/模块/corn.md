# corn
在远程主机上定义计划任务，分时日月周

1. name：计划任务名
2. job：远程主机上的计划任务脚本
3. disbaled：是否启用
4. state：为absent时表示删除

```yaml
- name: Manage cron.d and crontab entries
  cron:
      backup:                # If set, create a backup of the crontab before it is modified. The location of the backup
                               is returned in the `backup_file' variable by this module.
      cron_file:             # If specified, uses this file instead of an individual user's crontab. If this is a
                               relative path, it is interpreted with respect to
                               `/etc/cron.d'. If it is absolute, it will typically be
                               `/etc/crontab'. Many linux distros expect (and some
                               require) the filename portion to consist solely of upper-
                               and lower-case letters, digits, underscores, and hyphens.
                               To use the `cron_file' parameter you must specify the
                               `user' as well.
      day:                   # Day of the month the job should run ( 1-31, *, */2, etc )
      disabled:              # If the job should be disabled (commented out) in the crontab. Only has effect if
                               `state=present'.
      env:                   # If set, manages a crontab's environment variable. New variables are added on top of
                               crontab. `name' and `value' parameters are the name and
                               the value of environment variable.
      hour:                  # Hour when the job should run ( 0-23, *, */2, etc )
      insertafter:           # Used with `state=present' and `env'. If specified, the environment variable will be
                               inserted after the declaration of specified environment
                               variable.
      insertbefore:          # Used with `state=present' and `env'. If specified, the environment variable will be
                               inserted before the declaration of specified environment
                               variable.
      job:                   # The command to execute or, if env is set, the value of environment variable. The command
                               should not contain line breaks. Required if
                               `state=present'.
      minute:                # Minute when the job should run ( 0-59, *, */2, etc )
      month:                 # Month of the year the job should run ( 1-12, *, */2, etc )
      name:                  # Description of a crontab entry or, if env is set, the name of environment variable.
                               Required if `state=absent'. Note that if name is not set
                               and `state=present', then a new crontab entry will always
                               be created, regardless of existing ones. This parameter
                               will always be required in future releases.
      reboot:                # If the job should be run at reboot. This option is deprecated. Users should use
                               special_time.
      special_time:          # Special time specification nickname.
      state:                 # Whether to ensure the job or environment variable is present or absent.
      user:                  # The specific user whose crontab should be modified. When unset, this parameter defaults
                               to using `root'.
      weekday:               # Day of the week that the job should run ( 0-6 for Sunday-Saturday, *, etc )
```