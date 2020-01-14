# Here's a collection of handy lldb commands in the XCode debugger

Also, see the cheat sheet here: https://www.nesono.com/sites/default/files/lldb%20cheat%20sheet.pdf

## Manually symbolicate an address:
```
image lookup --address {address}
```

## Print callstack of the current thread:
```
thread backtrace
```

## List all the threads:
```
thread list
```

## Print the value of a variable:
```
p <variable>
```

## Print array elements:
```
parray <count> <array variable>

UE4 example: 
parray 100 (float*)FloatValues.AllocatorInstance.Data
```

## Watchpoint:
```
To set a watch point on memory address:
w s e -- <address>

To list all the watchpoint:
watchpoint list

To delete a watchpoint:
watchpoint delete <ID>
```
