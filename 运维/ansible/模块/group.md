# group
管理组

1. name：组名
2. state：删除或创建

```yaml
- name: Add or remove groups
  group:
      gid:                   # Optional `GID' to set for the group.
      local:                 # Forces the use of "local" command alternatives on platforms that implement it. This is
                               useful in environments that use centralized
                               authentication when you want to manipulate the local
                               groups. (e.g. it uses `lgroupadd' instead of `groupadd').
                               This requires that these commands exist on the targeted
                               host, otherwise it will be a fatal error.
      name:                  # (required) Name of the group to manage.
      non_unique:            # This option allows to change the group ID to a non-unique value. Requires `gid'. Not
                               supported on macOS or BusyBox distributions.
      state:                 # Whether the group should be present or not on the remote host.
      system:                # If `yes', indicates that the group created is a system group.
```