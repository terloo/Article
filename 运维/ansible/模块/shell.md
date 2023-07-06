# shell
在远程主机上执行shell命令，类似于command模块，但功能更强

```yaml
- name: Execute shell commands on targets
  shell:
      chdir:                 # Change into this directory before running the command.
      cmd:                   # The command to run followed by optional arguments.
      creates:               # A filename, when it already exists, this step will *not* be run.
      executable:            # Change the shell used to execute the command. This expects an absolute path to the
                               executable.
      free_form:             # The shell module takes a free form command to run, as a string. There is no actual
                               parameter named 'free form'. See the examples on how to
                               use this module.
      removes:               # A filename, when it does not exist, this step will *not* be run.
      stdin:                 # Set the stdin of the command directly to the specified value.
      stdin_add_newline:     # Whether to append a newline to stdin data.
      warn:                  # Whether to enable task warnings.
```