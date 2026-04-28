# 1 成员变量
```cpp

	UPROPERTY(Category = Zone, EditAnywhere, BlueprintReadWrite, meta=(AllowPrivateAccess="true", IncludeInHash))
	FZoneLaneProfileRef LaneProfile;

	/** True if lane profile should be reversed */
	UPROPERTY(Category = Zone, EditAnywhere, BlueprintReadWrite, meta = (AllowPrivateAccess = "true", IncludeInHash))
	bool bReverseLaneProfile = false;

	/** Array of lane templates indexed by the points when the shape is polygon. */
	UPROPERTY(Category = Zone, VisibleAnywhere, meta = (IncludeInHash, EditCondition = "false", EditConditionHides))
	TArray<FZoneLaneProfileRef> PerPointLaneProfiles;

	// 这个Component里面的点的数组
	/** Shape points */
	UPROPERTY(Category = Zone, EditAnywhere, BlueprintReadWrite, meta = (AllowPrivateAccess = "true", IncludeInHash))
	TArray<FZoneShapePoint> Points;

	/** Shape type, spline or polygon */
	UPROPERTY(Category = Zone, EditAnywhere, BlueprintReadWrite, meta = (AllowPrivateAccess = "true", IncludeInHash))
	FZoneShapeType ShapeType;

	/** Polygon shape routing type */
	UPROPERTY(Category = Zone, EditAnywhere, BlueprintReadWrite, meta = (AllowPrivateAccess = "true", IncludeInHash, EditCondition = "ShapeType == FZoneShapeType::Polygon"))
	EZoneShapePolygonRoutingType PolygonRoutingType = EZoneShapePolygonRoutingType::Bezier;
	
	/** Zone tags, the lanes inherit zone tags. */
	UPROPERTY(Category = Zone, EditAnywhere, BlueprintReadWrite, meta = (AllowPrivateAccess = "true", IncludeInHash))
	FZoneGraphTagMask Tags;

//#if HOTTA_ENGINE_MODIFY
	/*It's Modify Lane Code.  Modify By ShiZhiYuan*/
	UPROPERTY(Category = Zone, EditAnywhere, BlueprintReadWrite, meta = (AllowPrivateAccess = "true", IncludeInHash))
	TMap<int32, FExtraLaneInfo> ExtraLaneInfos;
//#endif

//#if HOTTA_ENGINE_MODIFY
	/*It's moving along the normal.  Modify By ShiZhiYuan*/
	UPROPERTY(Category = Zone, EditAnywhere, BlueprintReadWrite, meta = (AllowPrivateAccess = "true", IncludeInHash))
	float OffsetAlongNormal = 0.0f;
//#endif

	// 连接，一个compone
	/** Connectors for other shapes (not stored, these are refreshed from points). */
	UPROPERTY(Transient)
	TArray<FZoneShapeConnector> ShapeConnectors;

	/** Array of connections matching ShapeConnectors (not stored, these are refreshed from connectors). */
	UPROPERTY(Transient)
	TArray<FZoneShapeConnection> ConnectedShapes;
```