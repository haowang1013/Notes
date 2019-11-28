# Environment Variables

- UE4_GITDEPS: local path for shared UE4 git dependencies storage

# Building

- Disable optimization for the entire module:
```
OptimizeCode = CodeOptimization.Never;
```

- Disable unity build for the entire module:
```
bFasterWithoutUnity = true;
```

- Enable logging in the shipping build:
```
bUseLoggingInShipping = true;
```

# Running

- To execute console command during startup, use:
```
<YOUR GAME>.exe -ExecCmds="<COMMAND 1>,<COMMAND 2>,<COMMAND 3>"
```

You can also use this method to override CVars via the command line:
```
<YOUR GAME>.exe -ExecCmds="<CVAR 1> <VALUE 1>,<CVAR 2> <VALUE 2>"
```

# Packaging

- Additional cmd line param can be added to the packaged build via the RunUAT script:
```
-addcmdline="<PARAM>"
```

- Add "-Messaging" cmd line param to the packaged build to make it communicate with the Session Frontend feature in the editor

# Android

- To send a console command via adb:
```
adb shell "am broadcast -a android.intent.action.RUN -e cmd '<COMMAND>'"

or 

doskey com=adb shell "am broadcast -a android.intent.action.RUN -e cmd '$*'"
com <COMMAND>
```

- To pull logs from the device
```
adb pull /sdcard/UE4Game/<ProjectName>/<ProjectName>/Saved/Logs
```

- Screenshot
```
adb shell screencap -p /sdcard/screencap.png
adb pull /sdcard/screencap.png
```

- Launch an application
```
adb shell am start -n com.yourcompany.yourproject/com.epicgames.ue4.GameActivity
```

- Realtime logging
```
adb logcat
```

# Cooking

- Add the following to the cmd line to manually start a cooking process
```
-run=Cook -TargetPlatform=<IOS|TVOS|Android_ASTC|...>
```

- To enable cooking size reporting:
```
dg.EnableObjectCookingStats=1
dg.MinObjectSizeForCookingStats=X
```

# Debugging

- To launch a macOS cooked build from Xcode, set the executable to
```
PackagedGame/<PLATFORM>/<GAME>.app/Contents/MacOS/<GAME>
```
And set the working directory to
```
PackagedGame/<PLATFORM>/<GAME>.app/Contents/MacOS
```
