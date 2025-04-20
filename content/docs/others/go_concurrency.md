---
title: Go 并发编程
date: "2024-12-05"
---

---

### 一个简单的 Demo

```go
func main() {
	ticker := time.NewTicker(time.Second)
	result := make(chan int)

	go func() {
		defer func() {
			fmt.Println("Ticker stop")
			close(result)
		}()
		cnt := 0
		for range ticker.C {
			cnt += 1
			fmt.Println("Polling...")
			result <- cnt
			if cnt > 5 {
				ticker.Stop()
				result <- -1
				return
			}
		}
	}()

	ch := make(chan int)
	go func() {
		defer func() {
			fmt.Println("goroutine close")
			close(ch)
		}()
		for x := range result {
			fmt.Println(x)
			if x < 0 {
				ch <- x
				return
			}
		}
	}()
	data := <-ch
	fmt.Println("Received data:", data)
	fmt.Println("Finished")
	time.Sleep(time.Second * 2)
}
```
