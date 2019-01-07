## What's wrong with clock_gettime/gettimeofday?
Although current Linux systems are using VDSO for implementing clock_gettime/gettimeofday, they still have a nonnegligible overhead(usually more than 100 ns), and the latency is quite unstable(usually between 20 ~ 1000 ns) among different invocations in different frequency, and users have observed discrepant latency among different cpu cores under certain system configuration(e.g. isolcpus). Run `test.cc` to see how `clock_gettime` behaves on your machine.

All of these are not good for recording nanosecond timestamps in time-critical tasks where latency of getting timestamp itself should be minimized, nor for benchmarking a general program where stable and reliable timestamp latency is expected.

## How is TSCNS better?
TSCNS uses rdtsc instruction and simple arithmatic operations to get the same nanosecond timestamp provided by clock_gettime, it's much faster and stable in terms of latency: around 10 ns! Actually the whole work is in just 8 consecutive CPU instructions without a function call on current X86-64 architecture.

TSCNS is also thread safe in the most simple and efficient way: it's read-only after initialization, so different threads can keep TSCNS's member variable in the cache for as long as they can.

The precision can be assured by careful calibration which we'll talk about later.

## How does TSCNS calibrate time?
The most important factor in TSCNS is **tsc frequency** on the system, which it uses to convert tsc to ns. Linux kernel also maintains the same varable named `int tsc_khz` in `arch/x86/kernel/tsc.c`, which can be explored by `dmesg | grep "tsc: Refined TSC clocksource calibration"`. Can we just read the frequency in dmesg and use it? No, because it's not accurate: As you can see, kernel measures the frequency in units of 1000, so you can only get a number with 7 digits precision. If you use that value, your ns timestamp will drift apart from system time. But if kernel maintains tsc frequency in khz, how could your timestamp drift with the same value? Because kernel uses int32 multiplication and shift operation to simulate a floating point multiplication(because kernel is prohibited from using floating point), it would result in slight computational error, which in turn results in additional floating point digits after the 7 ones that we couldn't see. 

That said, TSCNS needs to find a way to find out those invisible digits: it calibrates. Just like how kernel calibrates its tsc clocksource with the help of hpet, TSCNS synchronizes its user space tsc with kernel tsc(it records two pairs of timestamps in different times to calculate the slope), but in a much more accurate manner(If kernel is not accurate, user has to accept; But if TSCNS is not accurate, user could be upset). 

So initially TSCNS's user need to wait some time before calling `calibrate()` which returns the resultant tsc ghz in a double value, then user can simply feed this frequency to TSCNS for future use without waiting. The long time user waits in the first run the more precise frequency calibration can get, and the returned tsc requency could be saved in a config file on the machine, because it remains valid until hardware or kernel is upgraded when user should calibrate again.

## Any limitations?
Yes.
1) The CPU must have `constant_tsc` feature, which can searched in `/proc/cpuinfo`.
2) TSCNS doesn't support NTP or other kinds of system time change after it's been initialized, it's always going forward in a steady speed, perhaps it's instead an upside to some users.

## Usage
Initialization if tsc_ghz is unknown or not saved previously:
```C++
TSCNS tn;

tn.init();
// sleep for some time or do other initialization work...
std::this_thread::sleep_for(std::chrono::seconds(1));
double tsc_ghz = tn.calibrate();
// now we can save tsc_ghz for future initialization

```

Initialization if tsc_ghz is known:
```C++
TSCNS tn;

tn.init(tsc_ghz);
```

Getting nanosecond timestamp in one step:
```C++
uint64_t ns = tn.rdns();
```

Or just recording a tsc in time-critical tasks and converting it to ns in tasks which can be delayed:
```C++
// in time-critical task
uint64_t tsc = tn.rdtsc();

...

// in logging task
uint64_t ns = tn.tsc2ns(tsc);
```

## Test
`test.cc` shows:
1) Your system's tsc_ghz
2) TSCNS's latency
3) clock_gettime's latency and how unstable it could be
4) If your tsc_ghz is precise enough and if ns from TSCNS is synced up with ns from system
5) If tsc from different cores are synced up

Try running it.
