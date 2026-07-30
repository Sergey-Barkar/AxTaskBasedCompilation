[![Apache License 2.0](https://img.shields.io/badge/license-Apache%202.0-blueviolet?style=for-the-badge)](https://www.apache.org/licenses/LICENSE-2.0)
[![NOTICE](https://img.shields.io/badge/notice-View-blue?style=for-the-badge)](./NOTICE)

# Task Based Application Compilation for Microsoft Dynamics AX 2009

This solution enables task-based parallel compilation of AOT objects in Microsoft Dynamics AX 2009, reducing compilation time by distributing work across multiple client instances.

## Installation
### 1. Import the project

Import `SharedProject_TaskBasedCompilation.xpo` into your AOT.

### 2. Configure SysStartupCmd

Add the following switch to the `construct` function of the `SysStartupCmd` class:
```
#DEFINE.COMMAND_TASKBASEDCOMPILE('taskbasedcompile')
;
...
    switch (s)
    {
    ...
        case #COMMAND_TASKBASEDCOMPILE :
            sysStartupCmd = new SysStartUpCmdTaskBasedCompile(s, parm);
            break;
    ...
    }
...
```

### 3. (Optional) Configure SysAutoRun

If you plan to use task-based compilation via `SysAutoRun`, modify the `SysAutoRun` class:

#### 3.1. Add the next function
```
private boolean tryToExecuteTaskBasedCompileApplication(XmlNode _command)
{
    int                         taskCompileWorkerCount  = str2int(this.getAttributeValue(_command, #XmlAttrTaskCompileWorkerCount)),
                                taskCompileNodesPerTask = str2int(this.getAttributeValue(_command, #XmlAttrTaskCompileNodesPerTask));
    TaskBasedCompileCoordinator coordinator;
    boolean                     ret;
    ;

    if (taskCompileWorkerCount && taskCompileNodesPerTask)
    {
        coordinator = new TaskBasedCompileCoordinator();

        if (coordinator.tryToCreateTasks(taskCompileNodesPerTask))
        {
            ret = coordinator.executeWorkers(taskCompileWorkerCount).awaitResult().publish();
        }

        coordinator.finalize();
    }

    return ret;
}
```

#### 3.2. Modify `execCompileApplication` function

Replace the code block: 
```
nodePath = this.getAttributeValue(_command, #XmlAttrNode);
if (nodePath == '')
{
    this.logInfo("@SYS96962");
    SysCompileAll::compile();
    result = true;
}
```

With:
```
nodePath = this.getAttributeValue(_command, #XmlAttrNode);
if (nodePath == '')
{
    this.logInfo("@SYS96962");
    if (!this.tryToExecuteTaskBasedCompileApplication(_command))
    {
        SysCompileAll::compile();
    }
    result = true;
}
```

## Usage 

### Via -startupcmd

Run compilation using the following command-line parameter:
```
-startupcmd=taskbasedcompile_{workerCount}:{nodesPerTask}
```
**Parameters:**
- `workerCount` — number of Microsoft Dynamics AX client instances to run in parallel
- `nodesPerTask` — number of AOT objects to compile per task (approximately: total AOT objects / nodesPerTask = number of tasks)

**Example:**
```
-startupcmd=taskbasedcompile_7:300
```

### Via SysAutoRun (XML)

Use the standard `CompileApplication` command with additional attributes:
- `taskCompileWorkerCount`
- `taskCompileNodesPerTask`

**Example:**
```xml
<?xml version="1.0" encoding="utf-8" ?>
<AxaptaAutoRun ExitWhenDone="true" version="5.0" logFile="...">
    <CompileApplication crossReference="false" taskCompileWorkerCount="7" taskCompileNodesPerTask="300" />
</AxaptaAutoRun>
```

## ⚠️ Warning

> The number of root AOT branches is hardcoded in the `TaskBasedCompileProcessCreator` class for performance optimization. Extend this class if your environment requires compiling additional AOT branch types.

> The `TaskBasedCompileResult` table stores only errors by default. Modify `createFromCompilerOutput` to capture additional compiler output (warnings/info messages).

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request