http://graphics.stanford.edu/~seander/bithacks.html
```objc
//Swap two integers:
x = x ^ y;
y = x ^ y;
x = x ^ y;

//Find the minimum r of two integers x and y:
r = y ^ ((x^y)& -(x<y));

//Compute (x+y) mod n:   0 <= x < n,  0 <= y < n
r = (x+y)%n

z = x+y;
r = (z<n)?z:n-n;

z = x+y;
r = z - (n & -(z >=n));


//Note:  mis-prediction on branch costs 16 cycles to empty the pipeline
//register : 1 cycle (6 ops issued per cycle per core)

//(per 64-byte cache line):
//L1 cache: 4 cycles
//L2 cache: 10 cycles
//L3 cache: 50cycles
//DRAM: 150 cycles


//Round up to a power of 2:   Compute 2^(ceil(log n))
—n;
n |= n >> 1;
n |= n >> 2;
n |= n >> 4;
n |= n >> 8;
n |= n >> 16;
n |= n >> 32;
++n

//Compute the mask of the least significant bit:
r = x & (-x);

//Compute lg(x), where x is a power of 2:
//debruijn sequence

//Count the number of 1 in a word x:
for (r = 0; x != 0; ++r) {
    x &= x-1;
}

//Table lookup:
static const int count[256] = {0, 1, 1, 2, 1, 2, 2, 3, 1, …, 8};
for (r = 0; x !=0; x>>=8) {
    r += count[x & 0xff];
}
```
