---
layout: wmt/docs
title:  Concord Task
side-navigation: wmt/docs-navigation.html
---

# {{ page.title }}

The `concord` task allows users to start and manage new processes from within
running processes.

The task is provided automatically for all flows, no external dependencies
necessary.

- [Examples](#examples)
- [Parameters](#parameters)
- [Starting a Process using a Payload Archive](#start-payload)
- [Starting a Process using an Existing Project](#start-project)
- [Starting an External Process](#start-external)
- [Scheduling a Process](#start-schedule)
- [Specifying Profiles](#start-profiles)
- [File Attachments](#start-attachments)
- [Forking a Process](#fork)
- [Forking Multiple Instances](#fork-multi)
- [Synchronous Execution](#sync)
- [Suspending Parent Process](#start-suspend)
- [Suspending for Completion](#suspend-for-completion)
- [Waiting for Completion](#wait-for-completion)
- [Handling Cancellation and Failures](#handle-onfailure)
- [Cancelling Processes](#cancel)
- [Tagging Subprocesses](#tags)
- [Output Variables](#outvars)

## Examples

- [process_from_a_process]({{ site.concord_source }}/tree/master/examples/process_from_a_process) 
- starting a new subprocess from a flow using a payload archive;
- [fork]({{ site.concord_source }}/tree/master/examples/fork) - starting a subprocess;
- [fork_join]({{ site.concord_source }}/tree/master/examples/fork_join)
- starting multiple subprocesses and waiting for completion.

## Parameters

All parameter sorted alphabetically. Usage documentation can be found in the
following sections:

- `action` - string, name of the action (`start`, `startExternal`, `fork`, `kill`);
- `activeProfiles` - list of string values, profiles to activate;
- `apiKey` - string, Concord API key to use. If not specified the task uses
the current process' session key;
- `arguments` - input arguments of the starting processes;
- `disableOnCancel` - boolean, disable `onCancel` flow in forked processes;
- `disableOnFailure` - boolean, disable `onFailure` flow in forked processes;
- `entryPoint` - string, name of the starting process' flow;
- `ignoreFailures` - boolean, ignore failed processes;
- `instanceId` - UUID, ID of the process to `kill`;
- `org` - string, name of the process' organization, optional, defaults to the 
organization of the calling process;
- `outVars` - list of string values, out variables to capture;
- `payload` - path to a ZIP archive or a directory, the process' payload;
- `requirements` - object, allows specifying the process'
[requirements](../processes-v1/configuration.html#requirements); 
- `project` - string, name of the process' project;
- `repo` - string, name of the project's repository to use;
- `repoBranchOrTag` - string, overrides the configured branch or tag name of
the project's repository;
- `repoCommitId` - string, overrides the configured GIT commit ID of the
project's repository;
- `startAt` - [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) date/time
value, the process' start time;
- `suspend` - boolean, if `true` and `sync` is enabled the process [suspends](#start-suspend)
waiting for the child process to complete (only for actions `start` and `fork`); 
- `sync` - boolean, wait for completion if `true`, defaults to `false`;
- `debug` - boolean, if `true` the plugin logs additional debug information, defaults to `false`;
- `tags` - list of string values, the process' tags;
- `attachments` - list of file attachments;

<a name="start-payload"/>

## Starting a Process using a Payload Archive

```yaml
flows:
  default:
  - task: concord
    in:
      action: start
      payload: payload.zip
```

The `start` action starts a new subprocess using the specified payload archive.
The ID of the started process is stored as the first element of `${jobs}` array:

```yaml
- log: "I've started a new process: ${jobs[0]}"
```

<a name="start-project"></a>

## Starting a Process using an Existing Project

```yaml
flows:
  default:
  - task: concord
    in:
      action: start
      org: myOrg
      project: myProject
      payload: payload.zip
```

The `start` expression with a `project` parameter and a `payload` in the form of
a zip archive or the name of a folder in your project repository, starts a new
subprocess in the context of the specified project.

Alternatively, if the project has a repository configured, the process can be
started by configuring the repository:

```yaml
flows:
  default:
  - task: concord
    in:
      action: start
      project: myProject
      repo: myRepo
```

The process is started using the resources provided by the specified archive, 
project and repository.

<a name="start-external"/>

## Starting an External Process

To start a process on an external Concord instance use the `startExternal` action:

```yaml
flows:
  default:
  - task: concord
    in:
      baseUrl: "http://another.concord.example.com:8001"
      apiKey: "myApiKey"
      action: startExternal
      project: myProject
      repo: myRepo
```

Connection parameters can be overridden using the following keys:

- `baseUrl` - Concord REST API endpoint. Defaults to the current
  server's API endpoint address;
- `apiKey` - user's REST API key.

**Note:** The `suspend: true` option is not supported with the `startExternal`
action.

<a name="start-schedule"/>

## Scheduling a Process

To schedule a process to a specific date and time, use the `startAt` parameter:

```yaml
flows:
  default:
  - task: concord
    in:
      action: start
      ...
      startAt: "2018-03-16T23:59:59-05:00"
```

The `startAt` parameter accepts an ISO-8601 string, `java.util.Date` or
`java.util.Calendar` values. It is important to include a timezone as the
server may use a different default timezone value.

<a name="start-profiles"/>

## Specifying Profiles

To specify which profiles are used to start the process, use the
`activeProfiles` parameter:

```yaml
flows:
  default:
  - task: concord
    in:
      action: start
      ...
      activeProfiles:
      - firstProfile
      - secondProfile
```

The parameter accepts either a YAML array or a comma-separated string value.

<a name="start-attachments"/>

## File Attachments

To start a process with file attachments, use the `attachments` parameter. An
attachment can be a single path to a file, or a map which specifies a source
and destination filename for the file. If the attachment is a single path, the
file is placed in the root directory of the new process with the same name.

```yaml
flows:
  default:
  - task: concord
    in:
      ...
      attachments:
      - ${workDir}/someDir/myFile.txt
      - src: anotherFile.json
        dest: someFile.json
```

This is equivalent to the curl command:
```
curl ... -F myFile.txt=@${workDir}/someDir/myFile.txt -F someFile.json=@anotherFile.json ...
```

<a name="fork"/>

## Forking a Process

Forking a process creates a copy of the current process. All variables and
files defined at the start of the parent process are copied to the child process
as well:

```yaml
flows:
  default:
  - task: concord
    in:
      action: fork
      entryPoint: sayHello
        
  sayHello:
  - log: "Hello from a subprocess!"
```

The IDs of the started processes are stored as `${jobs}` array.

**Note:** Due to the current limitations, files created after
the start of a process cannot be copied to child processes.

<a name="fork-multi"/>

## Forking Multiple Instances

It is possible to create multiple forks of a process with a different
sets of parameters:

```yaml
flows:
  default:
  - task: concord
    in:
      action: fork
      forks:
      - entryPoint: pickAColor
        arguments:
          color: "red"
      - entryPoint: pickAColor
        arguments:
          color: "green"
      - entryPoint: pickAColor
        instances: 2
        arguments:
          color: "blue"
```

The `instances` parameter allows spawning of more than one copy of a process.

The IDs of the started processes arestored as `${jobs}` array.

<a name="sync"/>

## Synchronous Execution

By default all subprocesses are started asynchronously. To start a process and
wait for it to complete, use `sync` parameter:

```yaml
flows:
  default:
  - tasks: concord
    in:
      action: start
      payload: payload.zip
      sync: true
```

If a subprocess fails, the task throws an exception. To ignore failed processes
use `ignoreFailures: true` parameter:

```yaml
flows:
  default:
  - tasks: concord
    in:
      action: start
      payload: payload.zip
      sync: true
      ignoreFailures: true
```

<a name="start-suspend"/>

## Suspending Parent Process

There's an option to suspend the parent process while it waits for the child process
completion:

```yaml
flows:
  default:
  - task: concord
    in:
      action: start
      org: myOrg
      project: myProject
      repo: myRepo
      sync: true
      suspend: true
      
  - log: "Done: ${jobs}"
```

This can be very useful to reduce the amount of Concord agents needed. With
`suspend: true`, the parent process does not consume any resources including
agent workers, while waiting for the child process.

`suspend` can be used with the `fork` action as well:

```yaml
flows:
  default:
  - task: concord
    in:
      action: fork
      forks:
        - entryPoint: sayHello
        - entryPoint: sayHello
        - entryPoint: sayHello
      sync: true
      suspend: true

  sayHello:
    - log: "Hello from a subprocess!"
```

Currently, `suspend` can only be used with the `start` and `fork` actions.

**Note:** Due to the current limitations, files created after the start of
the parent process are not preserved. Effectively, the suspend works in the same
way as the [forms](../getting-started/forms.html).

<a name="suspend-for-completion"/>

## Suspending for Completion

You can use the follow approach to suspend a process until the the completion of
other process:

```yaml
flows:
  default:
  - set:
      children: []

  - task: concord
    in:
      action: start
      payload: payload

  - ${children.addAll(jobs)}

  - task: concord
    in:
      action: start
      payload: payload

  - ${children.addAll(jobs)}

  - ${concord.suspendForCompletion(children)}

  - log: "process is resumed."
```

<a name="wait-for-completion"/>

## Waiting for Completion

To wait for a completion of a process:

```yaml
flows:
  default:
  - ${concord.waitForCompletion(jobIds)}
```

The `jobIds` value is a list (as in `java.util.List`) of process IDs.

The expression returns a map of process entries. To see all returned fields,
check with the [Process API](../api/process.html#status):

```json
{
  "56e3dcd8-a775-11e7-b5d6-c7787447ca6d": {
    "status": "FINISHED"
  },
  "5cd83364-a775-11e7-aadd-53da44242629": {
    "status": "FAILED",
    "meta": {
      "out": {
        "lastError": {
          "message": "Something went wrong."
        }
      }
    }
  }
}
```

<a name="handle-onfailure"/>

## Handling Cancellation and Failures

Just like regular processes, subprocesses can have `onCancel` and `onFailure`
flows.

However, as process forks share their flows, it may be useful to disable
`onCancel` or `onFailure` flows in subprocesses:

```yaml
flows:
  default:
  - task: concord
    in:
      action: fork
      disableOnCancel: true
      disableOnFailure: true
      entryPoint: sayHello

  sayHello:
  - log: "Hello!"
  - throw: "Simulating a failure..."
  
  # invoked only for the parent process
  onCancel:
  - log: "Handling a cancellation..."
  
  # invoked only for the parent process
  onFailure:
  - log: "Handling a failure..."
```

<a name="cancel"/>

## Cancelling Processes

The `cancel` action can be used to cancel the execution of a subprocess.

```yaml
flows:
  default:
  - task: concord
    in:
      action: cancel
      instanceId: ${someId}
      sync: true
```

The `instanceId` parameter can be a single value or a list of process
IDs.

Setting `sync` to `true` forces the the task to wait until the specified
processes are stopped.

<a name="tags"/>

## Tagging Subprocesses

The `tags` parameters can be used to tag new subprocess with one or multiple
labels.

```yaml
flows:
  default:
  - task: concord
    in:
      action: start
      payload: payload.zip
      tags: ["someTag", "anotherOne"]
```

The parameter accepts either a YAML array or a comma-separated string value.

Tags are useful for filtering (sub)processes:

```yaml
flows:
  default:
  # spawn multiple tagged processes
  
  onCancel:
  - task: concord
    in:
      action: kill
      instanceId: "${concord.listSubprocesses(parentInstanceId, 'someTag')}"
```

<a name="outvars"/>

## Output Variables

Variables of a child process can be accessed via the `outVars` configuration. 
The functionality requires the `sync` parameter to be set to `true`.

```yaml
# runtime v1 and v2 example

flows:
  default:
    - task: concord
      in:
        action: start
        project: myProject
        repo: myRepo
        sync: true
        # list of variable names
        outVars:
        - someVar1
        - someVar2
```

In processes using [the v1 runtime](../processes-v1/index.html) the result is
automatically stored as a `jobOut` variable:

```yaml
- log: "We got ${jobOut.someVar1} and ${jobOut.someVar2}!"
```

When using [the v2 runtime](../processes-v2/index.html), the result must be
explicitly saved using the `out` syntax:

```yaml
# runtime v2 example

configuration:
  runtime: "concord-v2"

flows:
  default:
    - task: concord
      in:
        action: start
        project: myProject
        repo: myRepo
        sync: true
        # list of variable names
        outVars:
        - someVar1
        - someVar2
      out: jobOut

    - log: "We got ${jobOut.someVar1} and ${jobOut.someVar2}"
```

When starting multiple forks their output variables are collected into a nested
object with fork IDs as keys:

```yaml
# runtime v1 example

configuration:
  arguments:
    name: "Concord"

flows:
  default:
    - task: concord
      in:
        action: fork
        forks:
          - entryPoint: onFork
            arguments:
              msg: "Hello"
          - entryPoint: onFork
            arguments:
              msg: "Bye"
        sync: true
        suspend: true
        outVars:
          - varFromFork

    - log: "${jobOut}"

  onFork:
    - log: "Running in a fork"
    - set:
        varFromFork: "${msg}, ${name}"
```

In the example above, the `jobOut` variable has the following structure:

```json
{
    "54e21668-d011-11e9-8369-6b27f0faf40f": {
        "varFromFork": "Hello, Concord"
    },
    "6ac8b356-d011-11e9-81f6-73dd6c99b3b2": {
        "varFromFork": "Bye, Concord"
    }
}
```

Because each fork can produce a variable with the same name, the values are
nested into objects with the fork ID as the key.

To get output variables for already running processes use `getOutVars` method:

```yaml
# runtime v1 example

flows:
  default:
    # empty list to store fork IDs
    - set:
        children: []

    # start the first fork
    - task: concord
      in:
        action: fork
        entryPoint: forkA
        sync: false
        outVars:
          - x

    # save the first fork's ID
    - ${children.addAll(jobs)}

    # start the second fork
    - task: concord
      in:
        action: fork
        entryPoint: forkB
        sync: false
        outVars:
          - y

    # save the second fork's ID
    - ${children.addAll(jobs)}

    # grab out vars of the forks
    - expr: ${concord.getOutVars(children)} # accepts a list of process IDs
      out: forkOutVars

    # print out out vars grouped by fork
    - log: "${forkOutVars}"

  forkA:
    - set:
        x: 1

  forkB:
    - set:
        y: 2
```

Or, when using [the runtime v2](../processes-v2/index.html):

```yaml
# runtime v2 example

configuration:
  runtime: "concord-v2"

flows:
  default:
    # empty list to store fork IDs
    - set:
        children: []

    # start the first fork and save the result as "firstResult"
    - task: concord
      in:
        action: fork
        entryPoint: forkA
        sync: false
        outVars:
          - x
      out: firstResult

    # save the first fork's ID
    - ${children.addAll(firstResult.forks)}

    # start the second fork and save the result as "secondResult"
    - task: concord
      in:
        action: fork
        entryPoint: forkB
        sync: false
        outVars:
          - y
      out: secondResult

    # save the second fork's ID
    - ${children.addAll(secondResult.forks)}

    # grab out vars of the forks
    - expr: ${concord.getOutVars(children)} # accepts a list of process IDs
      out: forkOutVars

    # print out out vars grouped by fork
    - log: "${forkOutVars}"

  forkA:
    - set:
        x: 1

  forkB:
    - set:
        y: 2
```

**Note:** the `getOutVars` method waits for the specified processes to finish.
If one of the specified processes fails, its output is going to be empty.

In [the runtime v2](../processes-v2/index.html) all variables are
"local" -- limited to the scope they were defined in. The `outVars` mechanism
grabs only the top-level variables, i.e. variables available in the `entryPoint`
scope:

```yaml
# runtime v2 example

configuration:
  runtime: "concord-v2"

flows:
  # caller
  default:
    - task: concord
      in:
        action: fork
        sync: true
        entryPoint: onFork
        outVars:
          - foo
          - bar
      out: forkResult

    - log: "${concord.getOutVars(forkResult.forks)}"

  # callee
  onFork:
    - set:
        foo: "abc" # ok, top-level variable

    - call: anotherFlow # not ok, "bar" stays local to "anotherFlow"

    - call: anotherFlow # ok, "bar" pushed into the current scope, becomes a top-level variable
      out:
        - bar

  anotherFlow:
    - set:
        bar: "xyz"
```
