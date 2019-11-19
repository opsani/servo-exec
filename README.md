# servo-exec
Arbitraty execution and HTTP request Optune servo driver

The purpose of this driver is to prepare the target environment for measurement, and clean up environment after the measurement. Possible preparation steps might include:
* Start VPN connection to tested environment
* Start and/or stop external load generator
* Start CI/CD job
* etc.


## Required control parameters:
* `duration` - Duration of the measurement in seconds

Note: this driver requires `measure.py` base class from the Optune servo core. It can be copied or symlinked here as part of packaging


## Configuration file specification
For security, all tasks executed by this driver must be specified in the config.yaml servo configuration file. None of the tasks. neither their parameters, could be overridden from backend.

Tasks are divided into four separate stages ('pre', 'start', 'stop', 'post'). Every stage is optional. Each stage is a list of  task definitions. Stages are executed in the following order: `pre` -> `start` --> `(wait for measurements duration)` --> `stop` -> `post`. The purpose of each stage is:
* `pre`: Verify that measurement environment is sane and clear. Errors in tasks that happened here, unless suppressed, would result in failed measurement. Tasks in `post` stage would be called if error happens.
* `start`: Start measurement, e.g. start external load generator using REST API. Errors in tasks that happened here, unless suppressed, would result in failed measurement. Tasks in `post` stage would be called if error happens.
* `stop`: Stop measurement, e.g. stop external load generator using REST API. Errors in tasks that happened here, unless suppressed, would result in failed measurement. Tasks in `post` stage would be called if error happens.
* `post`: Clean up environment, prepare for next measurement. Errors that happened in this stage are ignored.

Configuration file format:
```
exec:
  pre:
    - Task definition
    - Another task definition
  start:
    - Task definition
  stop:
    - Task definition
  post:
    - Task definition

```

## Task definition
```
- name: task name               # Task name, arbirtrary string that is used to identify task in logs and mesages.
                                # type: string, optional, highly recommended, default: None
  
  async: false                  # Indicates whether the task should be executed in sync (wait until task completes 
                                # or timeout is reached) or async (start task in background and let it run) mode.
                                # Exit code for async tasks would generally be ignored.
                                # type boolean (true|false), default is 'false'.
                                
  ignore_exitcode: false        # Indicates whether the exit code (HTTP status code) for the task should be ignored. 
                                # Exit codes for tasks in 'post' stage are always ignored. Unless this option is set,
                                # task returning non-zero exit code, or request returning a non-success code would
                                # interrupt execution. 
                                # type: boolean, optional, default: false
                                
  notify: false                 # Indicates whether stdout/stderr output from the task should be collected and sent 
                                # to backend.
                                # type: boolean, optional, default: false
                                
  retries: 0                    # How many times task should be retried in case of an error. 
                                # type: int, optional, default: 0 (no retries)
                                
  timeout: 10                   # Timeout for task. Unless task completes within this time, it would be terminated and 
                                # an error returned.
                                # type: int, in seconds, optional, highly recommended, default is none.


                                # This driver supports 3 different execution modes:
                                #   - exec:         executes a binary or shell script
                                #   - shell_exec:   executes a binary or shell script within a shell with support for
                                #                   variables expansion, pipes, redirections, etc.
                                #   - request:      performs http request
                                #
                                # Only one of the execution modes can be specified in task.

  exec:                         # Execute binary or shell script with argument list
    - argX
    - argY
    - argZ

  exec: /path/to/binary         # Execute binary or shell script without arguments

    
  shell_exec: "command string"  # Execute command(s) specified in string

  shell_exec:                   # Execute command with shell, the rest or arguments are passed to shell (not to command!)
    - command
    - shell_argumentA
    - shell_argumentB
    - shell_argumentC

  request:                      # Perform HTTP request
    method: GET                 # HTTP method to use. This is passed directly to requests.request, any method supported there
                                # can be specified.
                                # type: string, optional, default: 'GET'
                                
    url:                        # URL for the request. Must include schema (http|https).
                                # type: string, required.
                                
    http_timeout:               # Timeout in seconds for HTTP connect and HTTP idle. 
                                # type: int, optional, default: 10
                                
    data:                       # Data for HTTP POST request. It is recommended to specify correct content-type in headers as well.
                                # type: string, optional, default is '{}'
                                
    content-type:               # Content-type header for POST request, unused otherwise.
                                # type: string, optional, default 'application/json'
                                
    headers:                    # Additional headers to send with request.
                                # type: dict, default is empty ({})
                                
    success_codes:              # A string or list of values or ranges that identify successful HTTP request. Valid values are:
                                #   success_codes: 200
                                #   success_codes: "200-399"
                                #   success_codes: 
                                #     - 200
                                #     - 201
                                #     - "200-299"
                                # type: list or string, optional, default is "200-399"
```

## Example task definitions
```
  # HTTP HEAD request to 'http://api:8080?q=start', with 15-second timeout, special user-agent header and non-standard HTTP status codes.
  - name: HEAD request
    ignore_exitcode: true
    notify: true
    request:
      method: HEAD
      url: http://api:8080?q=start
      http_timeout: 15
      headers:
        user-agent: Mozilla/5.0
      success_codes:
        - 200-399
        - 876

  # Start OpenVPN in background
  - name: Start VPN tunnel
    async: true
    retries: 3
    exec:               
      - /bin/openvpn
      - -c
      - /etc/openvpn.conf
    
  # Grep for DATE in log
  - name: Find data
    shell_exec: "cat /var/log/log | grep DATE | mail user@host.com"
    ignore_exitcode: false
    notify: true
