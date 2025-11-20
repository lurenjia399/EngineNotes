# 1 FActorSpawnParameters 创建参数
```cpp
/*
struct FActorSpawnParameters  
{  
    UE_API FActorSpawnParameters();  
  
    /* A name to assign as the Name of the Actor being spawned. If no value is specified, the name of the spawned Actor will be automatically generated using the form [Class]_[Number]. */  
    FName Name;  
  
    /* An Actor to use as a template when spawning the new Actor. The spawned Actor will be initialized using the property values of the template Actor. If left NULL the class default object (CDO) will be used to initialize the spawned Actor. */  
    AActor* Template;  
  
    /* The Actor that spawned this Actor. (Can be left as NULL). */  
    AActor* Owner;  
  
    /* The APawn that is responsible for damage done by the spawned Actor. (Can be left as NULL). */  
    APawn* Instigator;  
  
    /* The ULevel to spawn the Actor in, i.e. the Outer of the Actor. If left as NULL the Outer of the Owner is used. If the Owner is NULL the persistent level is used. */  
    class  ULevel* OverrideLevel;
    }
*/
```