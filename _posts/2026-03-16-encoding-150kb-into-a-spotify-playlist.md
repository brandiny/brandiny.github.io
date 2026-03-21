---
layout: post
title: "Spotify Shenanigans: Encoding Romeo & Juliet into a playlist"
date: 2026-03-16
---

## Encoding 150KB into a Spotify Playlist

This all began with a thought I had at work. I'd add, reorganize, and create
structure in my folders of music of Spotify. Each track inherently contains information, not to mention all the additional metadata (duration, genre, artist),
which a song contains. This is reasonably obvious to anyone who uses Spotify.


Then, began my rabbit hole: how much information can
you store into a Spotify playlist?

---
### Initial Implementation (1.25KB) - Bitstring
Each playlist can have a maximum of 10,000 songs. Consider a playlist $P$ as a sequence of songs drawn from a song repository $S$. 

$$
P = \{s_1, s_2, \ldots, s_n\}
$$

In the simplest case, we
can consider $P$ as a bit string, where each song represents either a one or a zero. We would need exactly two songs here! This clearly isn't a very good idea,
but its enough to see if the idea works (+ encode hello world!).

$$
P = \{0, 1, 0, 0, 1, \ldots\}
$$

In the spirit of being basic, we will use two basic songs here (Yellow - Coldplay, Someone Like You - Adele), as our one and zero respectively. Lo and behold, it works like a charm:

```
@brandon ➜ 2026  echo "hello spotify" | python3 01.py encode | xargs python3 01.py decode
Input:    14 bytes → 112 bits
Hash:     c0614f6e59502829d6d14f6fff9b1fbbd72718632bb115156574f39a94846074
Looking up bit tracks...
  bit 0 → Yellow by Coldplay (spotify:track:3AJwUDP919kvQ9QcozQPxg)
  bit 1 → Someone Like You by Adele (spotify:track:3bNv3VuUOKgrf5hu3YcuRo)
Creating playlist...
Adding 112 tracks in batches of 100...
  100/112
  112/112

Done. Playlist: c0614f6e59502829d6d14f6fff9b1fbbd72718632bb115156574f39a94846074
  bit 0 → Yellow by Coldplay (spotify:track:3AJwUDP919kvQ9QcozQPxg)
  bit 1 → Someone Like You by Adele (spotify:track:3bNv3VuUOKgrf5hu3YcuRo)
Looking for playlist 'c0614f6e59502829d6d14f6fff9b1fbbd72718632bb115156574f39a94846074'...
Fetching tracks...
Hash verified OK.
hello spotify
```
There are a few takeways from this initial implementation:

- Use of a hash function. This is a nice way to key into a playlist given your input, and also act as a checksum for whether we encoded the information correctly. In this case, we use a simple SHA-1 hash.


- Read / write limits. Unsuprisingly, Spotify throttles how fast you can interact with a playlist, but this was more dire than expected. We can read / write from an offset within the playlist (almost think of it as a byte buffer) with a maximum length of 100.

- 10,000 bits at 8 bits / byte equates to a maximum of 1.25KB

---
### Improvement I - Permutative Encoding (14.46KB)
Not bad, but we can do a lot better. Instead of treating each track as an binary digit, how about we use the entire playist: treat the sequence (ordering) as information. It's like when you listen to a good playlist, the ordering of music tells a story too!

Let's assume the simplest case again, we have $n = 10000$ unique songs to put into our playlist. We can encode information in each unique ordering of the playlist -- a permutation. The total number of permutations here is $n!$. For our first song, we can choose from all the songs. For our second song, we can choose from all the songs but the one we just chose, and so on.

$$
n! = n\cdot(n-1)\cdot(n-2) \cdots 3\cdot2\cdot1
$$

The total quantity of information (maximum number of bitstrings) we can encode here is:

$$
\log_2(10000!) = \log_2(10000) + \log_2(9999) + \cdots + \log_2(1) \approx 14.46\text{KB}
$$

Now, this is cool and all, but: how do we actually implement this? We need a way to take data (a large number) and map it into one of the $n!$ permutations in this space. It needs to be a 1:1 mapping that is also invertible (so we can decode). This is a bijection!

#### Lehmer Codes: Theory
A simple way of generating a permutation is by sampling without replacement. Let say we have an array $A$ of $n$ songs. We first choose a random index $i$ uniformly from $\[0, n - 1\]$. Then, we add $A[i]$ to our permutation, and delete the item at $i$ from $A$. Repeat this until you have no items left. Then, you have a sequence of numbers which translates to a permutation! Let's do an example: suppose we have the following songs:

```
Alive - Empire of the Sun
Heather - Conan Gray
Breakeven - Script
Intro - The xx
```

An array which would correspond to a valid permutation would be $[3, 1, 1, 0]$. The first entry is between $\[0, 3\]$, the second between $\[0, 2\]$, and so on. This array translates to [Intro, Heather, Breakeven, Alive]. 

This is the cool part: we can treat this as a **number system**. A Lehmer code $[d_{n-1}, d_{n-2}, \ldots, d_0]$ is like a mixed-radix number, where digit $d_k$ has a maximum value of $k$. If we choose the multipler at each digit position as the maximum value but as a factorial (i.e. for the digit with a maximum value of 4, it gets multipled by 4!) - the value it represents is:

$$
d_{n-1} \cdot (n-1)! + d_{n-2} \cdot (n-2)! + \cdots + d_1 \cdot 1! + d_0 \cdot 0!
$$

The maximum value is when every digit is at its max:

$$
(n-1)\cdot(n-1)! + (n-2)\cdot(n-2)! + \cdots + 1\cdot1! = \sum_{k=1}^{n-1} k \cdot k!
$$

Using the identity $k \cdot k! = (k+1)! - k!$, this telescopes to $(n)! - 1$. So the system spans exactly $0$ to $n! - 1$ — every permutation, no gaps. This is exactly what we want - as it maps every permutation from this range into an array we can readily create.


#### Lehmer Codes: Implementation
This is nice, but how would you actually implement this? What we essentially want to achieve is a way to readily go from `big number (our data) <--> lehmer code (playlist)`. This is surprisingly nice. Let's think about how we would translate a number from base 10 to base 2 usually. We can use 42 as an example. To extract the least significant digit (radix 2), we would do $42 \space \text{mod} \space  2 = 0$, then divide by 2 to go to the next significant digit (also radix 2). I.e. $21 \space \text{mod} \space 2 = 1$. Repeat until you get to zero (performing integer division), $10 \space \text{mod} \space 2 = 0$, $5 \space \text{mod} \space 2 = 1$, $2 \space \text{mod} \space 2 = 0$, $1 \space \text{mod} \space 2 = 1$. Then arrange the digits in reverse order (most significant first):
```
42 (base 10) --> 101010
```

We can do the same for our factoradic number system.
```python
def int_to_lehmer(data: int, n: int) -> list[int]:
    digits = [0] * n
    current = data
    for i in range(n):
        radix = i + 1
        digits[n - i - 1] = current % radix
        current //= radix
    return digits
```

To go in the opposite direction (Lehmer -> BigNumber). We simply sum the lehmer code by each position's digit multiplied by its radix:

```python
def lehmer_to_int(digits: list[int]) -> int:
    n = len(digits)
    return sum(d * math.factorial(n - 1 - i) for i, d in enumerate(digits))
```

#### Putting it all Together

With both functions in hand, the full pipeline is straightforward. To encode: treat the raw bytes of your data as a big integer $N$, call `int_to_lehmer(N, n)` to get a sequence of indices into your song pool, then reorder the pool by those indices — that's your playlist. To decode: read the playlist back, reconstruct the Lehmer code from each track's original position in the pool, then call `lehmer_to_int`.

For "hello spotify" (14 bytes, 112 bits), we need a pool of $n$ songs where $n! \geq 2^{112}$. Since $\log_2(31!) \approx 112.7$, 31 songs is enough — down from 112 tracks:

```
@brandon ➜ 2026  echo "hello spotify" | python3 permute.py encode | xargs python3 permute.py decode
Input:    14 bytes → 112 bits
Hash:     c0614f6e59502829d6d14f6fff9b1fbbd72718632bb115156574f39a94846074
Looking up song pool (31 tracks)...
Creating playlist...
Adding 31 tracks...
  31/31

Done. Playlist: c0614f6e59502829d6d14f6fff9b1fbbd72718632bb115156574f39a94846074
Looking for playlist 'c0614f6e59502829d6d14f6fff9b1fbbd72718632bb115156574f39a94846074'...
Fetching tracks...
Hash verified OK.
hello spotify
```

### Improvement II - Adding many more songs (30.79KB)
Immediately, we can see an easy way to get more information into the playlist: choose a permutation from a larger catalog of songs. Spotify definitely has more than 10,000 unique songs (believe me). A quick peruse on the web yielded a [dataset on HuggingFace from GildasLeDrogoff](https://huggingface.co/datasets/GildasLeDrogoff/spotify-huge-track-analysis-dataset/discussions), which has 39M+ songs! However, there is a fatal algorithmic problem in our encoding / decoding pipeline at the moment. Going from `data <--> lehmer` as we can see above is linear O(N). The problem is going from `lehmer <--> playlist`.

- When we encode our lehmer code (the array) [3, 0, 1, 1] etc, we repeatedly search and remove from a sequence to construct our playlist. With an array the search would be O(1) but the removal would be O(N). With a linkedlist, the search would be O(N), but the removal O(1). Repeated N times, the complexity is effectively quadratic, O(N^2), which not feasible when N = 39M. 


- Similarly, when we decode from our playlist, we are effectively finding each song's rank among the remaining songs. For each song in the playlist, the previously decoded songs have already been removed from the catalog, shifting the positions of later songs. Finding that shifted rank requires a linear scan with a list — O(N) per song, O(N²) overall.

Let's whip out a Fenwick Tree! What we are going to do is compute prefix sums to answer our problems in encoding/decoding. Our tree will be of size $n$ (number of songs), with a 1 in each position if the song is still unselected, or 0 if it has been used.

- The easier case to visualize is decoding (computing rank). We want the rank of a song among the remaining songs — i.e. the prefix sum up to its catalog index. As used songs are invisible (0), this is a standard BIT prefix sum query: O(log n), with a final complexity of O(n log n).

- Encoding is also O(n log n). We want to find the k-th remaining song (given that the tree has holes from prior removals). Since the prefix sums are monotonically non-decreasing, we can walk down the tree directly to land on the k-th element in O(log n) — no need for a separate binary search. I.e. Then setting that song to 0 updates O(log n) nodes. Beautiful!

```
@brandon ➜ 2026  echo "hello spotify" | python3 steg.py encode | xargs python3 steg.py decode
Loaded 39283820 cached tracks
Data:     14 bytes (112 bits)
  Image:  0 bytes  → QR code cover
  Tracks: 14 bytes → 5 tracks (126 bits capacity, 25.20 bits/track)
Hash:     c0614f6e59502829d6d14f6fff9b1fbbd72718632bb115156574f39a94846074
Adding 5 tracks in batches of 100...
  5/5
Image:    0 bytes in COM markers (3 KB JPEG, 5 KB b64)

Done. Playlist: c0614f6e59502829d6d14f6fff9b1fbbd72718632bb115156574f39a94846074
Loaded 39283820 cached tracks
  Fetched 5/5 tracks
Hash verified OK.
hello spotify
```

With a larger catalog of $N \approx 39,000,000$ songs and a playlist of $K = 10,000$ tracks, we are now choosing an ordered selection of $K$ songs from $N$. The total possible information here is:

$$
\log_2\left(\frac{N!}{(N-K)!}\right) = \log_2(N) + \log_2(N-1) + \cdots + \log_2(N-K+1) \approx 30.79\text{KB}
$$



### Improvement III - Cover Image Steganography (42.7KB)
At this point, you are probably wondering how on earth you are going to encode more information into this 10,000 size playlist which is already bursting at its seams. The answer is: I don't. This part is a little bit hacky, but it works, so it doesn't matter. We can use the cover image of the playlist.

Originally, the plan was to stuff the JPEG image you can upload with comments (COM markers) which begin with the bytes `0xFF 0xFE`, which sit in the file header. This would've been nuts and we would be able to encode a ton of information. But, alas (unsurprisingly), these are stripped and we cannot use this channel.

So instead, we encode data directly into the pixels. The image is divided into a grid of $8 \times 8$ pixel cells, where each cell is either solid black (bit 0) or solid white (bit 1):

$$
\text{cell}_{r,c} = \begin{cases} \text{black} & \text{bit} = 0 \\ \text{white} & \text{bit} = 1 \end{cases}
$$

The key insight is that $8 \times 8$ pixels is exactly one JPEG DCT (Discrete Cosine Transform) block. Essentially, the compression occurs at this granularity, so by encoding our information at the block level, instead of pixels, we can get lossless encoding with a JPG image (albeit our image just becomes noise).

The layout uses a 2400×2400 image, giving a 300×300 cell grid. Two header rows are reserved:

- **Row 0**: alternating black/white cells — sentinel pattern to indicate an encoded image
- **Row 1** (cells 0–15): 16-bit payload byte count
- **Rows 2–299**: data bits, left-to-right, top-to-bottom

So the final encoding splits data across two channels:

- **Track order**: first ~31.5KB of data encoded as a Lehmer permutation of the playlist
- **Cover image**: remaining data encoded as a pixel-block grid in the playlist cover

Total capacity: $31.5 + 11.2 \approx 42.7\text{KB}$ per playlist.

---
### Demo - Romeo & Juliet (149KB)

As a grand finale, we will attempt to encode Romeo & Juliet into a playlist!. R&J is about 149KB of raw text which is pretty cooked - so we are going to need to compress with a hydraulic press. I looped through compressors (zstd, lzma, 7z, brotli). Eventually, we got it below the 42.7KB threshold with PPMd (prediction by partial matching), which essentially is built for compressing text probabilistically.

```
cat romeo_and_juliet.txt | python3 steg.py encode
Compressing 149,660 bytes with PPMd...
Compressed:  41,882 bytes
Loaded 39283820 cached tracks
Data:     41882 bytes
  Tracks: 31534 bytes → 10000 tracks (252272 bits, 25.23 bits/track)
  Image:  10348 bytes → pixel-block cover
  Desc:   0 bytes → zero-width Unicode
Hash:     b8bece758fa2bae4575a6853bb0874dfa26d64cee821aee026009898d04336b4
Adding 10000 tracks in batches of 100...
  ...
  10000/10000
Image:    10348 bytes in pixel grid (152 KB JPEG, 203 KB b64)

Done. Playlist: b8bece758fa2bae4575a6853bb0874dfa26d64cee821aee026009898d04336b4
```

To decode:

```
python3 steg.py decode b8bece758fa2bae4575a6853bb0874dfa26d64cee821aee026009898d04336b4
```

<iframe data-testid="embed-iframe" style="border-radius:12px" src="https://open.spotify.com/embed/playlist/00MIk00pogxNxEA6PhbK3l?utm_source=generator" width="100%" height="352" frameBorder="0" allowfullscreen="" allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture" loading="lazy"></iframe>

---
### Conclusions
Is this actually a viable encoding scheme? Maybe? I'd say probably not though. You have a bunch of limitations, such as:

- Daily API limits and read/write limits on the playlist
- A massive song catalog which you need to load (could be optimized)

However, this isn't the point. Was it fun? Very. I would say I'm a fan of constrained optimization problems in general, and this was definitely one that was cool to see play out.

