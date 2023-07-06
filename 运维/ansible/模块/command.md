# command
在远程主机上执行命令，默认模块。command模块不支持`$VARNAME < > | ; &`等shell操作符，使用shell模块来实现操作符

```yaml
- name: Execute commands on targets
  command:
      argv:                  # Passes the command as a list rather than a string. Use `argv' to avoid quoting values
                               that would otherwise be interpreted incorrectly (for
                               example "user name"). Only the string or the list form
                               can be provided, not both.  One or the other must be
                               provided.
      chdir:                 # Change into this directory before running the command.
      cmd:                   # The command to run.
      creates:               # A filename or (since 2.0) glob pattern. If it already exists, this step *won't* be run.
      free_form:             # The command module takes a free form command to run. There is no actual parameter named
                               'free form'.
      removes:               # A filename or (since 2.0) glob pattern. If it already exists, this step *will* be run.
      stdin:                 # Set the stdin of the command directly to the specified value.
      stdin_add_newline:     # If set to `yes', append a newline to stdin data.
      strip_empty_ends:      # Strip empty lines from the end of stdout/stderr in result.
      warn:                  # Enable or disable task warnings.
```

> free_form参数表示没有指定参数名的参数值均会被当做free_form参数的值进行处理