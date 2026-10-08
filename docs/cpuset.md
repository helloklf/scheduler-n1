## 线程CPU核心配置 `cpuset`
- 你可能发现，绝大多数Unity游戏`UnityMain`线程CPU占用极高
- 正常情况下内核会根据实际负载需要，决定要不要将任务迁移到`Big`核心
- 但有些系统会为了节省电力故意降低调用`Big`核心的积极性
- 针对这种情况，Scene实现了`核心分配`主动分析负载为线程分配合适的CPU核心


### threads.json
- 支持通过`threads.json`人工指定线程使用哪些CPU核心，替代核心分配的自动分配规则
- Scene N1相较于Scene9的配置格式变化
```txt
废弃了 `unity_main` 和 `rr`。其中 `unity_main`字段仍然生效，但不再推荐使用。
新增了 heaviest_thread 和 heaviest_cores
因此，即使是在非Unity游戏里，也能提供两个重要线程的配置
```

#### 针对游戏
- 通过`cpuset: { ... }`配置适用于游戏的分配规则，示例：

  ```json
  [
    {
      "friendly": "原神",
      "categories": ["GenshinImpact"],
      "cpuset": {
        "heaviest_thread": "UnityMain",
        "heaviest_cores": "7",
        "heavy_thread": "UnityGfx",
        "heavy_cores": "4-6",
        "comm": {
          "4-6": ["UnityMultiRende", "mali-cmar-backe"],
          "0-3": ["Worker Thread", "AudioTrack", "Audio"]
        },
        "trashy": ["Async"],
        "other": "0-6"
      }
    },
    {
      "friendly": "王者荣耀",
      "packages": ["com.tencent.tmgp.sgame"],
      "cpuset": {
        "heaviest_thread": "UnityMain",
        "heaviest_cores": "7",
        "heavy_thread": "UnityGfx;Thread-",
        "heavy_cores": "6",
        "comm": {
          "7": ["UnityMain"],
          "6": ["UnityPreload", "Thread-"],
          "3-5": ["Worker Thread", "NativeThread", "Audio"]
        },
        "other": "0-6"
      }
    }
  ]
  ```

| 参数 | 解释 | 类型 |
| :- | :- | :-: |
| `heaviest_thread` | 负载最重的线程名称 | string |
| `heaviest_cores` | 负载最重的线程使用的cpu核心 | string |
| `heavy_thread` | 第二重负载线程的名称 | string |
| `heavy_cores` | 第二重负载线程可以使用的cpu核心 | string |
| `main_thread` | 指定运行主线程的cpu核心(通常，游戏的主线程都不是重负载线程) | string |
| `other` | 未被comm、heaviest_thread、heavy_thread命中的线程可使用的CPU核心 | string |
| `trashy` | 指定垃圾线程，本质上是通过cpuctl(cpu.uclamp.max)对线程进行限制，因此需要LinuxKernel 5+ | string[] |
| `ni` | 需要通过修改nice值提高优先级的线程 | string[] |

- heaviest_thread 和 heavy_thread 支持匹配多个线程名称，以英文 ; 分隔
- 例如："Thread-;UnityGfx"。如果匹配到多个线程，则命中负载最高的那一个。
- 但是注意，heaviest_thread 和 heavy_thread 的匹配规则不可重叠
  ```json
  // 错误示例 ❌
  {
    "heaviest_thread": "Thread-;UnityGfxDevice",
    "heavy_thread": "Thread-"
  }
  ```
- 如果一个游戏里，真的有很多个叫 Thread-xx 的负载线程怎么办呢？建议不为这类游戏写配置，让Scene自动分析和决策

- 使用 heaviest_thread 和 heavy_thread 匹配重要线程，除了完成核心绑定
- 还**可能**附带 `优先使用L2/L3；提高优先级；作用于FAS和辅助调速器的[性能需求提示]` 这些额外联动优化效果
  ```json
  // 推荐做法 ✅️ 专用配置位，表明线程重要性，不要嫌格式啰嗦
  {
    "heaviest_thread": "UnityMain",
    "heaviest_cores": "7",
    "heavy_thread": "UnityGfx",
    "heavy_cores": "6",
    "comm": {
      "0-3": ["Async-", "Pool-"]
    },
    "other": "0-5"
  }
  // 不推荐做法 ⚠️ 无法区分线程重要性，无额外联动优化效果
  {
    "comm": {
      "7": ["UnityMain"],
      "6": ["UnityGfx"],
      "0-3": ["Async-", "Pool-"]
    },
    "other": "0-5"
  }
  ```


##### ni
- `ni`会为匹配到的线程设置nice值为`-10`，提高争夺CPU资源的能力
- `ni`策略默认包含 heaviest_thread 和 heavy_thread 命中的线程，无需重复声明
- 你只需要补充除二者以外的重要线程，如果没有就不配置，例如：

  ```json
  {
    "friendly": "王者荣耀/LOLM",
    "packages": ["com.tencent.tmgp.sgame", "com.tencent.tmgp.sgamece", "com.garena.game.kgtw", "com.tencent.lolm"],
    "cpuset": {
      "ni": ["CoreThread"],
      "heaviest_thread": "UnityMain",
      "heaviest_cores": "7",
      "heavy_thread": "UnityGfx;Thread-",
      "heavy_cores": "6",
      "comm": {
        "4-7": ["UnityPreload", "IL2CPP", "btm_"],
        "4-5": ["CoreThread", "Apollo-", "NativeThread", "UnityChoreo", "Worker Thread", "Job.worker"]
      },
      "other": "0-5"
    }
  }
  ```


#### 针对一般应用
- `app_cpuset: { ... }`它更适用于为普通应用，设置项更少，性能开销也更低。
- 它的设计思路为：
  > 将App中的线程分为 `主线程` `渲染线程` `其它线程`<br>
  > 用户触摸屏幕时，快速将线程迁移到指定核心，停止交互后恢复初始状态<br>
  > 从而避免在非交互状态依然长期占用高性能核心增加额外功耗<br>

  ```json
  [
    {
      "friendly": "优先大核",
      "packages": [
        "com.tencent.mobileqq",
        "com.tencent.qqlive", "tv.danmaku.bili",
        "com.taobao.taobao", "com.jingdong.app.mall",
        "com.tencent.news", "com.sina.weibo",
        "com.netease.cloudmusic", "com.miui.player",
        "com.tencent.mm",
        "com.miui.notes",
        "com.sankuai.meituan"
      ],
      "app_cpuset": {
        "main": "7",
        "render": "4-6",
        "other": "0-6",
        "webview": false,
        "children": false
      }
    }
  ]
  ```

| 参数 | 解释 | 类型 |
| :- | :- | :-: |
| `main` | 主线程使用的核心| string |
| `render` | 渲染线程使用的核心 | string |
| `other` | 其它线程使用的核心 | string |
| `webview` | 是否检测webview沙盒进程中的渲染线程 | string |
| `children` | 是否检测子进程 | string |

- 与 `辅助调速器` 联动的 `性能需求提示`
> Scene会跟踪应用主线程和负载最重的RenderThread <br>
> 如果线程持续运行，超出绘制1帧所需的时间，则提示辅助调速器提升性能


#### sched_ext
- sched_ext 是 Scene N1尚在试验阶段的新功能，当前仅测试适配Linux Kernel 6.12
- 且由于调速器部分尚未完成，需要搭配FAS Lite，或所有核心调速器使用performance的Scene FAS使用
- 注意：sched_ext 优先级低于cgroup/affinity，确保系统自身的线程放置已经完全关闭，并卸载其它线程管理模块
- 以下是配置案例

  ```json
  {
    "friendly": "王者荣耀/LOLM",
    "packages": ["com.tencent.tmgp.sgame", "com.tencent.tmgp.sgamece", "com.tencent.lolm"],
    "cpuset": {
      "heaviest_thread": "UnityMain",
      "heaviest_cores": "7",
      "heavy_thread": "UnityGfx;Thread-",
      "heavy_cores": "1-5",
      "comm": {
        "1-7": ["IL2CPP", "btm_"]
      },
      "other": "1-5"
    },
    "scx": {
      "heaviest": {
        "comm": "UnityMain",
        "cpu": 7,
        "cpus": "1-7",
        "preempt": true
      },
      "heavy": {
        "comm": "UnityGfx;Thread-",
        "cpu": 5,
        "cpus": "1-7",
        "preempt": true
      },
      "other": {
        "cpus": "0-4"
      }
    }
  }
  ```

- heaviest 和 heavy 可配置的参数
  | 参数 | 必须配置 | 说明 |
  | - | - |
  | comm | Y | 匹配的线程名称，可通过;分隔配置多个，并自动挑选匹配的线程中负载最高的1个 |
  | cpu | Y | 目标线程*优先使用的cpu核心* |
  | cpus | N | 目标线程*优先使用的cpu核心* |
  | preempt | N | 是否启用抢占，启用则自动踢开*优先使用的cpu核心*上的其它线程 |

- scx 配置和 cpuset配置不会同时生效，同时配置只是提供回退选项。支持sched_ext的内核用scx，不支持则继续用cpuset配置


#### threads_auto.json 自动策略
- 在未提供预设的线程放置策略预设的游戏中，Scene会自动分析负载分布
- 将负载识别为以下类型
  定义  | 必须配置 | 负载类型说明 |
  | :- | :-: | :- |
  | load1n | N | 具有1个重负载线程 |
  | load11n | N | 具有1个重负载线程，和1个中负载线程 |
  | load12n | N | 具有1个重负载线程，和2个中负载线程 |
  | load2n | N | 具有2个重负载线程 |
  | load3n | N | 具有3个或更多的负载线程 |
- Scene可能会随版本更新调整分配策略，通常不需要通过配置干预
- 但有时候会有类似“6+2的处理器，应该同时用2个大核还是只用1个大核”这样的争议
  > 注意：“6+2处理器是否应该同时使用两个大核，需要考虑具体型号和游戏，这里只抛出问题不作指示和建议”
- 因此你可以创建 `threads_auto.json` 来改变默认策略

- 每个类型均可按top4和other指定可以使用的CPU核心，格式如
  ```json
  {
    "primary": "7",
    "secondary": "6",
    "tertiary": "4-5",
    "quaternary": "4-5",
    "other": "0-5",
  }
  ```

- 对于非游戏应用，线程识别和CPU核心分配策略就简单的多
- 主要通过线程ID/名称区分为 主线程、渲染线程、其它线程
  ```json
  {
    "main": "7",
    "render": "4-6",
    "other": "0-6"
  }
  ```

- 下面是个配置示例

  ```json
  {
    "game": {
      "load1n": {
        "primary": "7",
        "secondary": "1-6",
        "tertiary": "1-6",
        "quaternary": "1-6",
        "others": "1-6"
      },
      "load11n": {
        "primary": "7",
        "secondary": "6",
        "tertiary": "1-5",
        "quaternary": "1-5",
        "others": "1-5"
      },
      "load12n": {
        "primary": "7",
        "secondary": "1-6",
        "tertiary": "1-6",
        "quaternary": "1-5",
        "others": "1-5"
      },
      "load2n": {
        "primary": "7",
        "secondary": "6",
        "tertiary": "1-5",
        "quaternary": "1-5",
        "others": "1-5"
      },
      "load3n": {
        "primary": "1-7",
        "secondary": "1-7",
        "tertiary": "1-7",
        "quaternary": "1-7",
        "others": "0-5"
      }
    },
    "app": {
      "main": "1-7",
      "render": "1-7",
      "other": "1-7"
    }
  }
  ```
