# service
实现服务管理

1. state：started开始服务，stopped停止服务，reloaded重载配置，restarted重启
2. enable：是否开机启动


```yaml
- name: Manage services
  service:
      arguments:             # Additional arguments provided on the command line.
      enabled:               # Whether the service should start on boot. *At least one of state and enabled are
                               required.*
      name:                  # (required) Name of the service.
      pattern:               # If the service does not respond to the status command, name a substring to look for as
                               would be found in the output of the `ps' command as a
                               stand-in for a status result. If the string is found, the
                               service will be assumed to be started.
      runlevel:              # For OpenRC init scripts (e.g. Gentoo) only. The runlevel that this service belongs to.
      sleep:                 # If the service is being `restarted' then sleep this many seconds between the stop and
                               start command. This helps to work around badly-behaving
                               init scripts that exit immediately after signaling a
                               process to stop. Not all service managers support sleep,
                               i.e when using systemd this setting will be ignored.
      state:                 # `started'/`stopped' are idempotent actions that will not run commands unless necessary.
                               `restarted' will always bounce the service. `reloaded'
                               will always reload. *At least one of state and enabled
                               are required.* Note that reloaded will start the service
                               if it is not already started, even if your chosen init
                               system wouldn't normally.
      use:                   # The service module actually uses system specific modules, normally through auto
                               detection, this setting can force a specific module.
                               Normally it uses the value of the 'ansible_service_mgr'
                               fact and falls back to the old 'service' module when none
                               matching is found.
```