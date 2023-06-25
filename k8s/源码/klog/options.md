# options
klog的所有选项

## 默认行为
klog将日志信息全部输出到标准错误流

## 选项
| 选项名            | 类型           | 默认值   | 说明                                                                                                                                           |
| ----------------- | -------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| logtostderr       | bool           | true     | 是否把日志输出到标准错误流。如果为true，将忽略其他配置，将所有等级的日志全部输出到标准错误流                                                   |
| log_file          | string         |          | 如果不为空，把文件输出到单个文件，并忽略logdir相关配置                                                                                         |
| log_file_max_size | int            | 1800     | 日志文件最大大小，设置为0表示无限制。单位为MB                                                                                                  |
| log_dir           | string         |          | 如果不为空，把文件输出到目录，目录中的文件按日志等级和大小进行切割，并建立便于查看日志的软连接。高等级的日志文件也会包含低等级的日志文件信息。 |
| one_output        | bool           | false    | 不同等级的日志文件是否只包含当前等级的日志信息，如果为false，高等级的日志文件会包含低等级的日志文件信息                                        |
| alsologtostderr   | bool           | false    | 是否在将日志输出到文件/文件夹同时也输出到标准错误流，与stderrthreshold是或的关系                                                               |
| stderrthreshold   | int            | 2(Error) | 在日志等级是否大于stderrthreshold时，日志输出到文件/文件夹同时也输出到标准错误流                                                               |
| v                 | int            | 0        | 只输出**大于等于**v等级的vlog                                                                                                                  |
| vmodule           | <pattern>=<N>  |          |
| skip_log_headers  | bool           | false    | 是否输出日志文件的头信息                                                                                                                       |
| skip_headers      | bool           | false    | 是否输出日志信息的前缀信息                                                                                                                     |
| add_dir_header    | bool           | false    | 日志信息的前缀信息中是否显示文件的完整路径                                                                                                     |
| log_backtrace_at  | <filename>:<N> |          | 当filename的日志长度超过N时，不输出堆栈信息                                                                                                    |