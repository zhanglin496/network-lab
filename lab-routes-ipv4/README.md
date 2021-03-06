# Experiments with Linux IPv4 FIB trie

See
[IPv4 route lookup on Linux](https://vincent.bernat.im/en/blog/2017-ipv4-route-lookup-linux) for
details.

After inserting routes, see:

 - `/proc/net/fib_trie`
 - `/proc/net/fib_triestat`

To get a MRT dump:

    wget http://data.ris.ripe.net/rrc00/latest-bview.gz

## Microbenchmark

The microbenchmark uses a small kernel module to run a benchmark. It
works only on x86-64. To compile it, use the following command:

    make kbench_mod.ko

Load it with `insmod`. You'll get a `/sys/kernel/kbench`
directory. There are various parameters tweakable. To run the
benchmark, tun `cat /sys/kernel/kbench/run`:

    min=76 max=760 average=95 50th=100 90th=112 95th=112

The results are the number of cycles. To make sense of the results,
let's assume a 2 GHz clock (use `cpupower frequency-info` to get the
information). This means a lookup takes 50 ns. This happens to be
about the time a 10 Gbps interface takes to send a 64 bytes packet. Of
course, the kernel doesn't do only a route lookup. Also, route lookup
is done using RCU and therefore scales well with the number of cores.

The benchmark is highly sensitive to timing. Be sure to disable any
power saving settings:

    cpupower frequency-set -g performance

## Results

When looking at a dump of the tree (from `/proc/net/fib_trie`), there
are some numbers for a non-leaf node:

      +-- 0.0.0.0/0 15 7193 20306

The first one is the number of bits (`nbits`) handled by the non-leaf
node, the second is the number of full children and the last one is
the number of empty children. Those numbers are kept up-to-date to
trigger resizing of the node. A node is doubled if the ratio of
non-empty children to all children (in the resulting node) is at least
50% (30% for the root). It is halved if the ratio (in the current
node) is at most 25% (15% for the root). In the example above, the
ratio in the current node is 38%, so it shouldn't be halved (more than
15%) and it should be doubled (less than 60%).

Here are results from `/proc/net/fib_triestat`. Note that `main` and
`local` tables are merged, so they have the same stats. We only show
`main`.

Some explanations:

 - A "tnode" is a non-leaf node.
 - "Null pointers" are empty children (if there are too many, the
   appropriate node need to be halved).
 - The statistics below "Internal nodes" are the number of internal
   nodes for a given depth.

Here are some examples:

### 100k /32

    $ cat /proc/net/fib_triestat
    Basic info: size of leaf: 48 bytes, size of tnode: 40 bytes.
    Main:
            Aver depth:     3.67
            Max depth:      5
            Leaves:         100009
            Prefixes:       100016
            Internal nodes: 30096
              1: 4207  2: 25046  3: 839  15: 4
            Pointers: 246382
    Null ptrs: 116278
    Total size: 13259  kB

### 100k partial view

    Basic info: size of leaf: 48 bytes, size of tnode: 40 bytes.
    Main:
            Aver depth:     2.81
            Max depth:      6
            Leaves:         97194
            Prefixes:       100016
            Internal nodes: 43717
              1: 22  2: 36589  3: 4507  4: 1726  5: 754  6: 116  7: 2  15: 1
            Pointers: 274648
    Null ptrs: 133738
    Total size: 13879  kB

We can also check memory usage in `/proc/slabinfo` (merging needs to
be disabled).

    ip_fib_trie        97202  97276     48   83    1 : tunables  120   60    0 : slabdata   1172   1172      0
    ip_fib_alias      100024 100039     56   71    1 : tunables  120   60    0 : slabdata   1409   1409      0

We notice that an object for `ip_fib_trie` is 48 bytes (which matches
the info in `/proc/net/fib_triestat`) and we have 97k of them, which
accounts for about 5MB for leaves. The non-leaves are allocated with
either `kzalloc()` or `vzalloc()` since they are expected to be
big. FIB aliases are equal to the number of prefixes and also
allocated on the slab. They are also already accounted for in the
`Total size` information.

An alias is a prefix, ToS and scope. It references a `struct fib_info`
that contains the remaining information. Those structs should be
shared among many aliases. They are also allocated directly (with
`kzalloc()`). The kernel keep count on how many of them they are but
the information is not exported. The size of this structure depends on
the number of hop (in case of a multihop route) and whenever metrics
are stored for the given route (you can check if a route has metrics
associated to it with `ip route get`). A hash table is also kept to be
able to find existing structs and reuse them.

## Using histograms for microbenchmark

This needs a 4.19+ kernel with `CONFIG_HIST_TRIGGERS=y`. We use the
existing tracepoint `fib_table_lookup`. We also need to create an
additional kprobe with the return of the function. Moreover, we need a
synthetic event to store the duration.

    cd /sys/kernel/debug/tracing
    echo 'r:fib_table_lookup_ret fib_table_lookup $retval' >> kprobe_events
    echo 'fn_duration unsigned int cpu; u64 duration' >> synthetic_events
    echo 'hist:keys=cpu:ts0=common_timestamp' > events/fib/fib_table_lookup/trigger
    echo 'hist:keys=cpu:lat=common_timestamp-$ts0:onmatch(fib.fib_table_lookup).fn_duration(cpu,$lat)' > events/kprobes/fib_table_lookup_ret/trigger
    echo 'hist:keys=cpu,duration.log2:sort=cpu,duration' > events/synthetic/fn_duration/trigger

Look at the histogram with:

    $ cat events/synthetic/fn_duration/hist
    # event histogram
    #
    # trigger info: hist:keys=cpu,duration.log2:vals=hitcount:sort=cpu,duration.log2:size=2048 [active]
    #
    
    { cpu:          0, duration: ~ 2^9  } hitcount:          2
    { cpu:          0, duration: ~ 2^11 } hitcount:          1
    { cpu:          0, duration: ~ 2^12 } hitcount:          1
    { cpu:          0, duration: ~ 2^13 } hitcount:         19
    { cpu:          0, duration: ~ 2^14 } hitcount:          6
    
    Totals:
        Hits: 29
        Entries: 5
        Dropped: 0

The numbers don't match the ones we get with the microbenchmark
module. I think there are two reasons (to be investigated):

 - the overhead of using the histogram infrastructure,
 - the cache not being as efficient since more code is executed and
   more data is fetched.

The latency with eBPF is the same:

    $ funclatency-bpfcc fib_table_lookup
    Tracing 1 functions for "fib_table_lookup"... Hit Ctrl-C to end.
    
         nsecs               : count     distribution
             0 -> 1          : 0        |                                        |
             2 -> 3          : 0        |                                        |
             4 -> 7          : 0        |                                        |
             8 -> 15         : 0        |                                        |
            16 -> 31         : 0        |                                        |
            32 -> 63         : 0        |                                        |
            64 -> 127        : 0        |                                        |
           128 -> 255        : 0        |                                        |
           256 -> 511        : 0        |                                        |
           512 -> 1023       : 0        |                                        |
          1024 -> 2047       : 0        |                                        |
          2048 -> 4095       : 0        |                                        |
          4096 -> 8191       : 5        |****************************************|
          8192 -> 16383      : 3        |************************                |

# References

 - https://www.kernel.org/doc/Documentation/networking/fib_trie.txt
 - http://www.drdobbs.com/cpp/fast-ip-routing-with-lc-tries/184410638
 - https://people.netfilter.org/pablo/netdev0.1/papers/Picking-the-low-hanging-fruit-from-the-FIB-tree.pdf
 - http://www.netdevconf.org/1.1/proceedings/slides/kubecek-ipv6-route-lookup-performance-scaling.pdf
 - https://git.kernel.org/pub/scm/linux/kernel/git/davem/net_test_tools.git/tree/
 - https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=5e9965c15ba88319500284e590733f4a4629a288
 - https://www.nada.kth.se/~snilsson/publications/IP-address-lookup-using-LC-tries/
 - http://www.csc.kth.se/~snilsson/software/dyntrie2/

Not read yet:

 - https://arxiv.org/pdf/1402.1194.pdf

