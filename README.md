This project is about a new sub-optio in the `go tool trace` named "diagreedy" 
which means "diagnoses and finds out the greediest several goroutines". This tool 
already helped us tracked down several deep hiden problems in our go applications 
and achieved more stable and short GC pause latency by fixing them afterwards.

(You could discuss this feature proposal in this [go issue](https://github.com/golang/go/issues/20792) if you want it in the next release of golang.)

## Build

Same as [build golang from source](https://golang.org/doc/install/source).

```bash
$ cd src
$ ./all.bash
```

After build and test, then:

```
$ cd ../bin 
$ ./go tool trace -h
Usage of 'go tool trace':
Given a trace file produced by 'go test':
        go test -trace=trace.out pkg

Open a web browser displaying trace:
        go tool trace [flags] [pkg.test] trace.out

Generate a pprof-like profile from the trace:
    go tool trace -pprof=TYPE [pkg.test] trace.out

[pkg.test] argument is required for traces produced by Go 1.6 and below.
Go 1.7 does not require the binary argument.

Supported profile types are:
    - net: network blocking profile
    - sync: synchronization blocking profile
    - syscall: syscall blocking profile
    - sched: scheduler latency profile

Flags:
        -http=addr: HTTP service address (e.g., ':6060')
        -pprof=type: print a pprof-like profile instead
        -dump: dump all traced out as format text to stdout    <---
        -diagreedy=N: dump the topest N greedy goroutines      <--- you would see the new options 
```

Then there is a more detailed [example](#example).

## Terminology
```
N        = GOMAXPROCS 

Tmax_stw = sum{top max N on-processor time slices}

Tgc      = Tmax_stw + Tgcwork
```


## Big Picture
```
             t1=12           t2=11           t3=10
      |                |              |               | 
      __________________________________________________
      |                |              |                 |
      ---------------------------------------------------> time line 
P0    [------go 1------][-go 7-][-go 1-][-go 6-][-go 4-]
P1    [-go 3-][-go 4-][------go 2-----][-go 1-][-go 2-]
P2    [-go 4-][-go 3-][-go 8-][-go 9-][-----go 10-----]
```
In this diagram, the value of N is 3 (i.e. the GOMAXPROCS), and the top 3 long on-processor 
time slices is t1, t2, t3 and they are corresponding to goroutine 1, 2, 10.

The max time cost of the pre-GC-action Stop-the-World is named Tmax_stw and is the 
sum of t1, t2 and t3.

The Tgcwork is the time cost by the acually effective Garbage Collecting action after 
the phase Stop-the-World. Tgcwork is determined by the amount of the allocated objects 
and the complexity of the relationship among them.

Tmax_stw and Tgcwork together finally determined the one typical GC pause cost. As 
for Tgcwork, we could optimize it by reusing the objects (via sync.Pool for example) and 
by many other methods. But we do not discuss Tgcwork futher here. We concentrate on 
the Tmax_stw right now.

We could get this conclusion plainly:
    
    The bigger the Tmax_stw is and the greedier and less cooperative the goroutines are, 
    the more thrashing and churning our go applications would become, thus the worse 
    latency our programs would perform (caused by the GC pause).

For some cpu intensive applications, the top max N on-processor time slices is often 
very big (if without very carefully task splits). Especially when our project is including 
tons of other and heavily used open source projects which will cause it becomes more 
complex and harder to trace.

Although we already had some tools to sample out the top n cpu usage of the functions. 
But the accumulated cpu usage of the functions and the longest top n time slices is not 
the same thing at all. For example, if we have 1000 different cpu intensive goroutines 
something like:

    go func(){
        do_cpu_staff() -- time cost 1ms
        time.Sleep(time.Second)
    }()

Our Tmax_stw would become GOMAXPROCS*1 ms. It's terrible because sometimes 
we would set the GOMAXPROCS to a pretty big value (16 or even 36 for instance).
With the sample of top n cpu usage, we may couldn't find out the sources of the problem 
easily. But with the 'top max N on-processor time slices' approach, we could track down 
the sources of the long GC pause immediately and fix them like this:

    go func(){
        do_cpu_staff0() -- time cost 200us
        runtime.Gosched()
        do_cpu_staff1() -- time cost 200us
        runtime.Gosched()
        do_cpu_staff2() -- time cost 200us
        runtime.Gosched()
        do_cpu_staff3() -- time cost 200us
        runtime.Gosched()
        do_cpu_staff4() -- time cost 200us
        time.Sleep(time.Second)
    }()

Then our Tmax_stw would turns from GOMAXPROCS*1 ms to GOMAXPROCS/5 ms afterwards :)
(If your GOMAXPROCS is very big, just splits do_cpu_staff even further. And you even
could implement some logic which would let the do_cpu_staff automated splits its tasks
based on the value of GOMAXPROCS and a other parameter could named time_cost_wanted to 
solve this problem for good.)

## Example

And there is a example about the usage of `go tool trace -diagreedy`

```bash
$ go tool trace -h
Usage of 'go tool trace':
Given a trace file produced by 'go test':
        go test -trace=trace.out pkg

Open a web browser displaying trace:
        go tool trace [flags] [pkg.test] trace.out

Generate a pprof-like profile from the trace:
    go tool trace -pprof=TYPE [pkg.test] trace.out

[pkg.test] argument is required for traces produced by Go 1.6 and below.
Go 1.7 does not require the binary argument.

Supported profile types are:
    - net: network blocking profile
    - sync: synchronization blocking profile
    - syscall: syscall blocking profile
    - sched: scheduler latency profile

Flags:
        -http=addr: HTTP service address (e.g., ':6060')
        -pprof=type: print a pprof-like profile instead
        -dump: dump all traced out as format text to stdout   <-
        -diagreedy=N: dump the top N greedy goroutines     <- new added options
```
code to diagnose latterly:
```bash
package main

import "os"
import "fmt"
import "runtime"

import "runtime/trace"

//import "sync/atomic"

//import "math/big"
import "time"

func init() {
    //runtime.LockOSThread()
}

func main() {
    //debug.SetGCPercent(-1)
    runtime.GOMAXPROCS(2)

    go func() {
        for {
            fmt.Println("---------------")
            f, err := os.Create("trace_my_" +
                time.Now().Format("2006-01-02-15-04-05") + ".out")
            if err != nil {
                panic(err)
            }
            err = trace.Start(f)
            if err != nil {
                panic(err)
            }
            time.Sleep(time.Second / 1)
            trace.Stop()
            time.Sleep(time.Second * 2)
        }
    }()

    go func() {
        for {
            time.Sleep(time.Millisecond * 10)
            go func() {
                time.Sleep(time.Millisecond * 100)
            }()
        }
    }()

    go func() {
        for {
            fmt.Println("heartbeat... ",
                time.Now().Format("2006-01-02-15:04:05"))
            time.Sleep(time.Second)
        }
    }()

    // maybe highest cpu usage
    go func() {
        l := make([]int, 0, 8)
        l = append(l, 0)
        ct := 0
        for {
            l = append(l, ct)
            ct++
            l = l[1:]
            runtime.Gosched()
        }
    }()

    amount := 1000000
    granul := 100000 
    go godeadloop0(amount, granul)
    go godeadloop1(amount, granul)
    godeadloop2(amount, granul)
    return
}

// potential greediest go functions
func godeadloop0(amount, granul int) {
    func() {
        for {
            ct := 0
            for {
                ct++
                if ct%granul == 0 {
                    runtime.Gosched()
                }
                if ct >= amount {
                    fmt.Println("--->")
                    break
                }
            }
            time.Sleep(time.Second / 10)
        }
    }()
}

func godeadloop1(amount, granul int) {
    func() {
        for {
            ct := 0
            for {
                ct++
                if ct%granul == 0 {
                    runtime.Gosched()
                }
                if ct >= amount {
                    fmt.Println("--->")
                    break
                }
            }
            time.Sleep(time.Second / 10)
        }
    }()
}

func godeadloop2(amount, granul int) {
    func() {
        for {
            ct := 0
            for {
                ct++
                if ct%granul == 0 {
                    runtime.Gosched()
                }
                if ct >= amount {
                    fmt.Println("--->")
                    break
                }
            }
            time.Sleep(time.Second / 10)
        }
    }()
}
```

```bash
$go tool trace -diagreedy=10 /root/go/src/diaggreedy/trace_my_2017-06-25-10-01-13.out
finally the found toppest 10 greedy goroutines info

    7486357ns -> 0.007486s

        GoStart Ts: 739889373 P: 1 G: 9 StkID: 0 args0: 9 args1: 0 args2: 0
             GoSched
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95
        GoSched Ts: 747375730 P: 1 G: 9 StkID: 20 args0: 0 args1: 0 args2: 0
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95

    3617038ns -> 0.003617s

        GoStartLabel Ts: 649291006 P: 0 G: 27 StkID: 0 args0: 27 args1: 7 args2: 2
             GoStart -> GoBlock -> GoUnblock -> GoStartLabel -> GoBlock -> GoUnblock -> GoStartLabel -> GoBlock -> GoUnblock
             4285073 runtime.gcBgMarkWorker /usr/local/go/src/runtime/mgc.go 1435
             GC (fractional)
        GoBlock Ts: 652908044 P: 0 G: 27 StkID: 0 args0: 0 args1: 0 args2: 0
             GoStart -> GoBlock -> GoUnblock -> GoStartLabel -> GoBlock -> GoUnblock -> GoStartLabel -> GoBlock -> GoUnblock -> GoStartLabel
             4285073 runtime.gcBgMarkWorker /usr/local/go/src/runtime/mgc.go 1435

    2763199ns -> 0.002763s

        GoStart Ts: 925956546 P: 1 G: 10 StkID: 0 args0: 10 args1: 0 args2: 0
             GoSleep -> GoUnblock
             4452857 time.Sleep /usr/local/go/src/runtime/time.go 59
             4746323 main.godeadloop1.func1 /root/go/src/diaggreedy/x.go 112
             4744437 main.godeadloop1 /root/go/src/diaggreedy/x.go 114
        GoSched Ts: 928719745 P: 1 G: 10 StkID: 21 args0: 0 args1: 0 args2: 0
             4746333 main.godeadloop1.func1 /root/go/src/diaggreedy/x.go 105
             4744437 main.godeadloop1 /root/go/src/diaggreedy/x.go 114
    
    2573480ns -> 0.002573s

        GoStart Ts: 750086169 P: 0 G: 9 StkID: 0 args0: 9 args1: 0 args2: 0
             GoSched
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95
        GoSched Ts: 752659649 P: 0 G: 9 StkID: 20 args0: 0 args1: 0 args2: 0
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95

    2558799ns -> 0.002559s

        GoStart Ts: 625418894 P: 0 G: 9 StkID: 0 args0: 9 args1: 0 args2: 0
             GoSched
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95
        GoSched Ts: 627977693 P: 0 G: 9 StkID: 20 args0: 0 args1: 0 args2: 0
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95

    2462919ns -> 0.002463s

        GoStart Ts: 975882408 P: 0 G: 9 StkID: 0 args0: 9 args1: 97 args2: 0
             GoSched
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95
        GoSched Ts: 978345327 P: 0 G: 9 StkID: 20 args0: 0 args1: 0 args2: 0
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95

    2374319ns -> 0.002374s

        GoStart Ts: 240141393 P: 1 G: 10 StkID: 0 args0: 10 args1: 0 args2: 0
             GoSleep -> GoUnblock
             4452857 time.Sleep /usr/local/go/src/runtime/time.go 59
             4746323 main.godeadloop1.func1 /root/go/src/diaggreedy/x.go 112
             4744437 main.godeadloop1 /root/go/src/diaggreedy/x.go 114
        GoSched Ts: 242515712 P: 1 G: 10 StkID: 21 args0: 0 args1: 0 args2: 0
             4746333 main.godeadloop1.func1 /root/go/src/diaggreedy/x.go 105
             4744437 main.godeadloop1 /root/go/src/diaggreedy/x.go 114

    2250439ns -> 0.002250s

        GoStart Ts: 724664379 P: 0 G: 1 StkID: 0 args0: 1 args1: 0 args2: 0
             GoSched
             4746621 main.godeadloop2.func1 /root/go/src/diaggreedy/x.go 124
             4744517 main.godeadloop2 /root/go/src/diaggreedy/x.go 133
             4744280 main.main /root/go/src/diaggreedy/x.go 74
        GoSched Ts: 726914818 P: 0 G: 1 StkID: 19 args0: 0 args1: 0 args2: 0
             4746621 main.godeadloop2.func1 /root/go/src/diaggreedy/x.go 124
             4744517 main.godeadloop2 /root/go/src/diaggreedy/x.go 133
             4744280 main.main /root/go/src/diaggreedy/x.go 74

    2230559ns -> 0.002231s

        GoStart Ts: 382544542 P: 1 G: 9 StkID: 0 args0: 9 args1: 0 args2: 0
             GoSched
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95
        GoSched Ts: 384775101 P: 1 G: 9 StkID: 20 args0: 0 args1: 0 args2: 0
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95

    2112279ns -> 0.002112s

        GoStartLabel Ts: 866437288 P: 1 G: 28 StkID: 0 args0: 28 args1: 3 args2: 2
             GoStart -> GoBlock -> GoUnblock
             4285073 runtime.gcBgMarkWorker /usr/local/go/src/runtime/mgc.go 1435
             GC (fractional)
        GoBlock Ts: 868549567 P: 1 G: 28 StkID: 0 args0: 0 args1: 0 args2: 0
             GoStart -> GoBlock -> GoUnblock -> GoStartLabel
             4285073 runtime.gcBgMarkWorker /usr/local/go/src/runtime/mgc.go 1435
```
And then, we could found the source of the long GC pause godeadloop0 
and chage the variable granul to value 10000. Test again:

```bash
$ go tool trace -diagreedy=10 /root/go/src/diaggreedy/trace_my_2017-06-25-09-52-03.out

finally the found toppest 10 greedy goroutines info

    408760ns -> 0.000409s

        GoStart Ts: 564369612 P: 0 G: 8 StkID: 0 args0: 8 args1: 0 args2: 0
             GoSched
             4745644 main.main.func4 /root/go/src/diaggreedy/x.go 66
        GoSched Ts: 564778372 P: 0 G: 8 StkID: 21 args0: 0 args1: 0 args2: 0
             4745644 main.main.func4 /root/go/src/diaggreedy/x.go 66

    393760ns -> 0.000394s

        GoStart Ts: 363796356 P: 1 G: 8 StkID: 0 args0: 8 args1: 0 args2: 0
             GoSched
             4745644 main.main.func4 /root/go/src/diaggreedy/x.go 66
        GoSched Ts: 364190116 P: 1 G: 8 StkID: 21 args0: 0 args1: 0 args2: 0
             4745644 main.main.func4 /root/go/src/diaggreedy/x.go 66

    375760ns -> 0.000376s

        GoStartLabel Ts: 583805009 P: 0 G: 16 StkID: 0 args0: 16 args1: 3 args2: 2
             GoStart -> GoBlock -> GoUnblock
             4285073 runtime.gcBgMarkWorker /usr/local/go/src/runtime/mgc.go 1435
             GC (fractional)
        GoBlock Ts: 584180769 P: 0 G: 16 StkID: 0 args0: 0 args1: 0 args2: 0
             GoStart -> GoBlock -> GoUnblock -> GoStartLabel
             4285073 runtime.gcBgMarkWorker /usr/local/go/src/runtime/mgc.go 1435

    364040ns -> 0.000364s

        GoStart Ts: 211900654 P: 0 G: 10 StkID: 0 args0: 10 args1: 0 args2: 0
             GoSleep -> GoUnblock
             4452857 time.Sleep /usr/local/go/src/runtime/time.go 59
             4746323 main.godeadloop1.func1 /root/go/src/diaggreedy/x.go 112
             4744437 main.godeadloop1 /root/go/src/diaggreedy/x.go 114
        GoSched Ts: 212264694 P: 0 G: 10 StkID: 23 args0: 0 args1: 0 args2: 0
             4746333 main.godeadloop1.func1 /root/go/src/diaggreedy/x.go 105
             4744437 main.godeadloop1 /root/go/src/diaggreedy/x.go 114

    358280ns -> 0.000358s

        GoStart Ts: 736130271 P: 1 G: 9 StkID: 0 args0: 9 args1: 0 args2: 0
             GoSched
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95
        GoSched Ts: 736488551 P: 1 G: 9 StkID: 24 args0: 0 args1: 0 args2: 0
             4746045 main.godeadloop0.func1 /root/go/src/diaggreedy/x.go 86
             4744357 main.godeadloop0 /root/go/src/diaggreedy/x.go 95

    357360ns -> 0.000357s

        GoStart Ts: 551030053 P: 0 G: 8 StkID: 0 args0: 8 args1: 0 args2: 0
             GoSched
             4745644 main.main.func4 /root/go/src/diaggreedy/x.go 66
        GoSched Ts: 551387413 P: 0 G: 8 StkID: 21 args0: 0 args1: 0 args2: 0
             4745644 main.main.func4 /root/go/src/diaggreedy/x.go 66

    323360ns -> 0.000323s

        GoStart Ts: 103892587 P: 1 G: 1 StkID: 0 args0: 1 args1: 0 args2: 0
             GoSched
             4746621 main.godeadloop2.func1 /root/go/src/diaggreedy/x.go 124
             4744517 main.godeadloop2 /root/go/src/diaggreedy/x.go 133
             4744280 main.main /root/go/src/diaggreedy/x.go 74
        GoSched Ts: 104215947 P: 1 G: 1 StkID: 22 args0: 0 args1: 0 args2: 0
             4746621 main.godeadloop2.func1 /root/go/src/diaggreedy/x.go 124
             4744517 main.godeadloop2 /root/go/src/diaggreedy/x.go 133
             4744280 main.main /root/go/src/diaggreedy/x.go 74

    320520ns -> 0.000321s

        GoStart Ts: 736132431 P: 0 G: 1 StkID: 0 args0: 1 args1: 0 args2: 0
             GoSched
             4746621 main.godeadloop2.func1 /root/go/src/diaggreedy/x.go 124
             4744517 main.godeadloop2 /root/go/src/diaggreedy/x.go 133
             4744280 main.main /root/go/src/diaggreedy/x.go 74
        GoSched Ts: 736452951 P: 0 G: 1 StkID: 22 args0: 0 args1: 0 args2: 0
             4746621 main.godeadloop2.func1 /root/go/src/diaggreedy/x.go 124
             4744517 main.godeadloop2 /root/go/src/diaggreedy/x.go 133
             4744280 main.main /root/go/src/diaggreedy/x.go 74

    320480ns -> 0.000320s

        GoStartLabel Ts: 777280946 P: 0 G: 16 StkID: 0 args0: 16 args1: 5 args2: 2
             GoStart -> GoBlock -> GoUnblock -> GoStartLabel -> GoBlock -> GoUnblock
             4285073 runtime.gcBgMarkWorker /usr/local/go/src/runtime/mgc.go 1435
             GC (fractional)
        GoBlock Ts: 777601426 P: 0 G: 16 StkID: 0 args0: 0 args1: 0 args2: 0
             GoStart -> GoBlock -> GoUnblock -> GoStartLabel -> GoBlock -> GoUnblock -> GoStartLabel
             4285073 runtime.gcBgMarkWorker /usr/local/go/src/runtime/mgc.go 1435

    319480ns -> 0.000319s

        GoStart Ts: 940403207 P: 0 G: 10 StkID: 0 args0: 10 args1: 0 args2: 0
             GoSleep -> GoUnblock
             4452857 time.Sleep /usr/local/go/src/runtime/time.go 59
             4746323 main.godeadloop1.func1 /root/go/src/diaggreedy/x.go 112
             4744437 main.godeadloop1 /root/go/src/diaggreedy/x.go 114
        GoSched Ts: 940722687 P: 0 G: 10 StkID: 23 args0: 0 args1: 0 args2: 0
             4746333 main.godeadloop1.func1 /root/go/src/diaggreedy/x.go 105
             4744437 main.godeadloop1 /root/go/src/diaggreedy/x.go 114
```

As you see, the typical GC pause finally changed from 8ms to 0.4ms. Of course 
the situation would be much more complex in a real big project, but the key idea 
of the tracking is the same and clear.

Also implemented a 'dump' option like this below:
```bash
$ # Ts is the event's timestamp in nanoseconds
$ # by reading the go/src/internal/trace/parser.go to get more info

$ go tool trace -dump /root/go/src/diaggreedy/trace_my_2017-06-25-10-03-15.out 

        0xc42001c120 Off: 24 type: 13 GoCreate Ts: 0 P: 1 G: 0 StkID: 1 args0: 1 args1: 2 args2: 0 link: 0xc42001d050
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c1b0 Off: 30 type: 13 GoCreate Ts: 3879 P: 1 G: 0 StkID: 1 args0: 2 args1: 3 args2: 0 link: 0x0
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c240 Off: 36 type: 31 GoWaiting Ts: 4119 P: 1 G: 2 StkID: 0 args0: 2 args1: 0 args2: 0 link: 0x0
        0xc42001c2d0 Off: 39 type: 13 GoCreate Ts: 8079 P: 1 G: 0 StkID: 1 args0: 3 args1: 4 args2: 0 link: 0x0
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c360 Off: 45 type: 31 GoWaiting Ts: 8199 P: 1 G: 3 StkID: 0 args0: 3 args1: 0 args2: 0 link: 0xc427e29e60
        0xc42001c3f0 Off: 48 type: 13 GoCreate Ts: 13639 P: 1 G: 0 StkID: 1 args0: 4 args1: 5 args2: 0 link: 0x0
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c480 Off: 55 type: 31 GoWaiting Ts: 13759 P: 1 G: 4 StkID: 0 args0: 4 args1: 0 args2: 0 link: 0x0
        0xc42001c510 Off: 58 type: 13 GoCreate Ts: 14159 P: 1 G: 0 StkID: 1 args0: 5 args1: 6 args2: 0 link: 0xc42001cb40
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c5a0 Off: 64 type: 13 GoCreate Ts: 14399 P: 1 G: 0 StkID: 1 args0: 6 args1: 7 args2: 0 link: 0x0
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c630 Off: 70 type: 31 GoWaiting Ts: 14439 P: 1 G: 6 StkID: 0 args0: 6 args1: 0 args2: 0 link: 0xc4223ee2d0
        0xc42001c6c0 Off: 73 type: 13 GoCreate Ts: 18039 P: 1 G: 0 StkID: 1 args0: 7 args1: 8 args2: 0 link: 0x0
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c750 Off: 79 type: 31 GoWaiting Ts: 18119 P: 1 G: 7 StkID: 0 args0: 7 args1: 0 args2: 0 link: 0xc44016d710
        0xc42001c7e0 Off: 82 type: 13 GoCreate Ts: 21479 P: 1 G: 0 StkID: 1 args0: 8 args1: 9 args2: 0 link: 0xc42001d290
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c870 Off: 88 type: 13 GoCreate Ts: 25199 P: 1 G: 0 StkID: 1 args0: 9 args1: 10 args2: 0 link: 0xc42592f440
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c900 Off: 94 type: 13 GoCreate Ts: 29159 P: 1 G: 0 StkID: 1 args0: 10 args1: 11 args2: 0 link: 0xc42001d170
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001c990 Off: 100 type: 13 GoCreate Ts: 33279 P: 1 G: 0 StkID: 1 args0: 17 args1: 12 args2: 0 link: 0x0
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001ca20 Off: 106 type: 32 GoInSyscall Ts: 33359 P: 1 G: 17 StkID: 0 args0: 17 args1: 0 args2: 0 link: 0x0
        0xc42001cab0 Off: 109 type: 5 ProcStart Ts: 33439 P: 1 G: 0 StkID: 0 args0: 3 args1: 0 args2: 0 link: 0x0
        0xc42001cb40 Off: 112 type: 14 GoStart Ts: 33719 P: 1 G: 5 StkID: 6 args0: 5 args1: 0 args2: 0 link: 0xc42001ce10
                 4744545 main.main.func1 /root/go/src/diaggreedy/x.go 22
        0xc42001cbd0 Off: 115 type: 33 HeapAlloc Ts: 36559 P: 1 G: 5 StkID: 0 args0: 253952 args1: 0 args2: 0 link: 0x0
        0xc42001cc60 Off: 120 type: 33 HeapAlloc Ts: 36879 P: 1 G: 5 StkID: 0 args0: 262144 args1: 0 args2: 0 link: 0x0
        0xc42001ccf0 Off: 172 type: 4 Gomaxprocs Ts: 42519 P: 1 G: 5 StkID: 13 args0: 2 args1: 0 args2: 0 link: 0x0
                 4366993 runtime.startTheWorld /usr/local/go/src/runtime/proc.go 951
                 4456480 runtime.StartTrace /usr/local/go/src/runtime/trace.go 256
                 4743634 runtime/trace.Start /usr/local/go/src/runtime/trace/trace.go 23
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001cd80 Off: 177 type: 13 GoCreate Ts: 64239 P: 1 G: 5 StkID: 15 args0: 18 args1: 14 args2: 0 link: 0xc42001cea0
                 4743712 runtime/trace.Start /usr/local/go/src/runtime/trace/trace.go 34
                 4744891 main.main.func1 /root/go/src/diaggreedy/x.go 30
        0xc42001ce10 Off: 184 type: 19 GoSleep Ts: 69599 P: 1 G: 5 StkID: 16 args0: 0 args1: 0 args2: 0 link: 0xc44016e2d0
                 4452857 time.Sleep /usr/local/go/src/runtime/time.go 59
                 4744919 main.main.func1 /root/go/src/diaggreedy/x.go 34
        0xc42001cea0 Off: 188 type: 14 GoStart Ts: 72679 P: 1 G: 18 StkID: 14 args0: 18 args1: 0 args2: 0 link: 0xc42001cfc0
                 4743809 runtime/trace.Start.func1 /usr/local/go/src/runtime/trace/trace.go 26
        0xc42001cf30 Off: 191 type: 28 GoSysCall Ts: 73959 P: 1 G: 18 StkID: 17 args0: 0 args1: 0 args2: 0 link: 0x0
                 4552069 syscall.write /usr/local/go/src/syscall/zsyscall_linux_amd64.go 1064
                 4550393 syscall.Write /usr/local/go/src/syscall/syscall_unix.go 181
                 4591004 os.(*File).write /usr/local/go/src/os/file_unix.go 186
                 4587980 os.(*File).Write /usr/local/go/src/os/file.go 142

        # ......
```

## Reference

https://groups.google.com/forum/#!topic/golang-nuts/8KYER1ALelg

https://github.com/golang/go/issues/10958
