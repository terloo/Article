# setup
收集远程主机相关的配置信息

1. gather_subset：指定需要收集的内容，不指定时代表手机全部信息
2. filter：仅返回匹配到的key，key使用shell通配符的写法

```yaml
- name: Gathers facts about remote hosts
  setup:
      fact_path:             # Path used for local ansible facts (`*.fact') - files in this dir will be run (if
                               executable) and their results be added to `ansible_local'
                               facts if a file is not executable it is read. Check notes
                               for Windows options. (from 2.1 on) File/results format
                               can be JSON or INI-format. The default `fact_path' can be
                               specified in `ansible.cfg' for when setup is
                               automatically called as part of `gather_facts'.
      filter:                # If supplied, only return facts that match this shell-style (fnmatch) wildcard.
      gather_subset:         # If supplied, restrict the additional facts collected to the given subset. Possible
                               values: `all', `min', `hardware', `network', `virtual',
                               `ohai', and `facter'. Can specify a list of values to
                               specify a larger subset. Values can also be used with an
                               initial `!' to specify that that specific subset should
                               not be collected.  For instance:
                               `!hardware,!network,!virtual,!ohai,!facter'. If `!all' is
                               specified then only the min subset is collected. To avoid
                               collecting even the min subset, specify `!all,!min'. To
                               collect only specific facts, use `!all,!min', and specify
                               the particular fact subsets. Use the filter parameter if
                               you do not want to display some collected facts.
      gather_timeout:        # Set the default timeout in seconds for individual fact gathering.
```