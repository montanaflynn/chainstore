# ChainStore

Chainstore is simple key-value interface to a variety of storage engines organized as a chain of operations. A store adapter is just an engine interface to ```Open```, ```Close```, ```Put```, ```Get```, and ```Del``` . Each store has their own inherent properties and so when chained together, it makes for a useful combinations of data caching, flow and persistence depending on your application.

Here is an example of Boltdb and S3 stores chained together to provide fast read/writes to a local working dataset of 500MB and async S3 access for long-term persistence / retrieval. Check out the LRUManager below too, its wrapped around Boltdb to make sure only the least-recently-used key/values are persisted -- the manager can be used with any of the stores and with the chain, which is pretty nifty. This example is also here: example/main.go.

```go
package main

import (
  "fmt"
  "time"

  "github.com/nulayer/chainstore"
  "github.com/nulayer/chainstore/boltdb"
  "github.com/nulayer/chainstore/s3"
)

func main() {
  diskStore, err := boltdb.NewStore("/tmp/store.db", "myBucket")
  if err != nil {
    panic(err)
  }
  diskLru, err := chainstore.NewLRUManager(diskStore, 500*1024*1024) // 500MB of working data
  if err != nil {
    panic(err)
  }

  // NOTE: you'll have to supply your own keys in order for this example to work properly
  remoteStore, err := s3.NewStore("myBucket", "access-key", "secret-key")
  if err != nil {
    panic(err)
  }
  dataStore, err := chainstore.New(diskLru, remoteStore) // flows top-down
  if err != nil {
    panic(err)
  }

  //--

  // Save the object in the chain. It will be Put() synchronously into diskStore,
  // the boltdb engine, and then immediately dispatch background Put()'s to the
  // other stores down the chain, in this case S3.
  fmt.Println("Example 1...")
  obj := []byte{1, 2, 3}
  dataStore.Put("k", obj)
  fmt.Println("Put 'k':", obj, "in the chain")

  v, _ := dataStore.Get("k")
  fmt.Println("Grabbing 'k' from the chain:", v) // => [1 2 3]

  // For demonstration, let's grab the key directly from the store instead of
  // through the chain. This is pretty much the same as above, as the chain's Get()
  // stops once it finds the object.
  v, _ = diskStore.Get("k")
  fmt.Println("Grabbing 'k' directly from boltdb:", v) // => [1 2 3]

  // lets pause for a moment and then try to retrieve the value from the s3 store
  time.Sleep(1e9)

  // Grab the object from s3
  v, _ = remoteStore.Get("k")
  fmt.Println("Grabbing 'k' directly from s3:", v) // => [1 2 3]

  // Delete the object from everywhere
  dataStore.Del("k")
  time.Sleep(1e9) // pause for s3 demo
  v, _ = dataStore.Get("k")
  fmt.Println("Deleted 'k' from the chain (all stores). Get(k) returns:", v)

  //--

  // Another interesting behavior of the chain is when doing a Get(), it goes down
  // the entire chain looking for the value, and when found, it will Put() that
  // object back up the chain for subsequent retrievals. Lets see..
  fmt.Println("Example 2...")
  obj = []byte("hope you enjoy")
  dataStore.Put("hi", obj)
  fmt.Println("Put 'hi':", obj, "in the chain")
  time.Sleep(1e9) // lets wait for s3 again with more then enough time

  diskStore.Del("hi")
  v, _ = diskStore.Get("hi")
  fmt.Println("Delete 'hi' from boltdb. diskStore.Get(k) returns:", v)

  v, _ = dataStore.Get("hi")
  fmt.Println("Let's ask the chain for 'hi':", v)
  time.Sleep(1e9) // pause for bg routine to fill our local cache

  // The diskStore now has the value again from remoteStore lower down the chain.
  v, _ = diskStore.Get("hi")
  fmt.Println("Now, let's ask our diskStore again! diskStore.Get(k) returns:", v)

  // Also.. even though it hasn't been demonstrated here, the diskStore will only
  // store a max of 500MB (as defined with diskLru) worth of objects. Give it a shot.
}

/* OUTPUT:

Example 1...
Put 'k': [1 2 3] in the chain
Grabbing 'k' from the chain: [1 2 3]
Grabbing 'k' directly from boltdb: [1 2 3]
Grabbing 'k' directly from s3: [1 2 3]
Deleted 'k' from the chain (all stores). Get(k) returns: []
Example 2...
Put 'hi': [104 111 112 101 32 121 111 117 32 101 110 106 111 121] in the chain
Delete 'hi' from boltdb. diskStore.Get(k) returns: []
Let's ask the chain for 'hi': [104 111 112 101 32 121 111 117 32 101 110 106 111 121]
Now, let's ask our diskStore again! diskStore.Get(k) returns: [104 111 112 101 32 121 111 117 32 101 110 106 111 121]

*/
```

Currently supported stores: memcache (not the daemon), filesystem, boltdb, leveldb, s3, and a lru manager that can be layered ontop. The memcache implementation is just a simple ```map[string][]byte``` with the LRUManager for example.

Thx to other great projects:
- github.com/boltdb/bolt
- github.com/mitchellh/goamz
- github.com/syndtr/goleveldb

# TODO / Ideas

- Error channel where bad puts are communicated so they can be properly handled
further down the chain

- Idea: provide option to hash the input keys which would make each key fixed-length
and smaller footprint everywhere

- Timeout (with error notification) when adding an item to a store (ie. 60 seconds max to confirm)

- Consider a 'config' structure to pass to stores that can configure things like:
    * syncWrite bool - specifying for 2nd level chain stores if they be run in foreground
      like the first store (strong consistency)

    * For boltdb store, add option for ```fsyncTime uint``` which will queue up Puts over number
      of seconds, and then write them in a go routine after that period, batching in Go makes
      it wayy more efficient

    * For s3 store, add ACL with options: private, public_read, public_read_write, authenticated_read

- Add ```Run(func(s Store))``` on the ```Chain``` that will execute the function on each
  store context


## License

Copyright (c) 2014 Peter Kieltyka / Pressly Inc. www.pressly.com

MIT License

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

