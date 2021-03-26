# Stats System

- Declare Stat Group
```
// Declare the stats group
DECLARE_STATS_GROUP(TEXT("MyGroup"), STATGROUP_MyGrop, STATCAT_Advanced);
```

- Declare Cycle Stat
```
// Declare a stat in the public header
DECLARE_CYCLE_STAT_EXTERN(TEXT("My Stat"), STAT_MyStat, STATGROUP_MyGrop, SOME_API);
// Define the stat in the cpp
DEFINE_STAT(STAT_MyStat);

// Or you can declare and define a stat in the cpp directly
DECLARE_CYCLE_STAT(TEXT("My Stat"), STAT_MyStat, STATGROUP_MyGrop);
```

- Use the Cycle Stat
```
{
  SCOPE_CYCLE_COUNTER(STAT_MyStat);
  // Some code uses cpu cycles...
}
```

- Unreal Insights Event Scope
```
{
  TRACE_CPUPROFILER_EVENT_SCOPE(ScopeName);
  // Some code uses cpu cycles...
}
```

# Unreal Insights
- See [UE4-Insights.md](UE4-Insights.md)


# Memory Profiling
- See [Low LevelMemory Tracker](https://docs.unrealengine.com/en-US/ProductionPipelines/DevelopmentSetup/Tools/LowLevelMemoryTracker/index.html)

- Pass "-llm" in the commmand line to enable

- Use "stat llm" to see the stats

- Custom LLM tags:

```
enum class ECustomLLMTag : LLM_TAG_TYPE
{
	CustomTag1 = (uint32)ELLMTag::ProjectTagStart + 10,
};

// In the header
DECLARE_LLM_MEMORY_STAT_EXTERN(TEXT("Custom Tag 1"), STAT_CustomTag1LLM, STATGROUP_MyGrop, SOME_API);

// In the cpp
DEFINE_STAT(STAT_CustomTag1LLM);

// In your module start up code
LLM(FLowLevelMemTracker::Get().RegisterProjectTag((int32)ECustomLLMTag::CustomTag1, TEXT("Custom Tag1"), GET_STATFNAME(STAT_CustomTag1LLM), NAME_None));

// Use the custom tag
{
  LLM_SCOPE((ELLMTag)ECustomLLMTag::CustomTag1);
  // Some code allocates memory
}
```
