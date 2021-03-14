---
layout: post

---
This is to discuss ways to generate random values in go using the rand package or the crypto rand. They both achieve the same end depending on your need regarding true randomness and performance.  Lets start with the common `math/rand` package

```go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	for i := 0; i < 5; i++ {
		fmt.Println(rand.Intn(50))
	}
}
```



```go
$ go run gotest.go 
31
37
47
9
31
$ go run gotest.go 
31
37
47
9
31
```

As you can see from above, the run produces the same sequence because the default seeding source.Now, lets try to use a different seeding for our random sequence generation.

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	for i := 0; i < 5; i++ {
		fmt.Println(r.Intn(50))
	}
}
```

> ```
> // Random numbers are generated by a Source. Top-level functions, such as
> // Float64 and Int, use a default shared Source that produces a deterministic
> // sequence of values each time a program is run. Use the Seed function to
> // initialize the default Source if different behavior is required for each run.
> // The default Source is safe for concurrent use by multiple goroutines, but
> // Sources created by NewSource are not.
> ```

What we simply did was provide new seeding via new  source. Please note this source is not safe for concurrent use unlike the shared source used globally.

```go
$ go run gotest.go 
44
17
41
23
5
$ go run gotest.go 
18
7
37
17
13
```

 Now to be truly random, you can use the better `crypto/rand` but with less performance.

## Crypto Rand

```go
package main

import (
	crypto_rand "crypto/rand"
	"encoding/base64"
	"fmt"
)

func GenerateRandomBytes(n int) ([]byte, error) {
	b := make([]byte, n)
	if _, err := crypto_rand.Read(b); err != nil {
		return nil, err
	}
	return b, nil
}

func GenerateRandomString(n int) (string, error) {
	b, err := GenerateRandomBytes(n)
	if err != nil {
		return "", err
	}
	return base64.URLEncoding.EncodeToString(b), nil
}
func main() {
	for i := 30; i < 35; i++ {
		if s, err := GenerateRandomString(i); err == nil {
			fmt.Println(s)
		}
	}
}

```

What we doing here is very simple, we are not using clock to generate our seedling.