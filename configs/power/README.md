# Scaling Available Frequencies (Exynos9820)

See https://www.kernel.org/doc/Documentation/power/pm_qos_interface.txt

> Alternatively the user mode program could write a hex
> string for the value using 10 char long format e.g. "0x12345678".

## CPU Little Cluster (CPU0)

               0x12345678

    1950000 -> 0x001dc130
    1846000 -> 0x001c2af0
    1742000 -> 0x001a94b0
    1586000 -> 0x00183350
    1456000 -> 0x00163780
    1300000 -> 0x0013d620
    1157000 -> 0x0011a788
    1053000 -> 0x00101148
     949000 -> 0x000e7b08
     806000 -> 0x000c4c70
     650000 -> 0x0009eb10
     546000 -> 0x000854d0
     442000 -> 0x0006be90

## CPU Big Cluster (CPU4)

    2314000 -> 0x00234f10
    2210000 -> 0x0021b8d0
    2106000 -> 0x00202290
    2002000 -> 0x001e8c50
    1898000 -> 0x001cf610
    1794000 -> 0x001b5fd0
    1690000 -> 0x0019c990
    1586000 -> 0x00183350
    1508000 -> 0x001702a0
    1404000 -> 0x00156c60
    1222000 -> 0x0012a570
    1066000 -> 0x00104410
     962000 -> 0x000eadd0
     845000 -> 0x000ce4c8
     754000 -> 0x000b8150
     650000 -> 0x0009eb10
     507000 -> 0x0007bc78

## CPU Big Plus Cluster (CPU6)

    2730000 -> 0x0029a810
    2600000 -> 0x0027ac40
    2530000 -> 0x00269ad0
    2470000 -> 0x0025b070
    2340000 -> 0x0023b4a0
    2236000 -> 0x00221e60
    2080000 -> 0x001fbd00
    1976000 -> 0x001e26c0
    1820000 -> 0x001bc560
    1664000 -> 0x00196400
    1560000 -> 0x0017cdc0
    1456000 -> 0x00163780
    1378000 -> 0x001506d0
    1248000 -> 0x00130b00
    1144000 -> 0x001174c0
    1040000 -> 0x000fde80
     936000 -> 0x000e4840
     819000 -> 0x000c7f38
     728000 -> 0x000b1bc0
     624000 -> 0x00098580
     520000 -> 0x0007ef40

## GPU Scaling

    702000 -> 0x0000ab630
    650000 -> 0x00009eb10
    572000 -> 0x00008ba60
    433000 -> 0x000069b68
    377000 -> 0x00005c0a8
    325000 -> 0x00004f588
    260000 -> 0x00003f7a0
    200000 -> 0x000030d40
    156000 -> 0x000026160

## Utilization clamping (uclamp)

Measured on the mainline kernel (7.3, schedutil, energy-aware scheduling on), one run per point on
an idle phone. A temporary cpu cgroup carried a load pinned to one CPU of a cluster; the
frequency is the dominant cpufreq state while the load ran.

The frequency follows the clamp linearly, with no headroom margin:
`f = fmax * (clamp% / 100) * 1024 / cpu_capacity`, rounded up to the next operating point.
The little, mid and big clusters have capacity 355, 693 and 1024 (34.7%, 67.7% and 100% of the
scale), so the same percentage means very different frequencies.

`cpu.uclamp.min` (MHz, little / mid / big):

     0%   442 /  507 /  520
     5%   442 /  507 /  520    (below every cluster's lowest step: no effect on frequency)
    10%   650 /  507 /  520
    20%  1157 /  754 /  624
    30%  1742 / 1066 /  819
    40%  1950 / 1404 / 1144
    60%  1950 / 2106 / 1664
   100%  1950 / 2314 / 2730

`cpu.uclamp.max` caps the frequency the same way. It has no effect above a cluster's capacity
(above 34.7% on little, above 67.7% on mid).

Placement follows the floor. A light unclamped task stays on the little cluster (about 95%). Any
floor of 10% or more moves it to the mid cluster, and a floor above the mid capacity (67.7%)
moves it to the big cluster, which has only two CPUs.

Values chosen here:

 - top-app `cpu.uclamp.min` 5%: the idle default. It does not change frequency; the latency
   sensitive flag is what carries over from the old schedtune prefer_idle.
 - INTERACTION 30%: touch boost, runs top-app on the mid cluster at about 1.1 GHz.
 - LAUNCH 50%: app launch, mid cluster at about 1.7 GHz. It stays below the mid capacity on
   purpose: a floor of 70% would only fit the two big CPUs and could pile every launch thread
   onto them.
 - ADPF sessions use per-thread clamps from the power HAL (powerhint.json, AdpfConfig, 0-1024
   scale). Init 100 stays on little, High 384 is about 1.3 GHz on mid, the surfaceflinger session
   at 157 sits just above the 10% step that moves it to mid.

The task profiles in task_profiles.json are the ones the power HAL applies. SCHED_QOS_SENSITIVE_EXTREME
is applied to the threads of the SystemUI, HWUI and SurfaceFlinger hint sessions. Those processes run in
the foreground cpu cgroup, which has no floor and no latency flag, so the top-app floors never reached
them. It is an alias of AOSP's MaxPerformance profile, which joins the top-app cpu cgroup. PreferIdleSet,
SCHED_QOS_SENSITIVE_STANDARD and SCHED_QOS_NONE are Pixel scheduler tags with no counterpart here and stay empty
(apps in the foreground are already in top-app; the profile cannot know a thread's earlier cgroup). The
per-thread clamps of a hint session are set by the HAL itself with sched_setattr.

Not measured: the effect of the real-time default (sched_util_clamp_min_rt_default is 0 in
init.exynos9820.rc) on audio threads, and the power cost of any value above.
