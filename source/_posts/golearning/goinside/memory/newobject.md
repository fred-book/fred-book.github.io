### 创建对象

------

#### 创建对象

在Golang中，对于使用者来说，面对的是对象object。在创建对象的时候Golang会为对象申请内存，并进行赋初值。创建对象包含创建单个对象和对象数组，其函数如下：

```go
func newobject(typ *_type) unsafe.Pointer {
   return mallocgc(typ.size, typ, true) // 为typ类型的对象申请内存空间
}
```

```go
func newarray(typ *_type, n int) unsafe.Pointer {
   if n == 1 {
      return mallocgc(typ.size, typ, true) // 如果数组长度为1，则直接申请typ类型的内存空间
   }
   mem, overflow := math.MulUintptr(typ.size, uintptr(n))
   if overflow || mem > maxAlloc || n < 0 {
      panic(plainError("runtime: allocation size out of range"))
   }
   return mallocgc(mem, typ, true) // 如果数组长度大于1，则直接申请n个typ类型的内存空间
}
```

从上述代码可以看出，两个函数最终都是调用内存分配函数`mallocgc`。`mallocgc`函数主要实现了对极小对象，一般对象和大对象的内存申请。

##### 申请极小对象

对于长度小于16字节且不为指针的极小对象，会从`mcache`的`tiny`为其申请内存。`tiny`本身为16字节的内存块（从规格为2的mspan中申请的对象），Golang会非常精细化使用该16字节内存块。

```go
if noscan && size < maxTinySize {
    off := c.tinyoffset//获取当前tiny已经被使用的内存偏移，并按照合适长度对齐
    if size&7 == 0 {
        off = alignUp(off, 8)
    } else if goarch.PtrSize == 4 && size == 12 {
        off = alignUp(off, 8)
    } else if size&3 == 0 {
        off = alignUp(off, 4)
    } else if size&1 == 0 {
        off = alignUp(off, 2)
    }
    if off+size <= maxTinySize && c.tiny != 0 {// 如果tiny剩余长度足够分配申请长度，则返回对应地址，更新tiny使用偏移
        // The object fits into existing tiny block.
        x = unsafe.Pointer(c.tiny + off)
        c.tinyoffset = off + size
        c.tinyAllocs++
        mp.mallocing = 0
        releasem(mp)
        return x
    }
    // 从规格2的mspan中申请一个新的object
    span = c.alloc[tinySpanClass]
    v := nextFreeFast(span)
    if v == 0 {
        v, span, shouldhelpgc = c.nextFree(tinySpanClass)
    }
    // 将新申请的16字节对象清零
    x = unsafe.Pointer(v)
    (*[2]uint64)(x)[0] = 0
    (*[2]uint64)(x)[1] = 0
    // 如果申请内存size小于已被使用内存偏移或者tiny对象为空，则替换tiny；否则退化成一般对象申请
    if !raceenabled && (size < c.tinyoffset || c.tiny == 0) {
        // Note: disabled when race detector is on, see comment near end of this function.
        c.tiny = uintptr(x)
        c.tinyoffset = size
    }
    size = maxTinySize
}
```

如果tiny上的内存使用完或者不够本次申请，则从mache本地规格2的mspan中获取一个空闲的对象，如果本地mspan没有空闲的对象，则向mheap申请一个同规格的mspan。

##### 申请一般对象

对于指针类型极小对象和长度不小于16字节且小于32768字节的对象，则通过一般对象申请流程申请内存。

```go
    var sizeclass uint8
    if size <= smallSizeMax-8 { // 获取申请内存长度所属规格
        sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
    } else {
        sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
    }
    size = uintptr(class_to_size[sizeclass])
    spc := makeSpanClass(sizeclass, noscan) // 根据内存规格和是否扫描获取mspan规格
    span = c.alloc[spc] // 从mspan中获取object，如果本地的mspan已满，则向mheap重新申请一个mspan
    v := nextFreeFast(span)
    if v == 0 {
        v, span, shouldhelpgc = c.nextFree(spc)
    }
    x = unsafe.Pointer(v)
    if needzero && span.needzero != 0 {
        memclrNoHeapPointers(unsafe.Pointer(v), size) // 将申请的内存清零
    }
```

同样，优先从mache本地mspan中获取一个空闲的对象，如果本地mspan没有空闲的对象，则向mheap申请一个同规格的mspan。

##### 申请大对象

对于内存长度小于32768字节的对象，认为是大对象，直接从mheap申请内存。大对象长度大于32768字节，超过了前一节讲到的mspan规格，但是管理上仍以mspan对象管理，挂载在规格为1的mspan对象下。

```go
    shouldhelpgc = true
    // For large allocations, keep track of zeroed state so that
    // bulk zeroing can be happen later in a preemptible context.
    span = c.allocLarge(size, noscan) // 申请大对象，仍然按照mspan对象管理，挂载在规格为1的mspan对象上
    span.freeindex = 1
    span.allocCount = 1
    size = span.elemsize
    x = unsafe.Pointer(span.base())
    if needzero && span.needzero != 0 {
        if noscan {
            delayedZeroing = true
        } else {
            memclrNoHeapPointers(x, size)
            // We've in theory cleared almost the whole span here,
            // and could take the extra step of actually clearing
            // the whole thing. However, don't. Any GC bits for the
            // uncleared parts will be zero, and it's just going to
            // be needzero = 1 once freed anyway.
        }
    }
```

##### 更新arena

对于包含指针的对象，在申请完内存后会将对象的指针引用信息保存到arena。其实现函数为`runtime.heapBitsSetType`，这是一个非常复杂的函数。

```go
func heapBitsSetType(x, size, dataSize uintptr, typ *_type) {
   const doubleCheck = false // slow but helpful; enable to test modifications to this code

   const (
      mask1 = bitPointer | bitScan                        // 00010001
      mask2 = bitPointer | bitScan | mask1<<heapBitsShift // 00110011
      mask3 = bitPointer | bitScan | mask2<<heapBitsShift // 01110111
   )

   // dataSize is always size rounded up to the next malloc size class,
   // except in the case of allocating a defer block, in which case
   // size is sizeof(_defer{}) (at least 6 words) and dataSize may be
   // arbitrarily larger.
   //
   // The checks for size == goarch.PtrSize and size == 2*goarch.PtrSize can therefore
   // assume that dataSize == size without checking it explicitly.

   if goarch.PtrSize == 8 && size == goarch.PtrSize {
      // It's one word and it has pointers, it must be a pointer.
      // Since all allocated one-word objects are pointers
      // (non-pointers are aggregated into tinySize allocations),
      // initSpan sets the pointer bits for us. Nothing to do here.
      if doubleCheck {
         h := heapBitsForAddr(x)
         if !h.isPointer() {
            throw("heapBitsSetType: pointer bit missing")
         }
         if !h.morePointers() {
            throw("heapBitsSetType: scan bit missing")
         }
      }
      return
   }

   h := heapBitsForAddr(x)
   ptrmask := typ.gcdata // start of 1-bit pointer mask (or GC program, handled below)

   // 2-word objects only have 4 bitmap bits and 3-word objects only have 6 bitmap bits.
   // Therefore, these objects share a heap bitmap byte with the objects next to them.
   // These are called out as a special case primarily so the code below can assume all
   // objects are at least 4 words long and that their bitmaps start either at the beginning
   // of a bitmap byte, or half-way in (h.shift of 0 and 2 respectively).

   if size == 2*goarch.PtrSize {
      if typ.size == goarch.PtrSize {
         // We're allocating a block big enough to hold two pointers.
         // On 64-bit, that means the actual object must be two pointers,
         // or else we'd have used the one-pointer-sized block.
         // On 32-bit, however, this is the 8-byte block, the smallest one.
         // So it could be that we're allocating one pointer and this was
         // just the smallest block available. Distinguish by checking dataSize.
         // (In general the number of instances of typ being allocated is
         // dataSize/typ.size.)
         if goarch.PtrSize == 4 && dataSize == goarch.PtrSize {
            // 1 pointer object. On 32-bit machines clear the bit for the
            // unused second word.
            *h.bitp &^= (bitPointer | bitScan | (bitPointer|bitScan)<<heapBitsShift) << h.shift
            *h.bitp |= (bitPointer | bitScan) << h.shift
         } else {
            // 2-element array of pointer.
            *h.bitp |= (bitPointer | bitScan | (bitPointer|bitScan)<<heapBitsShift) << h.shift
         }
         return
      }
      // Otherwise typ.size must be 2*goarch.PtrSize,
      // and typ.kind&kindGCProg == 0.
      if doubleCheck {
         if typ.size != 2*goarch.PtrSize || typ.kind&kindGCProg != 0 {
            print("runtime: heapBitsSetType size=", size, " but typ.size=", typ.size, " gcprog=", typ.kind&kindGCProg != 0, "\n")
            throw("heapBitsSetType")
         }
      }
      b := uint32(*ptrmask)
      hb := b & 3
      hb |= bitScanAll & ((bitScan << (typ.ptrdata / goarch.PtrSize)) - 1)
      // Clear the bits for this object so we can set the
      // appropriate ones.
      *h.bitp &^= (bitPointer | bitScan | ((bitPointer | bitScan) << heapBitsShift)) << h.shift
      *h.bitp |= uint8(hb << h.shift)
      return
   } else if size == 3*goarch.PtrSize {
      b := uint8(*ptrmask)
      if doubleCheck {
         if b == 0 {
            println("runtime: invalid type ", typ.string())
            throw("heapBitsSetType: called with non-pointer type")
         }
         if goarch.PtrSize != 8 {
            throw("heapBitsSetType: unexpected 3 pointer wide size class on 32 bit")
         }
         if typ.kind&kindGCProg != 0 {
            throw("heapBitsSetType: unexpected GC prog for 3 pointer wide size class")
         }
         if typ.size == 2*goarch.PtrSize {
            print("runtime: heapBitsSetType size=", size, " but typ.size=", typ.size, "\n")
            throw("heapBitsSetType: inconsistent object sizes")
         }
      }
      if typ.size == goarch.PtrSize {
         // The type contains a pointer otherwise heapBitsSetType wouldn't have been called.
         // Since the type is only 1 pointer wide and contains a pointer, its gcdata must be exactly 1.
         if doubleCheck && *typ.gcdata != 1 {
            print("runtime: heapBitsSetType size=", size, " typ.size=", typ.size, "but *typ.gcdata", *typ.gcdata, "\n")
            throw("heapBitsSetType: unexpected gcdata for 1 pointer wide type size in 3 pointer wide size class")
         }
         // 3 element array of pointers. Unrolling ptrmask 3 times into p yields 00000111.
         b = 7
      }

      hb := b & 7
      // Set bitScan bits for all pointers.
      hb |= hb << wordsPerBitmapByte
      // First bitScan bit is always set since the type contains pointers.
      hb |= bitScan
      // Second bitScan bit needs to also be set if the third bitScan bit is set.
      hb |= hb & (bitScan << (2 * heapBitsShift)) >> 1

      // For h.shift > 1 heap bits cross a byte boundary and need to be written part
      // to h.bitp and part to the next h.bitp.
      switch h.shift {
      case 0:
         *h.bitp &^= mask3 << 0
         *h.bitp |= hb << 0
      case 1:
         *h.bitp &^= mask3 << 1
         *h.bitp |= hb << 1
      case 2:
         *h.bitp &^= mask2 << 2
         *h.bitp |= (hb & mask2) << 2
         // Two words written to the first byte.
         // Advance two words to get to the next byte.
         h = h.next().next()
         *h.bitp &^= mask1
         *h.bitp |= (hb >> 2) & mask1
      case 3:
         *h.bitp &^= mask1 << 3
         *h.bitp |= (hb & mask1) << 3
         // One word written to the first byte.
         // Advance one word to get to the next byte.
         h = h.next()
         *h.bitp &^= mask2
         *h.bitp |= (hb >> 1) & mask2
      }
      return
   }

   // Copy from 1-bit ptrmask into 2-bit bitmap.
   // The basic approach is to use a single uintptr as a bit buffer,
   // alternating between reloading the buffer and writing bitmap bytes.
   // In general, one load can supply two bitmap byte writes.
   // This is a lot of lines of code, but it compiles into relatively few
   // machine instructions.

   outOfPlace := false
   if arenaIndex(x+size-1) != arenaIdx(h.arena) || (doubleCheck && fastrandn(2) == 0) {
      // This object spans heap arenas, so the bitmap may be
      // discontiguous. Unroll it into the object instead
      // and then copy it out.
      //
      // In doubleCheck mode, we randomly do this anyway to
      // stress test the bitmap copying path.
      outOfPlace = true
      h.bitp = (*uint8)(unsafe.Pointer(x))
      h.last = nil
   }

   var (
      // Ptrmask input.
      p     *byte   // last ptrmask byte read
      b     uintptr // ptrmask bits already loaded
      nb    uintptr // number of bits in b at next read
      endp  *byte   // final ptrmask byte to read (then repeat)
      endnb uintptr // number of valid bits in *endp
      pbits uintptr // alternate source of bits

      // Heap bitmap output.
      w     uintptr // words processed
      nw    uintptr // number of words to process
      hbitp *byte   // next heap bitmap byte to write
      hb    uintptr // bits being prepared for *hbitp
   )

   hbitp = h.bitp

   // Handle GC program. Delayed until this part of the code
   // so that we can use the same double-checking mechanism
   // as the 1-bit case. Nothing above could have encountered
   // GC programs: the cases were all too small.
   if typ.kind&kindGCProg != 0 {
      heapBitsSetTypeGCProg(h, typ.ptrdata, typ.size, dataSize, size, addb(typ.gcdata, 4))
      if doubleCheck {
         // Double-check the heap bits written by GC program
         // by running the GC program to create a 1-bit pointer mask
         // and then jumping to the double-check code below.
         // This doesn't catch bugs shared between the 1-bit and 4-bit
         // GC program execution, but it does catch mistakes specific
         // to just one of those and bugs in heapBitsSetTypeGCProg's
         // implementation of arrays.
         lock(&debugPtrmask.lock)
         if debugPtrmask.data == nil {
            debugPtrmask.data = (*byte)(persistentalloc(1<<20, 1, &memstats.other_sys))
         }
         ptrmask = debugPtrmask.data
         runGCProg(addb(typ.gcdata, 4), nil, ptrmask, 1)
      }
      goto Phase4
   }

   // Note about sizes:
   //
   // typ.size is the number of words in the object,
   // and typ.ptrdata is the number of words in the prefix
   // of the object that contains pointers. That is, the final
   // typ.size - typ.ptrdata words contain no pointers.
   // This allows optimization of a common pattern where
   // an object has a small header followed by a large scalar
   // buffer. If we know the pointers are over, we don't have
   // to scan the buffer's heap bitmap at all.
   // The 1-bit ptrmasks are sized to contain only bits for
   // the typ.ptrdata prefix, zero padded out to a full byte
   // of bitmap. This code sets nw (below) so that heap bitmap
   // bits are only written for the typ.ptrdata prefix; if there is
   // more room in the allocated object, the next heap bitmap
   // entry is a 00, indicating that there are no more pointers
   // to scan. So only the ptrmask for the ptrdata bytes is needed.
   //
   // Replicated copies are not as nice: if there is an array of
   // objects with scalar tails, all but the last tail does have to
   // be initialized, because there is no way to say "skip forward".
   // However, because of the possibility of a repeated type with
   // size not a multiple of 4 pointers (one heap bitmap byte),
   // the code already must handle the last ptrmask byte specially
   // by treating it as containing only the bits for endnb pointers,
   // where endnb <= 4. We represent large scalar tails that must
   // be expanded in the replication by setting endnb larger than 4.
   // This will have the effect of reading many bits out of b,
   // but once the real bits are shifted out, b will supply as many
   // zero bits as we try to read, which is exactly what we need.

   p = ptrmask
   if typ.size < dataSize {
      // Filling in bits for an array of typ.
      // Set up for repetition of ptrmask during main loop.
      // Note that ptrmask describes only a prefix of
      const maxBits = goarch.PtrSize*8 - 7
      if typ.ptrdata/goarch.PtrSize <= maxBits {
         // Entire ptrmask fits in uintptr with room for a byte fragment.
         // Load into pbits and never read from ptrmask again.
         // This is especially important when the ptrmask has
         // fewer than 8 bits in it; otherwise the reload in the middle
         // of the Phase 2 loop would itself need to loop to gather
         // at least 8 bits.

         // Accumulate ptrmask into b.
         // ptrmask is sized to describe only typ.ptrdata, but we record
         // it as describing typ.size bytes, since all the high bits are zero.
         nb = typ.ptrdata / goarch.PtrSize
         for i := uintptr(0); i < nb; i += 8 {
            b |= uintptr(*p) << i
            p = add1(p)
         }
         nb = typ.size / goarch.PtrSize

         // Replicate ptrmask to fill entire pbits uintptr.
         // Doubling and truncating is fewer steps than
         // iterating by nb each time. (nb could be 1.)
         // Since we loaded typ.ptrdata/goarch.PtrSize bits
         // but are pretending to have typ.size/goarch.PtrSize,
         // there might be no replication necessary/possible.
         pbits = b
         endnb = nb
         if nb+nb <= maxBits {
            for endnb <= goarch.PtrSize*8 {
               pbits |= pbits << endnb
               endnb += endnb
            }
            // Truncate to a multiple of original ptrmask.
            // Because nb+nb <= maxBits, nb fits in a byte.
            // Byte division is cheaper than uintptr division.
            endnb = uintptr(maxBits/byte(nb)) * nb
            pbits &= 1<<endnb - 1
            b = pbits
            nb = endnb
         }

         // Clear p and endp as sentinel for using pbits.
         // Checked during Phase 2 loop.
         p = nil
         endp = nil
      } else {
         // Ptrmask is larger. Read it multiple times.
         n := (typ.ptrdata/goarch.PtrSize+7)/8 - 1
         endp = addb(ptrmask, n)
         endnb = typ.size/goarch.PtrSize - n*8
      }
   }
   if p != nil {
      b = uintptr(*p)
      p = add1(p)
      nb = 8
   }

   if typ.size == dataSize {
      // Single entry: can stop once we reach the non-pointer data.
      nw = typ.ptrdata / goarch.PtrSize
   } else {
      // Repeated instances of typ in an array.
      // Have to process first N-1 entries in full, but can stop
      // once we reach the non-pointer data in the final entry.
      nw = ((dataSize/typ.size-1)*typ.size + typ.ptrdata) / goarch.PtrSize
   }
   if nw == 0 {
      // No pointers! Caller was supposed to check.
      println("runtime: invalid type ", typ.string())
      throw("heapBitsSetType: called with non-pointer type")
      return
   }

   // Phase 1: Special case for leading byte (shift==0) or half-byte (shift==2).
   // The leading byte is special because it contains the bits for word 1,
   // which does not have the scan bit set.
   // The leading half-byte is special because it's a half a byte,
   // so we have to be careful with the bits already there.
   switch {
   default:
      throw("heapBitsSetType: unexpected shift")

   case h.shift == 0:
      // Ptrmask and heap bitmap are aligned.
      //
      // This is a fast path for small objects.
      //
      // The first byte we write out covers the first four
      // words of the object. The scan/dead bit on the first
      // word must be set to scan since there are pointers
      // somewhere in the object.
      // In all following words, we set the scan/dead
      // appropriately to indicate that the object continues
      // to the next 2-bit entry in the bitmap.
      //
      // We set four bits at a time here, but if the object
      // is fewer than four words, phase 3 will clear
      // unnecessary bits.
      hb = b & bitPointerAll
      hb |= bitScanAll
      if w += 4; w >= nw {
         goto Phase3
      }
      *hbitp = uint8(hb)
      hbitp = add1(hbitp)
      b >>= 4
      nb -= 4

   case h.shift == 2:
      // Ptrmask and heap bitmap are misaligned.
      //
      // On 32 bit architectures only the 6-word object that corresponds
      // to a 24 bytes size class can start with h.shift of 2 here since
      // all other non 16 byte aligned size classes have been handled by
      // special code paths at the beginning of heapBitsSetType on 32 bit.
      //
      // Many size classes are only 16 byte aligned. On 64 bit architectures
      // this results in a heap bitmap position starting with a h.shift of 2.
      //
      // The bits for the first two words are in a byte shared
      // with another object, so we must be careful with the bits
      // already there.
      //
      // We took care of 1-word, 2-word, and 3-word objects above,
      // so this is at least a 6-word object.
      hb = (b & (bitPointer | bitPointer<<heapBitsShift)) << (2 * heapBitsShift)
      hb |= bitScan << (2 * heapBitsShift)
      if nw > 1 {
         hb |= bitScan << (3 * heapBitsShift)
      }
      b >>= 2
      nb -= 2
      *hbitp &^= uint8((bitPointer | bitScan | ((bitPointer | bitScan) << heapBitsShift)) << (2 * heapBitsShift))
      *hbitp |= uint8(hb)
      hbitp = add1(hbitp)
      if w += 2; w >= nw {
         // We know that there is more data, because we handled 2-word and 3-word objects above.
         // This must be at least a 6-word object. If we're out of pointer words,
         // mark no scan in next bitmap byte and finish.
         hb = 0
         w += 4
         goto Phase3
      }
   }

   // Phase 2: Full bytes in bitmap, up to but not including write to last byte (full or partial) in bitmap.
   // The loop computes the bits for that last write but does not execute the write;
   // it leaves the bits in hb for processing by phase 3.
   // To avoid repeated adjustment of nb, we subtract out the 4 bits we're going to
   // use in the first half of the loop right now, and then we only adjust nb explicitly
   // if the 8 bits used by each iteration isn't balanced by 8 bits loaded mid-loop.
   nb -= 4
   for {
      // Emit bitmap byte.
      // b has at least nb+4 bits, with one exception:
      // if w+4 >= nw, then b has only nw-w bits,
      // but we'll stop at the break and then truncate
      // appropriately in Phase 3.
      hb = b & bitPointerAll
      hb |= bitScanAll
      if w += 4; w >= nw {
         break
      }
      *hbitp = uint8(hb)
      hbitp = add1(hbitp)
      b >>= 4

      // Load more bits. b has nb right now.
      if p != endp {
         // Fast path: keep reading from ptrmask.
         // nb unmodified: we just loaded 8 bits,
         // and the next iteration will consume 8 bits,
         // leaving us with the same nb the next time we're here.
         if nb < 8 {
            b |= uintptr(*p) << nb
            p = add1(p)
         } else {
            // Reduce the number of bits in b.
            // This is important if we skipped
            // over a scalar tail, since nb could
            // be larger than the bit width of b.
            nb -= 8
         }
      } else if p == nil {
         // Almost as fast path: track bit count and refill from pbits.
         // For short repetitions.
         if nb < 8 {
            b |= pbits << nb
            nb += endnb
         }
         nb -= 8 // for next iteration
      } else {
         // Slow path: reached end of ptrmask.
         // Process final partial byte and rewind to start.
         b |= uintptr(*p) << nb
         nb += endnb
         if nb < 8 {
            b |= uintptr(*ptrmask) << nb
            p = add1(ptrmask)
         } else {
            nb -= 8
            p = ptrmask
         }
      }

      // Emit bitmap byte.
      hb = b & bitPointerAll
      hb |= bitScanAll
      if w += 4; w >= nw {
         break
      }
      *hbitp = uint8(hb)
      hbitp = add1(hbitp)
      b >>= 4
   }

Phase3:
   // Phase 3: Write last byte or partial byte and zero the rest of the bitmap entries.
   if w > nw {
      // Counting the 4 entries in hb not yet written to memory,
      // there are more entries than possible pointer slots.
      // Discard the excess entries (can't be more than 3).
      mask := uintptr(1)<<(4-(w-nw)) - 1
      hb &= mask | mask<<4 // apply mask to both pointer bits and scan bits
   }

   // Change nw from counting possibly-pointer words to total words in allocation.
   nw = size / goarch.PtrSize

   // Write whole bitmap bytes.
   // The first is hb, the rest are zero.
   if w <= nw {
      *hbitp = uint8(hb)
      hbitp = add1(hbitp)
      hb = 0 // for possible final half-byte below
      for w += 4; w <= nw; w += 4 {
         *hbitp = 0
         hbitp = add1(hbitp)
      }
   }

   // Write final partial bitmap byte if any.
   // We know w > nw, or else we'd still be in the loop above.
   // It can be bigger only due to the 4 entries in hb that it counts.
   // If w == nw+4 then there's nothing left to do: we wrote all nw entries
   // and can discard the 4 sitting in hb.
   // But if w == nw+2, we need to write first two in hb.
   // The byte is shared with the next object, so be careful with
   // existing bits.
   if w == nw+2 {
      *hbitp = *hbitp&^(bitPointer|bitScan|(bitPointer|bitScan)<<heapBitsShift) | uint8(hb)
   }

Phase4:
   // Phase 4: Copy unrolled bitmap to per-arena bitmaps, if necessary.
   if outOfPlace {
      // TODO: We could probably make this faster by
      // handling [x+dataSize, x+size) specially.
      h := heapBitsForAddr(x)
      // cnw is the number of heap words, or bit pairs
      // remaining (like nw above).
      cnw := size / goarch.PtrSize
      src := (*uint8)(unsafe.Pointer(x))
      // We know the first and last byte of the bitmap are
      // not the same, but it's still possible for small
      // objects span arenas, so it may share bitmap bytes
      // with neighboring objects.
      //
      // Handle the first byte specially if it's shared. See
      // Phase 1 for why this is the only special case we need.
      if doubleCheck {
         if !(h.shift == 0 || h.shift == 2) {
            print("x=", x, " size=", size, " cnw=", h.shift, "\n")
            throw("bad start shift")
         }
      }
      if h.shift == 2 {
         *h.bitp = *h.bitp&^((bitPointer|bitScan|(bitPointer|bitScan)<<heapBitsShift)<<(2*heapBitsShift)) | *src
         h = h.next().next()
         cnw -= 2
         src = addb(src, 1)
      }
      // We're now byte aligned. Copy out to per-arena
      // bitmaps until the last byte (which may again be
      // partial).
      for cnw >= 4 {
         // This loop processes four words at a time,
         // so round cnw down accordingly.
         hNext, words := h.forwardOrBoundary(cnw / 4 * 4)

         // n is the number of bitmap bytes to copy.
         n := words / 4
         memmove(unsafe.Pointer(h.bitp), unsafe.Pointer(src), n)
         cnw -= words
         h = hNext
         src = addb(src, n)
      }
      if doubleCheck && h.shift != 0 {
         print("cnw=", cnw, " h.shift=", h.shift, "\n")
         throw("bad shift after block copy")
      }
      // Handle the last byte if it's shared.
      if cnw == 2 {
         *h.bitp = *h.bitp&^(bitPointer|bitScan|(bitPointer|bitScan)<<heapBitsShift) | *src
         src = addb(src, 1)
         h = h.next().next()
      }
      if doubleCheck {
         if uintptr(unsafe.Pointer(src)) > x+size {
            throw("copy exceeded object size")
         }
         if !(cnw == 0 || cnw == 2) {
            print("x=", x, " size=", size, " cnw=", cnw, "\n")
            throw("bad number of remaining words")
         }
         // Set up hbitp so doubleCheck code below can check it.
         hbitp = h.bitp
      }
      // Zero the object where we wrote the bitmap.
      memclrNoHeapPointers(unsafe.Pointer(x), uintptr(unsafe.Pointer(src))-x)
   }

   // Double check the whole bitmap.
   if doubleCheck {
      // x+size may not point to the heap, so back up one
      // word and then advance it the way we do above.
      end := heapBitsForAddr(x + size - goarch.PtrSize)
      if outOfPlace {
         // In out-of-place copying, we just advance
         // using next.
         end = end.next()
      } else {
         // Don't use next because that may advance to
         // the next arena and the in-place logic
         // doesn't do that.
         end.shift += heapBitsShift
         if end.shift == 4*heapBitsShift {
            end.bitp, end.shift = add1(end.bitp), 0
         }
      }
      if typ.kind&kindGCProg == 0 && (hbitp != end.bitp || (w == nw+2) != (end.shift == 2)) {
         println("ended at wrong bitmap byte for", typ.string(), "x", dataSize/typ.size)
         print("typ.size=", typ.size, " typ.ptrdata=", typ.ptrdata, " dataSize=", dataSize, " size=", size, "\n")
         print("w=", w, " nw=", nw, " b=", hex(b), " nb=", nb, " hb=", hex(hb), "\n")
         h0 := heapBitsForAddr(x)
         print("initial bits h0.bitp=", h0.bitp, " h0.shift=", h0.shift, "\n")
         print("ended at hbitp=", hbitp, " but next starts at bitp=", end.bitp, " shift=", end.shift, "\n")
         throw("bad heapBitsSetType")
      }

      // Double-check that bits to be written were written correctly.
      // Does not check that other bits were not written, unfortunately.
      h := heapBitsForAddr(x)
      nptr := typ.ptrdata / goarch.PtrSize
      ndata := typ.size / goarch.PtrSize
      count := dataSize / typ.size
      totalptr := ((count-1)*typ.size + typ.ptrdata) / goarch.PtrSize
      for i := uintptr(0); i < size/goarch.PtrSize; i++ {
         j := i % ndata
         var have, want uint8
         have = (*h.bitp >> h.shift) & (bitPointer | bitScan)
         if i >= totalptr {
            if typ.kind&kindGCProg != 0 && i < (totalptr+3)/4*4 {
               // heapBitsSetTypeGCProg always fills
               // in full nibbles of bitScan.
               want = bitScan
            }
         } else {
            if j < nptr && (*addb(ptrmask, j/8)>>(j%8))&1 != 0 {
               want |= bitPointer
            }
            want |= bitScan
         }
         if have != want {
            println("mismatch writing bits for", typ.string(), "x", dataSize/typ.size)
            print("typ.size=", typ.size, " typ.ptrdata=", typ.ptrdata, " dataSize=", dataSize, " size=", size, "\n")
            print("kindGCProg=", typ.kind&kindGCProg != 0, " outOfPlace=", outOfPlace, "\n")
            print("w=", w, " nw=", nw, " b=", hex(b), " nb=", nb, " hb=", hex(hb), "\n")
            h0 := heapBitsForAddr(x)
            print("initial bits h0.bitp=", h0.bitp, " h0.shift=", h0.shift, "\n")
            print("current bits h.bitp=", h.bitp, " h.shift=", h.shift, " *h.bitp=", hex(*h.bitp), "\n")
            print("ptrmask=", ptrmask, " p=", p, " endp=", endp, " endnb=", endnb, " pbits=", hex(pbits), " b=", hex(b), " nb=", nb, "\n")
            println("at word", i, "offset", i*goarch.PtrSize, "have", hex(have), "want", hex(want))
            if typ.kind&kindGCProg != 0 {
               println("GC program:")
               dumpGCProg(addb(typ.gcdata, 4))
            }
            throw("bad heapBitsSetType")
         }
         h = h.next()
      }
      if ptrmask == debugPtrmask.data {
         unlock(&debugPtrmask.lock)
      }
   }
}
```

该函数的主要作用是更新arena下申请指针所对应内存块的指针引用信息，其数据来源于申请对象对应类型的gcdata。
