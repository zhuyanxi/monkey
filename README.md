# Monkey, Go!

This package implements actual arbitrary monkeypatching for Go. Yes really.

## Examples

```go
package main

import (
	"fmt"
	"os"
	"strings"

	"github.com/bouk/monkey"
)

func main() {
	monkey.Patch(fmt.Println, func(a ...interface{}) (n int, err error) {
		s := make([]interface{}, len(a))
		for i, v := range a {
			s[i] = strings.Replace(fmt.Sprint(v), "hell", "*bleep*", -1)
		}
		return fmt.Fprintln(os.Stdout, s...)
	})
	fmt.Println("what the hell?") // what the *bleep*?
}
```

```go
package main

import (
	"fmt"
	"net/http"
	"reflect"

	"github.com/bouk/monkey"
)

func main() {
	var guard *monkey.PatchGuard
	guard = monkey.PatchInstanceMethod(reflect.TypeOf(http.DefaultClient), "send", func(c *http.Client, req *http.Request) (*http.Response, error) {
		guard.Unpatch()
		defer guard.Restore()

		if req.URL.Scheme == "http" {
			return nil, fmt.Errorf("no http requests allowed")
		}

		return c.Do(req)
	})

	_, err := http.Get("http://google.com")
	fmt.Println(err) // Get http://google.com: no http requests allowed
	resp, err := http.Get("https://google.com")
	fmt.Println(resp.Status, err) // 200 OK <nil>
}
```

## Notes

1. Monkey sometimes fails to patch a function if inlining is enabled. Try running your tests with inlining disabled, for example: `go test -gcflags=-l`. The same command line argument can also be used for build.
2. Monkey won't work on some security-oriented operating system that don't allow memory pages to be both write and execute at the same time. With the current approach there's not really a reliable fix for this.
3. Monkey is not threadsafe. Or any kind of safe