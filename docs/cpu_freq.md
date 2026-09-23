## 频率控制
- Scene N1设计了全新的CPU频率控制器，使得它可以无限制的拓展各种“限频”或“提频”策略
- 修改CPU频率不再是“后设置”覆盖“先设置”的的单一路径，而是多种“条件”或者说“理由”的聚合
  ```sh
  FreqMgrUniversalLimit // 通用的(配置文件中中设定的常态频率)
  FreqMgrSensorLimit    // 由Sensor设定触发的限制(由Sensor配置的策略调用)
  [x] FreqMgrCpuLimiter     // 辅助调速器 - 基于负载的调速器轮询赋值(由辅助调速器调用)
  FreqMgrFASLimit       // FAS - 基于帧的调速器轮询赋值(由SceneFAS调用)
  FreqMgrGestureBoost   // 手势Boost(触发屏幕边缘手势时，应用配置中声明的参数)
  FreqMgrLauncherBoost  // 应用启动Boost(LauncherBoost自动调用)
  FreqMgrFastShareBoost // 文件传输Boost(FastShare传输文件时自动调用)
  FreqMgrReclaimBoost   // 主动Reclaim(增强LMK执行主动压缩时自动调用)
  FreqMgrIOBoost        // 文件操作(文件浏览进行文件操作时自动调用)
  FreqMgrKswapdBoost    // kswapd0持续活动Boost(由增强lmk模块调用)
  ```

- 这里的“理由”分为“Boost”和“Limit”两种
  > Limit的行为是设置"BoostMin|LimitMax"，Boost的行为是设置"BootMin|BoostMax"

- BoostMin的行为是提高频率下限
  > 例如：多个条件分别设置了 300 400 500，最终生效的是其中的最大值 500

- LimitMax的行为是限制频率上限
  > 例如：多个条件分别设置了 800 900 100，最终生效的事其中的最小值 800

- BoostMax的行为是提高频率下限
  > 例如：多个条件分别设置了 800 900 100，最终生效的事其中的最小值 1000

- BoostMax的意思是突破LimitMax设定的限制，
  > 例如：省电模式本该限制1.6GHz，但此时正在冷启动应用，就可以以`FreqMgrLauncherBoost`理由，短时间突破限制提升到更高频率


### FreqMgrUniversalLimit 通用的/基础 设定
  ```json
  // 在这里分别制定了省电模式下，一般app 和 游戏 中的CPU频率上限，
  // 它是一个通用设定，也就是说如果不触发什么特别的条件，CPU将一直保持这个限制
  {
    "schemes": {
      "powersave": {
        "call": [],
        "app": [
          ["@cpu_freqs_max", "1900800", "#2227200"]
        ],
        "game": [
          ["@cpu_freqs_max", "1900800", "#2820200"]
        ]
      }
    }
  }
  ```

### FreqMgrSensorLimit 由传感器触发的限制
  ```json
  // 在 FreqMgrUniversalLimit 的基础上，增加了Sensor定义，目的是根据电量调整CPU限制
  // 这里只针对原神这个游戏，写了一条规则，电量 < 15% 时，大核限制 2133MHz
  // 在电量低于15%时，大核限制 2133MHz 和 通用限制的 2820MHz叠加，CPU最终被限制在 2133MHz
  // 在电量高于15%时，CPU频率限制条件解除，频率恢复为 2820MHz
  {
    "schemes": {
      "powersave": {
        "call": [],
        "app": [
          ["@cpu_freqs_max", "1900800", "#2227200"]
        ],
        "game": [
          ["@cpu_freqs_max", "1900800", "#2820200"]
        ]
      }
    },
    "apps": [
      {
        "friendly": "原神",
        "packages": ["com.miHoYo.Yuanshen"],
        "sensors": [
          {
            "sensor": "capacity", // Scene阈值的传感器名称，代表电池电量%
            "interval": 5000, // 轮询间隔5s
            "rules": [
              {
                "threshold": [15, -1], // 电量 14 ~ 0% 之间
                "enter_once" : [
                  ["@cpu_freqs_max", "1900800", "#2133000"]
                ],
                "exit_once" : [
                  ["@cpu_freqs_max", "max", "max"]
                ]
              }
            ]
          }
        ]
      }
    ]
  }
  ```

### FreqMgrCpuLimiter
- 这个“理由”目前还没有被使用，辅助调速器目前也是通过 FreqMgrUniversalLimit 修改频率

### FreqMgrFASLimit FAS设定
- 由于Scene FAS需要完全控制频率，因此它从原理上不应和其它限制条件叠加
- Scene FAS会在启动前，重置 FreqMgrUniversalLimit 的限制条件，并以 FreqMgrFASLimit 动态调节频率限制

### 其它 “理由”
- 因为都是Scene主动调用，不需要人工配置，就不过多解释