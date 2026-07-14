1 
```cpp
1 障碍物的应该带有FMassTrafficObstacleTag的Tag，以障碍物为中心，构建一个查询Box，通过ZoneGraph的OverlapLane接口查询覆盖的lane都有哪些，通过遍历BVTree找到相交的叶子节点
2 遍历找到的Lane，
```
2 