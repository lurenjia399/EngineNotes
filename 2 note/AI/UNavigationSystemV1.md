# AddNavigationBoundsUpdateRequest
``` cpp
void UNavigationSystemV1::PerformNavigationBoundsUpdate(const TArray<FNavigationBoundsUpdateRequest>& UpdateRequests)
{
	TArray<FBox> UpdatedAreas;
	for (const FNavigationBoundsUpdateRequest& Request : UpdateRequests)
	{
		FSetElementId ExistingElementId = RegisteredNavBounds.FindId(Request.NavBounds);

		switch (Request.UpdateRequest)
		{
		case FNavigationBoundsUpdateRequest::Removed:
			{
				if (ExistingElementId.IsValidId())
				{
					UpdatedAreas.Add(RegisteredNavBounds[ExistingElementId].AreaBox);
					RegisteredNavBounds.Remove(ExistingElementId);
				}
			}
			break;

		case FNavigationBoundsUpdateRequest::Added:
		case FNavigationBoundsUpdateRequest::Updated:
			{
				if (ExistingElementId.IsValidId())
				{
					const FBox ExistingBox = RegisteredNavBounds[ExistingElementId].AreaBox;
					const bool bSameArea = (Request.NavBounds.AreaBox == ExistingBox);
					if (!bSameArea)
					{
						UpdatedAreas.Add(ExistingBox);
					}

					// always assign new bounds data, it may have different properties (like supported agents)
					RegisteredNavBounds[ExistingElementId] = Request.NavBounds;
				}
				else
				{
					AddNavigationBounds(Request.NavBounds);
				}

				UpdatedAreas.Add(Request.NavBounds.AreaBox);
			}

			break;
		}
	}
}
```

# 
