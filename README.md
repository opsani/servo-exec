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

