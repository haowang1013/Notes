# Oculus Quest

- To set the [Fixed Foveated Rendering](https://developer.oculus.com/documentation/quest/latest/concepts/mobile-ffr/) level on the system level:
```
adb shell setprop debug.oculus.foveation.level <0..4>
```

- Make sure [OVR Metrics Tool](https://developer.oculus.com/documentation/mobilesdk/latest/concepts/mobile-ovrmetricstool) is used for monitoring in-game performance, tt's much cheaper than stats rendering in UE4.

