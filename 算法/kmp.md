kmp

```
package main

import "fmt"

func main() {
	var m, n int
	var p, s string
	fmt.Scan(&n, &p, &m, &s)
	kmp(s, p, m, n)
}


func kmp(s, pattern string, s_len, pat_len int) int {
	next := next(pattern, pat_len)
	//fmt.Println(next)
	j := 0
	for i := 0; i < s_len; i++ {
		for j > 1 && s[i] != pattern[j] {
			j = next[j-1]
		}
		if s[i] == pattern[j] {
			if j == pat_len-1 {
				//return i-pat_len+1
				fmt.Print(i-pat_len+1," ")
				j=-1
				i-=pat_len-1
			}
			j++
		}

	}

	return -1
}

func next(pattern string, length int) []int {
	var next []int
	next = make([]int, length)

	/*for i:=0;i<len(next);i++{
		next[i]=-1
	}*/

	for i := 1; i < length-1; i++ {
		j := next[i-1]
		for j != 0 && pattern[j] != pattern[i] {
			j = next[j-1]
		}
		if pattern[j] == pattern[i] {
			j++
		}
		next[i] = j
	}
	return next
}

```

