# hostname
管理远程主机的hostname

```yaml
- name: Manage hostname
  hostname:
      name:                  # (required) Name of the host
      use:                   # Which strategy to use to update the hostname. If not set we try to autodetect, but this
                               can be problematic, specially with containers as they can
                               present misleading information.
```