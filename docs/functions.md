# 函数列表

- @ioctl
  > ioctl写入int值
  ```json
  ["@ioctl", "/proc/cpuqosmgr/cpuqos_ioctl", "0x4014670E", "1"]
  // 支持一次性指定多个写入，格式如 [@ioctl] [path] [cmd] [value] [cmd] [value]
  ["@ioctl", "/proc/easmgr/eas_ioctl",
    "0x40046715", "500000",
    "0x40046717", "500000",
    "0x40046719", "1000000"
  ]
  ```

- @shell
  > 运行shell脚本
  ```json
  ["@shell", "/xxx/yyy/zzz.sh"]
  ```

- @msm_reset @mtk_reset @xring_reset
  > 清理函数，通常是去除一些阻碍控制性能的限制
  ```json
  // 骁龙
  ["@msm_reset"]
  // MTK
  ["@mtk_reset"]
  // XRing
  ["@xring_reset"]
  ```

- @ttj
  > 设置CPU/GPU温度墙，适用于少数处理器，如：天玑8100~9500，骁龙845 855
  ```json
  ["@ttj", "85000"]
  ```

- @set_value
  > 向路径写入值
  ```json
  ["@set_value", "/xx/yy/zz", "12345"]
  // 一般简写为
  ["/xx/yy/zz", "12345"]
  ```

- @cpu_freq_min
  ```json
  ["@cpu_freq_min", "cpu0", "300000"]
  ```
- @cpu_freq_max
  ```json
  ["@cpu_freq_max", "cpu0", "2000000"]
  ```

- @cpu_freqs_min
  > 设置cpu各个cluster的频率下限
  ```json
  ["@cpu_freqs_min", "300000", "300000", "300000"]
  ```

- @cpu_freqs_max
  > 设置cpu各个cluster的频率上限
  ```json
  ["@cpu_freqs_min", "2000000", "2420000", "2820000"]
  ```

- @cpu_freq
  > 设置某个cluster的频率上下限
  ```json
  ["@cpu_freq", "cpu0", "300000", "2000000"]
  ```

- @gpu_freq
  > 设置GPU的频率上下限
  ```json
  ["@gpu_freq", "120000", "750000"]
  ```

- @gpu_freq_min
  ```json
  ["@gpu_freq", "120000"]
  ```

- @gpu_freq_max
  ```json
  ["@gpu_freq", "750000"]
  ```

- @cpuset
  ```json
  // 设置 background 、system-background 、foreground、top-app 组可使用的cpu核心
  ["@cpuset", "0-3", "0-6", "0-7", "0-7"]
  ```

- @limiter
  ```json
  // 清除所有辅助调速器，NONE可以省略，加上提高可读性
  ["@limiter", "NONE"]
  // 启用指定名称的一组辅助调速器
  ["@limiter", "p1"]
  ```

- @ddr_l3_booster `变更`
  > L3/DDR Boost(可能需要配合CPU/DDR频率映射配置)
  ```json
  // 启用，可分别设置 [ddr] [l3] Boost是否启用
  ["@ddr_l3_booster", "true", "true"]
  // 停用
  ["@ddr_l3_booster", "false", "false"]
  // 等效于全 false
  ["@ddr_l3_booster"]
  ```

- @core_ctl `相对Scene9 有变更`
  ```json
  // 启用，设置[cluster0], [cluster1]...是否启用CoreCtl
  ["@core_ctl", "false", "false", "true"]
  // 禁用
  ["@core_ctl", "false", "false", "false"]
  // 等效于全false
  ["@core_ctl"]
  ```

- @input_detect `取代Scene9的 @gesture`
  > 临时启用或禁用输入检测(在不需要切换active/inactive状态时用，减少检测开销)
  ```json
  // 启用
  ["@input_detect", "true"]
  // 禁用
  ["@input_detect", "false"]
  ```

- @governor
  > 切换cpu调速器，可配置多个。会从前到后自动匹配第一个支持的
  ```json
  ["@governor", "walt", "schedutil"]
  ```

- @sched_limit
  > 分别设置 [cluster0], [cluster1]... 的升&降频延迟(rate_limit_us)
  ```json
  ["@sched_limit", "1000", "1000", "2000"]
  ```

- @target_loads
  > 分别设置 [cluster0], [cluster1]... 的目标负载
  ```json
  ["@target_loads", "80 1800000:90", "80 2100000:95", "85 2100000:95"]
  ```

- @hispeed_freq
  > 分别设置 [cluster0], [cluster1]... 的hispeed_freq(需要调速器自身支持)
  ```json
  ["@target_loads", "1200000", "1200000", "1200000"]
  ```

- @hispeed_load
  > 分别设置 [cluster0], [cluster1]... 的hispeed_load(需要调速器自身支持)
  ```json
  ["@target_loads", "95", "95", "95"]
  ```

- @uclamp
  > 分别设置[background]、[foreground]、[top-app] 3个组的 cpu.uclamp.`min/max`
  ```json
  ["@uclamp", "0.0~max", "0.0~max", "0.0~max"]
  ```

- @cpu_online
  > 设置各个核心的online状态
  ```json
  // 前面不用变状态的核心可以留空，如果核心在后面也也可以省略
  ["@uclamp", "1", "1", "0", "0", "1", "1"]
  ```

- @mtk_renew
  > 一个特殊的函数，用于还原性能控制参数所有者和selinux context。这是因为天玑部分处理器，如果系统perfhal写入值失败，会一直重试可能导致待机耗电异常。主要在D8100和D9000上用到
  ```json
  ["@mtk_renew"]
  ```

- @preset
  > 使用若干组参数设定
  ```json
  // ddr_freq_middle 和 powersave_active 都是自定义参数组，scene里并不存在这样的预设
  ["@apply", "ddr_freq_middle", "powersave_active"]
  ```

- @values
  > 使用一组参数设定，并将指定的值于参数组路径结合
  ```json
  ["@values", "set_xx_freq", "100000", "500000"]
  ```

- @apply
  > 和@preset类似，区别是@preset会对路径写入去重，剔除框架认为会被覆盖的写入，@apply则不管是否重复一律写入
  ```json
  ["@apply", "powersave_active"]
  ```

- @repeat
  > 将某个预设重复应用两次，主要是用于应对存在大小值互相校验，可能导致写入失败的场景
  ```json
  ["@repeat", "xxx"]
  ```

- @fps
  > 限制当前应用画面渲染帧数，当前刷新率必须可以被设定的数值整除，否则无效
  ```json
  ["@fps", "30"]
  ```

- @chmod
  > 用于将某个节点恢复为可写状态
  ```json
  ["@chmod", "/xx/yy/zz", "644"]
  ```
