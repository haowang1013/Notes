- Only enabled when **UE_TRACE_ENABLED** is defined

# Command Line Options

See FTraceAuxiliary::Initialize

- -tracememmb={NUMBER}

  Set max memory used by the trace system

- -trace={channel1,channel2,presets,...}
  
  Set which trace channels are enabled

- -tracehost={HOST IP}
  
  Set the host IP where the Unreal Insights app is running
  
- -tracefile={FILE NAME}

  Set a local file name to which the trace data is stored
  
- -statnamedevents

  Enable stats based events
  
# INI Options

In BaseEngine.ini/DefaultEngine.ini

```
[Trace.ChannelPresets]
Default=cpu,frame,log,bookmark
Rendering=gpu,cpu,frame,log,bookmark
```

The Default preset will be used if nothing is specified in the command line.

Alternatively presets can be enabled via "-trace={PRESET NAME}"


# Console Commands

- Trace.Start {channel,preset...}

- Trace.Stop
