# 摘录自这个[网站](https://jodezer.github.io/2017/05/golangSlice%E7%9A%84%E6%89%A9%E5%AE%B9%E8%A7%84%E5%88%99)：
# https://jodezer.github.io/2017/05/golangSlice%E7%9A%84%E6%89%A9%E5%AE%B9%E8%A7%84%E5%88%99

  以前一直以为go语言中的slice，也就是切片，其容量增长规则与std::vector一样，指数扩容，
每次扩容都增长一倍，没有去详细了解过源代码。直到同事丢给了我以下这段代码：
```golang
s := []int{1,2}
s = append(s,4,5,6)
fmt.Printf("%d %d",len(s),cap(s))
```

   如果简单地按照指数扩容，那么结果应该是 5，8。从初始化时的 2,2 扩容到 4,4 ，然后增长到 5, 8。但结果并不是这样，而是输出了 5,6。
  深入测试这段代码，每次往s中append两个元素，往后cap(s)都是6的倍数增长。
   源代码位于runtime/slice.go中，关于slice增长的函数是growslice，其中的代码，决定了slice的扩容规则

基本cap的增长规则

```golang
newcap := old.cap
	if newcap+newcap < cap {
		newcap = cap
	} else {
		for {
			if old.len < 1024 {
				newcap += newcap
			} else {
				newcap += newcap / 4
			}
			if newcap >= cap {
				break
			}
		}
	}
```

  此处的cap是旧容量加上新加入元素大小的结果，也就是此处slice扩容的理论上的最小值，old就是旧的slice。
可以看到cap增长基本规则是，若新入元素大小通过倍数增长能够hold住，
  那就根据旧容量是否超过1024决定是倍数增长还是1.25倍逐步增长；若新入元素大小超过了原有的容量，则新容量取两者相加计算出来的最小cap值。
  于是在例子中，经过扩容规则，()2 + 3 = 5) > (2 *2 = 4 ),newcap应当取5

# 内存对齐
  计算出了新容量之后，还没有完，出于内存的高效利用考虑，还要进行内存对齐
```golang
capmem := roundupsize(uintptr(newcap) * uintptr(et.size))
```

 newcap就是前文中计算出的newcap，et.size代表slice中一个元素的大小，capmem计算出来的就是此次扩容需要申请的内存大小。roundupsize函数就是处理内存对齐的函数。

```golang
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize {
		if size <= 1024-8 {
			return uintptr(class_to_size[size_to_class8[(size+7)>>3]])
		} else {
			return uintptr(class_to_size[size_to_class128[(size-1024+127)>>7]])
		}
	}
	if size+_PageSize < size {
		return size
	}
	return round(size, _PageSize)
}
```

 _MaxSmallSize的值在64位macos上是32«10，也就是2的15次方，32k。golang事先生成了一个内存对齐表。通过查找(size+7) » 3，
也就是需要多少个8字节，然后到class_to_size中寻找最小内存的大小。承接上文的size，应该是40，size_to_class8的内容是这样的：

```golang
size_to_class8：1 1 2 3 3 4 4 5 5 6 6 7 7 8 8 9 9 10 10 11 11...
```

查表得到的数字是4，而class_to_size的内容是这样的：

```golang
class_to_size：0 8 16 32 48 64 80 96 112 128 144 160 176 192 208 224 240 256...
```

因此得到最小的对齐内存是48字节。完成内存对齐计算后，重新计算应有的容量，也就是48/8 = 6。扩容得到的容量就是6了。
