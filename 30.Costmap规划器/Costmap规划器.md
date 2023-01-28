# Costmap规划器

Isaac SDK 中的标准导航规划器指示机器人在避开障碍物的同时采用最短路线到达目标。 但是，对于许多环境，您可能需要指定限制区域或为机器人提供交通规则。

Costmap Planner 应用程序允许您创建更细粒度的导航行为。 它提供四种主要类型的 FlatmapCost：

* OccupancyFlatmapCost，可用于提供地图

* PolygonFlatmapCost，可用于实现禁区或惩罚区

* PolylineFlatmapCost，可用于定义高速公路

* DirectedAreaFlatmapCost，可用于创建有向矢量场

这些层可以使用 MultiplicationFlatmapCost 或 AdditionFlatmapCost 相乘或相加，您可以通过实现 FlatmapCost 接口添加您自己的自定义 FlatmapCost。

## 组件
此应用程序使用以下组件：

* [isaac.path_planner.Pose2GridGraphBuilder](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-path-planner-pose2gridgraphbuilder)

* [isaac.path_planner.Pose2GraphPlanner](https://docs.nvidia.com/isaac/doc/doc/component_api.html#isaac-path-planner-pose2graphplanner)

## 消息
此应用程序使用 [Pose2GraphProto](https://docs.nvidia.com/isaac/doc/doc/message_api.html#pose2graphproto) 消息。

## 入门
运行以下命令在 `/tmp/`中为未来工厂 (`demo_5`) 的地图创建一个图：

```bash
bazel run packages/path_planner/apps:pose2_graph_builder
```

然后，使用新的规划器运行 flatsim：
```bash
bazel run //packages/flatsim/apps:flatsim -- --demo demo_5
```


## 自定义图
要自定义图形，请编辑 packages/path_planner/apps/pose2_graph_builder.app.json 文件。

要选择地图，请编辑“map”配置：

```json
"map": {
      "occupancy": {
        "cell_size": 0.05,
        "filename": "apps/assets/maps/virtual_factory_1.png"
      }
    },
```

要将平面图添加到列表中，请编辑“flatmap_cost”配置：

```json
{
  "name": "flatmap_cost",
  "start_order": -100,
  "components": [
    {
      "name": "OccupancyFlatmapCost",
      "type": "isaac::map::OccupancyFlatmapCost"
    },
    {
      "name": "inside_round",
      "type": "isaac::map::PolylineFlatmapCost"
    },
    {
      "name": "outside_round",
      "type": "isaac::map::PolylineFlatmapCost"
    },
    {
      "name": "restricted_area",
      "type": "isaac::map::PolygonFlatmapCost"
    },
    {
      "name": "MultiplicationFlatmapCost",
      "type": "isaac::map::MultiplicationFlatmapCost"
    }
  ]
}
```
* MultiplicationFlatmapCost 组件目前是必需的。 但是，您可以删除或添加更多自定义层。

* OccupancyFlatmapCost 组件确保机器人不会在地图中发生碰撞。

* “inside_round”是指示机器人在内环行驶的成本图。

* “outside_round”是指示机器人在外环行驶的成本图。

* “restricted_area”禁止在中间区域行驶。

## 使用自定义地图
### 改变规划器
要将 GlobalPlanner 切换为新的 planner，需要在创建图形后切换行为，如 `//packages/flasim/apps/demo_5.json` 所示：
```json
"flatsim.navigation.planner.planner_switch_behavior": {
  "SwitchBehavior": {
    "desired_behavior": "$(fullname flatsim.navigation.planner.pose2_graph_planner)"
  }
},
```
您还需要编辑 SwitchBehavior 节点以确保节点名称与新应用匹配。

您可能还需要更改新规划器的配置以确保它加载正确的图表：


```json
"navigation.planner.pose2_graph_planner": {
   "Pose2DirectedGraphLoader": {
     "bucket_size": 0.5,
     "graph_filename": "/tmp/pose2_grid_graph.capnp.bin"
   }
 },
```
### 将 Costmap 添加到视线中
此步骤是可选的，但它可能有助于在 Sight 中可视化用于成本地图的不同区域。 只需添加构建图形的应用程序中使用的 Costmap。 请注意，您只需要添加渲染某些东西的代价地图——即 只有“PolylineFlatmapCost”：

```json
{
 "graph": {
   "nodes: [
    {
     "name": "flatmap_cost",
     "start_order": -100,
     "components": [
       {
         "name": "inside_round",
         "type": "isaac::map::PolylineFlatmapCost"
       },
       {
         "name": "outside_round",
         "type": "isaac::map::PolylineFlatmapCost"
       }
     ]
   },
   ...
 "config": {
   "flatmap_cost": {
      "inside_round": {
        "polyline": [
          [36.0, 109.5], [36.0, 16.0], [34.5, 14.5], [16.0, 14.5], [14.5, 16.0],
          [14.5, 109.5], [16.5, 111.0], [34.5, 111.0], [36.0, 109.5]
        ],
        "width": 0.8,
        "penality_angle": 1.0,
        "penality_distance": 2.0,
        "outside_weight": 10.0
      },
      "outside_round": {
        "polyline": [
          [38.0, 110.0], [35.0, 113.0], [16.0, 113.0], [12.5, 110.0],
          [12.5, 15.5], [15.5, 12.5], [35.0, 12.5], [38.0, 15.5], [38.0, 110.0]
        ],
        "width": 0.8,
        "penality_angle": 1.0,
        "penality_distance": 2.0,
        "outside_weight": 10.0
      }
    },
    ...
  }
}

```

## 将通道添加到配置
如果要默认启用配置，则需要创建自己的 SightRenderer 或修改现有的：

```json
"flatsim.navigation.sight_widgets": {
   "Map View": {
     "type": "2d",
     "channels": [
       { "name": "map/occupancy/map" },
       { "name": "flatmap_cost/outside_round/area" },
       { "name": "flatmap_cost/outside_round/polyline" },
       { "name": "flatmap_cost/restricted_area/area" },
       { "name": "flatmap_cost/inside_round/polyline" },
       { "name": "flatmap_cost/inside_round/area" },
       { "name": "map/waypoints/waypoints" },
       { "name": "flatsim.navigation.localization.viewers/FlatscanViewer/beam_endpoints" },
       { "name": "flatsim.navigation.localization.viewers/FlatscanViewer2/beam_endpoints" },
       { "name": "flatsim.navigation.go_to.goal_viewer/GoalViewer/goal" },
       { "name": "flatsim.navigation.localization.viewers/RobotViewer/robot_model" },
       { "name": "flatsim.navigation.localization.viewers/RobotViewer/robot" },
       { "name": "flatsim.navigation.planner.viewers/Plan2Viewer/plan" },
       { "name": "flatsim.navigation.control.lqr/isaac.lqr.DifferentialBaseLqrPlanner/plan" }
     ]
   }
 },
```

更多精彩内容:
**[https://www.nvidia.cn/gtc-global/?ncid=ref-dev-876561](https://www.nvidia.cn/gtc-global/?ncid=ref-dev-876561)**
























