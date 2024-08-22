# 1 测试用例

MyTest.h 文件
```cpp

#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "MyTest.generated.h"

/**
 * 
 */
UCLASS()
class AZURE_API UMyTest : public UObject
{
	GENERATED_BODY()
	public:
	UMyTest();

	UFUNCTION()
	int AddFunc(int a, int b);
};

```

MyTest.cpp文件
```cpp
#include "MyTest.h"

UMyTest::UMyTest()
{
}

int UMyTest::AddFunc(int a, int b)
{
	return a + b;
}
```
# 2 展开宏

MyTest.h