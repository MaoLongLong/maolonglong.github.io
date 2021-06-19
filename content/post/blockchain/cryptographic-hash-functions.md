---
title: '比特币笔记 —— 哈希函数'
date: 2021-06-19T22:46:59+08:00
draft: false
categories:
  - 区块链
tags:
  - Bitcoin
  - SHA256
---

比特币主要用到了密码学中的两个功能：

1. 哈希
2. 签名

密码学中用到的哈希函数被称为 **cryptographic hash function** ，它有 3 个重要的性质：

## collision free (collision resistance)

> 已知哈希函数 $H$，如果存在 $H(x)=H(y)$ 则称该现象为哈希碰撞

注意这个性质不是 “无哈希碰撞” ，因为哈希函数的输出范围远没有输入范围大，碰撞是不可避免的。所以说成 collision resistance（抗哈希碰撞）更合适些，即无法**人为**制造哈希碰撞。

曾经的 MD5 也号称 collision free，但早已经被密码学专家研究出人为制造哈希碰撞的方法，具体可以参考 [维基百科](https://zh.wikipedia.org/wiki/MD5#%E7%BC%BA%E9%99%B7)

## hiding

区分清楚**哈希**和**加密**的概念的话，hiding 就不难理解了，加密自然是有对应的解密方法，但是哈希函数的计算过程是单向的，不可逆的。hiding 性质前提是输入空间足够大，分布比较均匀。如果不是足够大，一般在 x 后面拼接一个随机数，如 $H\(x \parallel nonce\)$ 。

## puzzle friendly

puzzle friendly 是指挖矿过程中没有捷径，为了使输出值落在指定范围，只能一个一个去试。所以这个过程还可以作为工作量证明（proof of work）。

比特币挖矿的过程中实际就是找一个随机数 nonce，nonce 跟区块的块头（block header）里的其他信息合一起作为输入，得出的哈希值前 n 位都为 0。

### Secure Hash Algorithm 2

> 比特币中使用的哈希算法 [SHA256](https://zh.wikipedia.org/zh/SHA-2) 就满足了以上 3 个性质

以下是摘自维基百科伪代码，与 golang 的简单实现：

```
Note: All variables are unsigned 32 bits and wrap modulo 2^32 when calculating

Initialize variables
(first 32 bits of the fractional parts of the square roots of the first 8 primes 2..19):
h0 := 0x6a09e667
h1 := 0xbb67ae85
h2 := 0x3c6ef372
h3 := 0xa54ff53a
h4 := 0x510e527f
h5 := 0x9b05688c
h6 := 0x1f83d9ab
h7 := 0x5be0cd19

Initialize table of round constants
(first 32 bits of the fractional parts of the cube roots of the first 64 primes 2..311):
k[0..63] :=
   0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5, 0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
   0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3, 0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
   0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc, 0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
   0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7, 0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
   0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13, 0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
   0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3, 0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
   0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5, 0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
   0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208, 0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2

Pre-processing:
append the bit '1' to the message
append k bits '0', where k is the minimum number >= 0 such that the resulting message
    length (in bits) is congruent to 448(mod 512)
append length of message (before pre-processing), in bits, as 64-bit big-endian integer

Process the message in successive 512-bit chunks:
break message into 512-bit chunks
for each chunk
    break chunk into sixteen 32-bit big-endian words w[0..15]

    Extend the sixteen 32-bit words into sixty-four 32-bit words:
    for i from 16 to 63
        s0 := (w[i-15] rightrotate 7) xor (w[i-15] rightrotate 18) xor(w[i-15] rightshift 3)
        s1 := (w[i-2] rightrotate 17) xor (w[i-2] rightrotate 19) xor(w[i-2] rightshift 10)
        w[i] := w[i-16] + s0 + w[i-7] + s1

    Initialize hash value for this chunk:
    a := h0
    b := h1
    c := h2
    d := h3
    e := h4
    f := h5
    g := h6
    h := h7

    Main loop:
    for i from 0 to 63
        s0 := (a rightrotate 2) xor (a rightrotate 13) xor(a rightrotate 22)
        maj := (a and b) xor (a and c) xor(b and c)
        t2 := s0 + maj
        s1 := (e rightrotate 6) xor (e rightrotate 11) xor(e rightrotate 25)
        ch := (e and f) xor ((not e) and g)
        t1 := h + s1 + ch + k[i] + w[i]
        h := g
        g := f
        f := e
        e := d + t1
        d := c
        c := b
        b := a
        a := t1 + t2

    Add this chunk's hash to result so far:
    h0 := h0 + a
    h1 := h1 + b
    h2 := h2 + c
    h3 := h3 + d
    h4 := h4 + e
    h5 := h5 + f
    h6 := h6 + g
    h7 := h7 + h

Produce the final hash value (big-endian):
digest = hash = h0 append h1 append h2 append h3 append h4 append h5 append h6 append h7
```

```go
func sha256(b []byte) [32]byte {

	h0 := uint32(0x6a09e667)
	h1 := uint32(0xbb67ae85)
	h2 := uint32(0x3c6ef372)
	h3 := uint32(0xa54ff53a)
	h4 := uint32(0x510e527f)
	h5 := uint32(0x9b05688c)
	h6 := uint32(0x1f83d9ab)
	h7 := uint32(0x5be0cd19)

	k := [64]uint32{
		0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5, 0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
		0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3, 0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
		0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc, 0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
		0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7, 0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
		0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13, 0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
		0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3, 0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
		0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5, 0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
		0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208, 0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2}

	padded := append(b, 0x80)
	if len(padded)%64 < 56 {
		padded = append(padded, make([]byte, 56-len(padded)%64)...)
	} else {
		padded = append(padded, make([]byte, 56+64-len(padded)%64)...)
	}

	var l [8]byte
	binary.BigEndian.PutUint64(l[:], uint64(len(b)*8))
	padded = append(padded, l[:]...)

	n := len(padded) / 64
	for t := 0; t < n; t++ {
		chunk := padded[t*64 : t*64+64]

		var w [64]uint32
		for i := 0; i < 16; i++ {
			w[i] = binary.BigEndian.Uint32(chunk[i*4 : i*4+4])
		}

		for i := 16; i < 64; i++ {
			s0 := rightRotate(w[i-15], 7) ^ rightRotate(w[i-15], 18) ^ (w[i-15] >> 3)
			s1 := rightRotate(w[i-2], 17) ^ rightRotate(w[i-2], 19) ^ (w[i-2] >> 10)
			w[i] = w[i-16] + s0 + w[i-7] + s1
		}

		a := h0
		b := h1
		c := h2
		d := h3
		e := h4
		f := h5
		g := h6
		h := h7

		for i := 0; i < 64; i++ {
			s0 := rightRotate(a, 2) ^ rightRotate(a, 13) ^ rightRotate(a, 22)
			maj := (a & b) ^ (a & c) ^ (b & c)
			t2 := s0 + maj
			s1 := rightRotate(e, 6) ^ rightRotate(e, 11) ^ rightRotate(e, 25)
			ch := (e & f) ^ ((^e) & g)
			t1 := h + s1 + ch + k[i] + w[i]
			h = g
			g = f
			f = e
			e = d + t1
			d = c
			c = b
			b = a
			a = t1 + t2
		}

		h0 += a
		h1 += b
		h2 += c
		h3 += d
		h4 += e
		h5 += f
		h6 += g
		h7 += h
	}

	var digest [32]byte
	binary.BigEndian.PutUint32(digest[0:], h0)
	binary.BigEndian.PutUint32(digest[4:], h1)
	binary.BigEndian.PutUint32(digest[8:], h2)
	binary.BigEndian.PutUint32(digest[12:], h3)
	binary.BigEndian.PutUint32(digest[16:], h4)
	binary.BigEndian.PutUint32(digest[20:], h5)
	binary.BigEndian.PutUint32(digest[24:], h6)
	binary.BigEndian.PutUint32(digest[28:], h7)
	return digest
}

func rightRotate(x, n uint32) uint32 {
	return (x >> n) | (x << (32 - n))
}
```
