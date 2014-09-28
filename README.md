![Haneke](https://raw.githubusercontent.com/Haneke/HanekeSwift/master/Assets/github-header.png)

Haneke is a lightweight *generic* cache for iOS written in Swift. It's designed to be super-simple to use. Here's how you would initalize a JSON cache and fetch JSON objects from a url:

```swift
let cache = Cache<JSON>("movies")
let URL = NSURL(string: "http://haneke.io/movies.json")
cache.fetch(URL: URL).onSuccess { JSON in
    // Do something with JSON
}
```

Haneke provides a memory and LRU disk cache for `UIImage`, `NSData`, `JSON`, `String` or any other type that can be read or written as data.

Particularly, Haneke excels at working with images. It includes a zero-config image cache with automatic resizing. Everything is done in background, allowing for fast, responsive scrolling. Asking Haneke to load, resize, cache and display an *appropriately sized image* is as simple as:

```swift
imageView.hnk_setImageFromURL(url)
```

_Really._

##Features

* Generic cache with out-of-the-box support for `UIImage`, `NSData`, `JSON` and `String`
* First-level memory cache using `NSCache`
* Second-level LRU disk cache using the file system
* Asynchronous [fetching](#fetchers) of original values from network or disk
* All disk access is performed in background
* Thread-safe
* Automatic cache eviction on memory warnings or disk capacity reached
* Comprehensive unit tests
* Extensible by defining [custom formats](#formats), supporting [additional types](#supporting-additional-types) or implementing [custom fetchers](#custom-fetchers)

For images:

* Zero-config `UIImageView` category to use the cache, optimized for `UITableView` and `UICollectionView` cell reuse
* Background image resizing and decompression

##Using the cache

Haneke is generic cache with out-of-the-box support for `UIImage`, `NSData`, `JSON` and `String`. You can use the provided shared caches, or create your own. 

The cache is a key-value store. For example, here's how you would cache and then fetch some data.

```Swift
let cache = Haneke.sharedDataCache
        
cache.setValue(data, key: "some-data")
        
// Eventually...

cache.fetchValueForKey("some-data").onSuccess { data in
    // Do something with data
}
```

For cases in which the value is not readily available and must be fetched from network or disk, Haneke provides convenience `fetch` functions. Let's go back to the first example, now using a shared cache: 

```Swift
let cache = Haneke.sharedJSONCache
let URL = NSURL(string: "http://haneke.io/movies.json")
cache.fetch(URL: URL).onSuccess { JSON in
    // Do something with JSON
}
```

The above call will first attempt to fetch the required JSON from (in order) memory, disk or `NSURLCache`. If not available, Haneke will fetch the JSON from the source, return it and then cache it. In this case, the URL itself is used as the key.

Further customization can be achieved by using [formats](#formats), [supporting additional types](#supporting-additional-types) or implementing [custom fetchers](#custom-fetchers).

##Extra ♡ for images

Haneke provides convenience methods for `UIImageView` with optimizations for `UITableView` and `UICollectionView` cell reuse. Images will be resized appropriately and cached in a shared cache.

```swift
// Setting a remote image
imageView.hnk_setImageFromURL(url)

// Setting an image manually. Requires you to provide a key.
imageView.hnk_setImage(image, key: key)
```

The above lines take care of:

1. If cached, retrieving an appropriately sized image (based on the `bounds` and `contentMode` of the `UIImageView`) from the memory or disk cache. Disk access is performed in background.
2. If not cached, loading the original image from web/memory and producing an appropriately sized image, both in background. Remote images will be retrieved from the shared `NSURLCache` if available.
3. Setting the image and animating the change if appropriate.
4. Or doing nothing if the `UIImageView` was reused during any of the above steps.
5. Caching the resulting image.
6. If needed, evicting the least recently used images in the cache.

##Formats

Formats allow to specify the disk cache size and any transformations to the values before being cached. For example, the `UIImageView` extension uses a format that resizes images to fit or fill the image view as needed.

You can also use custom formats. Say you want to limit the disk capacity for icons to 10MB and apply rounded corners to the images. This is how it could like:

```swift
let cache = Haneke.sharedImageCache

let format = Format<UIImage>("icons", diskCapacity: 10 * 1024 * 1024) { image in
    return imageByRoundingCornersOfImage(image)
}
cache.addFormat(format)

let URL = NSURL(string: "http://haneke.io/icon.png")
cache.fetch(URL: URL, formatName: "icons").onSuccess { image in
    // image will be a nice rounded icon
}
```

Because we told the cache to use the "icons" format Haneke will execute the format transformation in background and return the resulting value.

##Fetchers

The `fetch` functions for urls and path are actually convenience methods. Under the hood Haneke uses fetcher objects. To illustrate, here's another way of fetching from a url:

```swift
let URL = NSURL(string: "http://haneke.io/icon.png")
let fetcher = NetworkFetcher<UIImage>(URL: URL)
cache.fetchValueForFetcher(fetcher).onSuccess { image in
    // Do something with image
}
```

Fetching an original value from network or disk is an expensive operation. Fetchers act as a proxy for the value, and allow Haneke to perform the fetch operation only if absolutely necessary.

In the above example the fetcher will be executed only if there is no value associated with `"http://haneke.io/icon.png"` in the memory or disk cache. If that happens, the fetcher will be responsible from fetching the original value, which will then be cached to avoid further network activity.

Haneke provides two specialized fetchers: `NetworkFetcher<T>` and `DiskFetcher<T>`. You can also implement your own fetchers by subclassing `Fetcher<T>`.

###Custom fetchers

Custom fetchers must subclass `Fetcher<T>` and are responsible for:

* Providing the key (e.g., `NSURL.absoluteString` in the case of `NetworkFetcher`) associated with the value to be fetched
* Fetching the value in background and calling the success or failure closure accordingly, both in the main queue
* Cancelling the fetch on demand, if possible

Fetchers are generic, and the only restriction on their type is that it must implement `DataConvertible`. 

##Supporting additional types

Haneke can cache any type that can be read and saved as data. This is indicated to Haneke by implementing the protocols `DataConvertible` and `DataRepresentable`.

```Swift
public protocol DataConvertible {
    typealias Result
    
    class func convertFromData(data:NSData) -> Result?
}

public protocol DataRepresentable {
    
    func asData() -> NSData!
}
```

This is how one could add support for `NSDictionary`:

```Swift
extension NSDictionary : DataConvertible, DataRepresentable {
    
    public typealias Result = NSDictionary
    
    public class func convertFromData(data:NSData) -> Result? {
        return NSKeyedUnarchiver.unarchiveObjectWithData(data) as? NSDictionary
    }
    
    public func asData() -> NSData! {
        return NSKeyedArchiver.archivedDataWithRootObject(self)
    }
    
}
```

Then creating a `NSDictionary` cache would be as simple as:

```swift
let cache = Cache<NSDictionary>("my-dictionaries")
```

##Roadmap

Haneke Swift is in initial development and its public API should not be considered stable.

##License

 Copyright 2014 Hermes Pique ([@hpique](https://twitter.com/hpique))    
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2014 Joan Romano ([@joanromano](https://twitter.com/joanromano))   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2014 Luis Ascorbe ([@lascorbe](https://twitter.com/Lascorbe))   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2014 Oriol Blanc ([@oriolblanc](https://twitter.com/oriolblanc))   
 
 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at
 
 http://www.apache.org/licenses/LICENSE-2.0
 
 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
