### 调用
- *调用(call)*是Scene性能调节配置的核心单元
- Scene执行配置入口时，总是从主文件`profile.json`入口开始
- 自上而下逐级匹配和执行对应的操作

#### 写入数值
  ```js
  // 示例 正常的写入一个值
  ["/proc/sys/kernel/sched_boost", "1"]
  ```
- 如果需要想要在写入数值后，将文件设为只读(0444)，可以在value前加#
  > 这么做主要是防止参数被第三方应用或系统再覆盖，并不影响Scene下次写入

  ```js
  ["/proc/sys/kernel/sched_boost", "#1"]
  ```


### 频率控制
- Scene内置了一些常用的频率调节函数
- 这些函数对(Qualcomm|MediaTek)等常见平台做了兼容适配
- 当然，如果你已经非常熟悉性能控制，直接写路径也是可以的

#### CPU频率范围 **`@cpu_freq`**
- 参数格式为 **@cpu_freq [clusterExpr] [freqExpr] [freqExpr]**
- 例如，我准备在省电模式下将CPU小核限制为最高900MHz
  ```json
  {
    "schemes": {
      "powersave": {
        "call": [
          ["@cpu_freq", "cpu0", "min", "900MHz"]
        ]
      }
    }
  }
  ```


### CPU最小频率 **`@cpu_freq_min`**
- 参数格式为 **@cpu_freq_min [clusterExpr] [freqExpr]**
- 例如，我准备在省电模式下将CPU小核限制为最低300MHz
  ```json
  {
    "schemes": {
      "powersave": {
        "call": [
          ["@cpu_freq_min", "cpu0", "300MHz"]
        ]
      }
    }
  }
  ```

#### CPU最大频率 **`@cpu_freq_max`**
- 参数格式为 **@cpu_freq_max [clusterExpr] [freqExpr]**
- 例如，我准备在省电模式下将CPU小核限制为最高900MHz
  ```json
  {
    "schemes": {
      "powersave": {
        "call": [
          ["@cpu_freq_max", "cpu0", "900MHz"]
        ]
      }
    }
  }
  ```

#### CPU各Cluster的最小频率 **`@cpu_freqs_min`**
- 参数格式为 **@cpu_freq_max [freqExpr] [freqExpr] ...**
  ```json
  ["@cpu_freqs_min", "300000", "300000", "300000"]
  ```

#### CPU各Cluster的最大频率 **`@cpu_freqs_max`**
- 参数格式为 **@cpu_freq_max [freqExpr] [freqExpr] ...**
  ```json
  ["@cpu_freqs_min", "2000000", "2400000", "2800000"]
  ```

> 以上cpu频率控制函数没有优劣之分，根据实际需求和喜好选择使用即可

<br>

#### GPU频率范围 **`@gpu_freq`**
- 参数格式为 **@gpu_freq [freqExpr] [freqExpr]**
- 例如，我准备在省电模式下将GPU最大频率限制500MHz
  ```json
  {
    "schemes": {
      "powersave": {
        "call": [
          ["@gpu_freq", "min", "500MHz"]
        ]
      }
    }
  }
  ```

#### GPU最小频率 **`@gpu_freq_min`**
- 参数格式为 **@gpu_freq_min [freqExpr]**
- 例如，我准备在极速模式下将GPU最低频率限制为400MHz
  ```json
  {
    "schemes": {
      "fast": {
        "call": [
          ["@gpu_freq_min", "400MHz"]
        ]
      }
    }
  }
  ```


#### GPU最大频率 **`@gpu_freq_max`**
- 参数格式为 **@gpu_freq_max [freqExpr]**
- 例如，我准备在省电模式下将GPU最膏限制为最高300MHz
  ```json
  {
    "schemes": {
      "powersave": {
        "call": [
          ["@gpu_freq_max", "400MHz"]
        ]
      }
    }
  }
  ```

#### 补充说明
- 频率写入校验
  > 假设，CPU0此时已经被限制为 [600MHz ~ 1.2GHz]<br>
  > 我们调用 **`@cpu_freq_min` cpu0 1.5GHz** 可能不会成功<br>
  > 因为 1.5GHz 不符合 [600MHz ~ 1.2GHz]这个区间<br>
  > 建议使用 **`@cpu_freq` cpu0 1.5GHz 2.0GHz** 直接指定CPU频率范围

- clusterExpr 说明
  > 我们修改CPU频率通常是以核心集群(Cluster)为单位进行的<br>
  > 例如，4+3+1的骁龙处理器，小、中、大核可以用以下几种格式来表示<br>
  > `cpu0`、`cpu4`、`cpu7`<br>
  > `policy0`、`policy4`、`policy7`<br>
  > `cluster0`、`cluster1`、`cluster2`<br>
  > 但是，千万不要用意义不明的数字来表示，例如 `0`、`4`、`7`，这是非常不利于理解

- freqExpr 说明
  > 为了更方便的表示频率，Scene内置了多种格式兼容和特殊值<br>
  > 例：`min`、`max` 分别表示该核心(Cluster)支持的最小、最大频率<br>
  > 例：`1800MHz`、`1.8GHz` 均等同于 `1800000KHz` 或 直接写 `1800000`<br>
  > 如果你指定了一个不存在的频率，那么Scene会帮你选择低于指定频率的最大频率<br>
  > 又或者，指定的频率比核心支持的最高频率还要高，那么Scene会帮你选择支持的最高频率<br>
  > 又或者，指定的频率比核心支持的最低频率还低，那么Scene会帮你选择支持的最低频率<br>

  > 但是，小心！用`GHz|MHz`表示频率虽然非常方便，但你可能会掉进陷阱<br>
  > 因为CPU的频率经常不是 2.8GHz(2800000KHz) 这样整齐的数字，更多是 2841600KHz这样<br>
  > 如果你写2.8GHz，是不会匹配到2841600KHz这个频率的！<br>

  > 负值频率<br>
  > 这是Scene特有的频率表达方式，它是指在 `max` 的基础上减去一个频率<br>
  > 例如 **-300MHz**、**-0.3GHz** 、**-300000** <br>
  > 如果，小核支持的最高频率为1800MHz<br>
  > 那么 **["@cpu_freq", "cpu0", "-1200MHz", "-300MHz"]** 等效于 **["@cpu_freq", "cpu0", "600MHz", "1500MHz"]**<br>
  > 如果，小核支持的最高频率为2000MHz<br>
  > 那么 **["@cpu_freq", "cpu0", "-1200MHz", "-300MHz"]** 等效于 **["@cpu_freq", "cpu0", "800MHz", "1700MHz"]**
