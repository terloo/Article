# fetch

将远程主机上的文件拷贝到ansible主机

1. dest：存放文件的ansible主机目录，会对每个远程主机建立一个名为`$HOSTNAME`的目录

```yaml
- name: Fetch files from remote nodes
  fetch:
      dest:                  # (required) A directory to save the file into. For example, if the `dest' directory is
                               `/backup' a `src' file named `/etc/profile' on host
                               `host.example.com', would be saved into
                               `/backup/host.example.com/etc/profile'. The host name is
                               based on the inventory name.
      fail_on_missing:       # When set to `yes', the task will fail if the remote file cannot be read for any reason.
                               Prior to Ansible 2.5, setting this would only fail if the
                               source file was missing. The default was changed to `yes'
                               in Ansible 2.5.
      flat:                  # Allows you to override the default behavior of appending hostname/path/to/file to the
                               destination. If `dest' ends with '/', it will use the
                               basename of the source file, similar to the copy module.
                               This can be useful if working with a single host, or if
                               retrieving files that are uniquely named per host. If
                               using multiple hosts with the same filename, the file
                               will be overwritten for each host.
      src:                   # (required) The file on the remote system to fetch. This `must' be a file, not a
                               directory. Recursive fetching may be supported in a later
                               release.
      validate_checksum:     # Verify that the source and destination checksums match after the files are fetched.
```