# Usage
使用方法

## 选项绑定
```go
// 将klog的选项与某个flagset绑定
func InitFlags(flagset *flag.FlagSet) {
    // 如果没有传入，则使用命令行flagSet
	if flagset == nil {
		flagset = flag.CommandLine
	}

    // 对全局变量logging进行绑定
	flagset.StringVar(&logging.logDir, "log_dir", logging.logDir, "If non-empty, write log files in this directory")
	flagset.StringVar(&logging.logFile, "log_file", logging.logFile, "If non-empty, use this log file")
	flagset.Uint64Var(&logging.logFileMaxSizeMB, "log_file_max_size", logging.logFileMaxSizeMB,
		"Defines the maximum size a log file can grow to. Unit is megabytes. "+
			"If the value is 0, the maximum file size is unlimited.")
	flagset.BoolVar(&logging.toStderr, "logtostderr", logging.toStderr, "log to standard error instead of files")
	flagset.BoolVar(&logging.alsoToStderr, "alsologtostderr", logging.alsoToStderr, "log to standard error as well as files")
	flagset.Var(&logging.verbosity, "v", "number for the log level verbosity")
	flagset.BoolVar(&logging.addDirHeader, "add_dir_header", logging.addDirHeader, "If true, adds the file directory to the header of the log messages")
	flagset.BoolVar(&logging.skipHeaders, "skip_headers", logging.skipHeaders, "If true, avoid header prefixes in the log messages")
	flagset.BoolVar(&logging.oneOutput, "one_output", logging.oneOutput, "If true, only write logs to their native severity level (vs also writing to each lower severity level)")
	flagset.BoolVar(&logging.skipLogHeaders, "skip_log_headers", logging.skipLogHeaders, "If true, avoid headers when opening log files")
	flagset.Var(&logging.stderrThreshold, "stderrthreshold", "logs at or above this threshold go to stderr")
	flagset.Var(&logging.vmodule, "vmodule", "comma-separated list of pattern=N settings for file-filtered logging")
	flagset.Var(&logging.traceLocation, "log_backtrace_at", "when logging hits line file:N, emit a stack trace")
}
```