# Testing compression algorithms for cache

## Compressing local Git repository of a project, ~457.8 MiB

### Attempt no. 1: compression level 6, XZ

Command:

`XZ_OPT="-6" time tar -c -v -J -f test6.tar.xz project/`

Result:

```
234.34user 2.42system 3:54.47elapsed 100%CPU (0avgtext+0avgdata 97808maxresident)k
464inputs+241136outputs (4major+24578minor)pagefaults 0swaps
```

Notes:

- Although I have chosen 4 threads for the compression, and later I've
  tried with 2 threads, none were used, `xz` was running single-threaded
- Documentation shows that multithreading is NOT implemented yet
- Output file size: 117.7 MiB
- Total run time: ~3 minutes 26 seconds
- Output size: 25% of the original, 75% savings

 
### Attempt no. 2: compression level 4, XZ

Command:

`XZ_OPT="-4" time tar -c -v -J -f test4.tar.xz project/`

Result:

```
135.43user 1.92system 2:14.96elapsed 101%CPU (0avgtext+0avgdata 50784maxresident)k
0inputs+258560outputs (0major+24040minor)pagefaults 0swaps
```

Notes:

- Output file size: 126.2 MiB
- Total run time: ~2 minutes 15 seconds
- Output size: 27.5% of the original, 72.5% savings


### Attempt no. 3: compression level 9, GZIP

Command:

`GZIP=-9 time tar -c -v -z -f test9.tar.gz project/`

Result:

```
109.00user 1.09system 1:49.06elapsed 100%CPU (0avgtext+0avgdata 3588maxresident)k
288inputs+372688outputs (2major+22770minor)pagefaults 0swaps
```

Notes:

- Output file size: 182 MiB
- Total run time: ~1 minute 49 seconds
- Output size: 39.7% of the original, 60.3% savings


### Attempt no. 4: compression level 6, GZIP

Command:

`GZIP=-6 time tar -c -v -z -f test6.tar.gz project/`

Result:

```
21.78user 1.08system 0:22.50elapsed 101%CPU (0avgtext+0avgdata 3644maxresident)k
0inputs+375456outputs (0major+22771minor)pagefaults 0swaps
```

- Output file size: 183.3 MiB
- Total run time: ~23 seconds
- Output size: 40% of the original, 60% savings


### Attempt no. 5: compression level 4, GZIP

Command:

`GZIP=-4 time tar -c -v -z -f test4.tar.gz project/`

Result:

```
13.28user 0.98system 0:14.73elapsed 96%CPU (0avgtext+0avgdata 3608maxresident)k
0inputs+385696outputs (0major+22770minor)pagefaults 0swaps
```

- Output file size: 188.3 MiB
- Total run time: ~15 seconds
- Output size: 41.1% of the original, 58.9% savings


### Attempt no. 6: compression level 3, GZIP

Command:

`GZIP=-3 time tar -c -v -z -f test3.tar.gz project/`

Result:

```
13.02user 1.02system 0:14.62elapsed 96%CPU (0avgtext+0avgdata 3420maxresident)k
0inputs+395760outputs (0major+22773minor)pagefaults 0swaps
```

- Output file size: 193.2 MiB
- Total run time: ~15 seconds
- Output size: 42.2% of the original, 57.8% savings


### Attempt no. 7: compression level 4, PIGZ (parallel gzip)

Command:

`time tar cf - project/ | pigz -4 -p 4 > test4p.tar.gz`

Result:

```
real  0m5.990s
user  0m21.148s
sys   0m0.916s
```

- Output file size: 187.7 MiB
- Total run time: ~6 seconds
- Interestingly, it saved ~0.5 MiB and ran twice as fast as GZIP with
  compression level 4!
- Output size: 41% of the original, 59% savings


### Attempt no. 8: compression level 6, PIGZ (parallel gzip)

Command:

`time tar cf - project/ | pigz -6 -p 4 > test6p.tar.gz`

Result:

```
real  0m8.939s
user  0m33.400s
sys   0m0.924s

```

- Output file size: 183 MiB
- Total run time: ~9 seconds
- Even better, it saved ~0.3 MiB and ran almost three times faster than
  GZIP with compression level 6!
- Output size: 39.9% of the original, 60.1% savings


### Attempt no. 9: compression level 9, PIGZ (parallel gzip)

Command:

`time tar cf - project/ | pigz -9 -p 4 > test9p.tar.gz`

Result:

```
real  0m39.518s
user  2m26.472s
sys   0m0.928s

```

- Output file size: 181,9 MiB
- Total run time: ~40 seconds
- It saved ~0.1 MiB and ran almost 2,72 times faster than GZIP with
  compression level 9!
- Output size: 39.7% of the original, 60.3% savings


## Conclusions

- XZ has good compression ratio, but it is single-threaded and takes too
  much time - we need to get the cache out as soon as possible and
  destroy the container, so XZ is not suitable at all
- GZIP is much faster, and the difference between level 3 and 4 is
  negligient, they lasted the same time and level 4 brought about 5 MiB
  of savings, so we should go with level 4
- Unfortunately, GZIP is also single-threaded
- PIGZ is a parallel implementation of GZIP, and is *much faster* than
  the other algorithms - definitely better suited
- PIGZ with compression level 6 or 4 sounds like a good choice

PIGZ installation: `sudo apt-get install pigz`
PIGZ manual: `http://www.zlib.net/pigz/pigz.pdf`
