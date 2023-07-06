# script
传输ansible主机上的脚本到远程主机并执行

```yaml
- name: Runs a local script on a remote node after transferring it
  script:
      chdir:                 # Change into this directory on the remote node before running the script.
      cmd:                   # Path to the local script to run followed by optional arguments.
      creates:               # A filename on the remote node, when it already exists, this step will *not* be run.
      decrypt:               # This option controls the autodecryption of source files using vault.
      executable:            # Name or path of a executable to invoke the script with.
      free_form:             # Path to the local script file followed by optional arguments.
      removes:               # A filename on the remote node, when it does not exist, this step will *not* be run.
```