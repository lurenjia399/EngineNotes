1 这种Translator会在Trait的BuildTemplate中将自己需要的RequireTag添加到TemplateData中
2 就是获取mass的位置，然后将其设到Actor的rootComp上
```cpp
void UHTMassTransformToActorTranslator::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	for (FMassExecutionContext::FEntityIterator EntityIt = 
		Context.CreateEntityIterator(); EntityIt; ++EntityIt)
	{
		AsComponent->SetWorldTransform(UpdateTransform);
	}
}
```