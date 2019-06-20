# QuickXorHash
microsoft's QuickXorHash algorithm in swift (optimized)



## explaining it all

how does the hash that microsoft explains in pseudo code works and why it's super slow :


### microsoft pseudo code:
```
method block zero():
    returns a block with all zero bits.


method block reverse(block b)
    returns a block with all of the bytes reversed (00010203... => ...03020100)


method block extend8(byte b):
    returns a block with all zero bits except for the lower 8 bits which come from b.


method block extend64(int64 i):
    returns a block of all zero bits except for the lower 64 bits which come from i
    and are in little-endian byte order.


method block rotate(block bl, int n):
    returns bl rotated left by n bits.


method block xor(block bl1, block bl2):
    returns a bitwise xor of bl1 with bl2


method block XorHash0(byte[] rgb):
    block ret = zero()
    for (int i = 0; i < rgb.Length; i ++)
    ret = xor(ret, rotate(extend8(rgb[i]), i * 11))
    returns reverse(ret)


entrypoint block XorHash(byte[] rgb):
    returns xor(extend64(rgb.Length), XorHash0(rgb))


```

### optimizations:

the main bottleneck of the hash method is reading a file byte-by-byte and doing the rotated on
extend8 block like so:


```
for (int i = 0; i < rgb.Length; i ++)
    ret = xor(ret, rotate(extend8(rgb[i]), i * 11))
returns reverse(ret)
```


if we run the this we will take each byte of the file and extend8 it (i.e. put it in a block of 20 bytes)
rotate that block i*11 bits and then xor the result


if we take a look at this loop we cloud notice a pattern that look like this: (without the xor action)


```
*'bi'  is the i-th byte of data
here numbers represents bits




0    8                    160
| b0 |_______________...___|




0 11    19               160
|__| b1 |___________...___|




0       22   30          160
|_______| b2 |______...___|




0            33   41     160
|____________| b3 |_...___|


.
.
.


0      8                    160
| b160 |_______________...___|


0 11     19               160
|__| b161 |___________...___|


.
.
.
```




now we notice that when i (from above) goes beyond 159 we get the same positions of bytes in the the block
this happens since the rotate function will do just that: rotate the bits left, and since we only have 160 bits
in a block each iteration of 160 bit we loop over the positions in the block


so a first way to optimize this hash method is to throw away the extend8 method and just xor every 0 to 159th bytes
of the file with the next 160th to 319th bytes of the file and so on, i.e:


```
| b0   | b1   | b2   | ... | b159 |
xor
| b160 | b161 | b162 | ... | b319 |
.
.
xor
| b800 | b801 | b802 | ... | b759 |
.
.
.


```
and we'll get the same result as we got by doing extend8() and xor


BUT we're not done yet!


now we have all the bytes of the file xor-ed into 160 bytes which is much better than reading the whole file byte-by-byte
and doing the extend8 call on it, but we still need to xor all those 160 bytes into a 20 byte block with all the bytes in
their correct rotated position

so we want to order the bytes in blocks that will allow us to xor all those blocks to get the final result

if we look closer on the positions each byte is found after the rotate and extend8 calls, we'll notice that there are
8 possible bit offsets that bytes are able to be in (since we align data in bytes) so we'll use an array of
8 blocks in each n'th block will store all the bytes that have the offset of n bits, in each of these blocks we'll position
the byte in it's byte aligned placement

i.e.
a byte that should be rotated 11 bits originally will be placed at the 3rd (11 % 8) block at position 1 (11/8)
a byte that should be rotated 33 bits originally will be placed at the 1st (33 % 8) block at position 4 (33/8)
a byte that should be rotated 154 bits originally will be placed at the 2nd (154 % 8) block at position 19 (154/8)

doing this will give use all the bytes in their correct byte aligned order but we want the exact correct order, so to achieve
that we'll just rotate all the bytes in the n'th block n bits to the left, and now we have all the bits in the correct placement
inside a block

and since the xor operations is associative and commutative the order in which we xor the bytes is meaningless

so we can now xor all the 8 blocks we have into a single block and get the exact same result we would have gotten in the
original hash method just much lot faster

### code specific optimizations
in the swift code i added some optimizations like in-place xor operations and reading the file 8 bytes at a time instead of reading it one byte in a time, more explanations are in the code itself

## benchmarks

from a few benchmarks i did this code will get ~600milisec per ~100MB which was good enough for my usage

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

