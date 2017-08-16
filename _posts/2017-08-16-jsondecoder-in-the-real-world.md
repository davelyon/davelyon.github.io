---
title: Swift's JSONDecoder in the real world
description: A few helpful tips on dealing with less-than-ideal JSON responses with Codable in Swift 4 and Xcode 9.
---

# When JSONDecoder meets the real world, things get ugly...

If you've had a chance to try out the new `Codable` protocols recently made available in Xcode 9, you may have noticed that they're not exactly flexible in the way your previous JSON library might have been. The new `JSONDecoder` that is provided with the Swift standard library is a bit of a control freak, but with a little bit of protocol extension we can fix that.

Available now for Swift 3 and 4 (in Xcode 9), the `Codable` API might at first seem intimidating. What exactly is a `KeyedDecodingContainer` and how to I turn my raw JSON data in to one? I won't be spending too much time on those details as they've been well covered by others, but I do want to set up a few things concepts before we use them.

To begin, we'll focus on the `Decodable` part of the API since that's where we need to be more flexible in dealing with what we get from a server that may well be out of our control. When we want to decode our payload, we'll create an instance of a `JSONDecoder`object and ask it to decode our raw JSON data in to a generic type:

```swift
class JSONDecoder {
  func decode<T>(_ type: T.Type, from data: Data) throws -> T where T : Decodable { ... }
}

```

Let's use this to decode a `Book` class:

```swift
struct Book: Decodable {
  var title: String
  var author: String
}
```

This is already enough to decode a JSON object, but makes the assumption that the JSON keys match the variables exactly, so the payload should look like this: 

```swift
let jsonString = """
{   "title": "War and Peace: A protocol oriented approach to diplomacy",
    "author": "A. Keed Decoder"
}
"""
```

Given our finely crafted JSON payload, we can use a `JSONDecoder` object to decode this in to an instance of `Book` as follows:

```swift
if let data = jsonString.data(using: .utf8) {
  let decoder = JSONDecoder()

  if let book = try? decoder.decode(Book.self, from: data) {
    print(book.title) // "War and Peace: A protocol oriented approach to diplomacy"
  }

}
```

What's happening under the hood? `JSONDecoder` and the `Codable` protocols are making use of generics to save a lot of boilerplate, but if we wanted to write it all out, we certainly could:

```swift

struct Book: Decodable {
    var title: String
    var author: String

    // Everything from here on is generated for you by the compiler
    init(from decoder: Decoder) throws {
        let keyedContainer = try decoder.container(keyedBy: CodingKeys.self)
        title = try keyedContainer.decode(String.self, forKey: .title)
        author = try keyedContainer.decode(String.self, forKey: .author)
    }

    enum CodingKeys: String, CodingKey {
        case title
        case author
    }
}
```

The `KeyedDecodingContainer` is the most critical part to understand here, it's essentially a special dictionary that only allows you to access values with specific keys (specified by any object that conforms to the `CodingKey` protocol. This is what we'll leverage to help with handling special cases in APIs that don't quite provide what the `JSONDecoder` expects.

## Decoding Real-world API data

When decoding JSON from an API you often don't really get a say in how it's formatted. When you couple this with how strict the JSONDecoder object introduced alongside the new Codable pattern in Swift can be you quickly run in to things that break unexpectedly when faced with "real world" JSON data.

Take for example the "empty" object in JSON. You might expect that an object nested under another object would either be a complete and valid object or would be `null` -- and this is what JSONDecoder expects as well. But in many cases an API might return an empty object (that looks like `{}`) and cause your JSON decoding to fail even if you're decoding in to an Optional.

When faced with this issue recently I came up with a protocol based solution to work around this issue. I also made sure that the behavior was "opt-in" so that its possible to be flexible if there are differences between API endpoints.

Let's start with a simple example of what this might look like:

Our object is going to be a very simple "Book":

```swift
let jsonData = """
{   "title": "Blah",
    "frontCover": {},
    "backCover": { "image": "", "text": "It's good, read it" }
}
"""
```

Though our "frontCover" and "backCover" are optional we don't get a null value, we get an empty object. Let's create structs for the "Book" and "Cover":

```swift
struct BookCover: Decodable {
    var text: String
    var image: String?

    enum CodingKeys: String, CodingKey {
        case text
        case image
    }

}

struct Book: Decodable {
    var title: String
    var frontCover: BookCover?
    var backCover: BookCover?
}
```

If we try to decode our example as a `Book` we get an error:
```
keyNotFound(BookCover.CodingKeys.text, Swift.DecodingError.Context(codingPath: [Optional(Book.CodingKeys.frontCover), Optional(BookCover.CodingKeys.text)], 
debugDescription: "Key not found when expecting non-optional type String for coding key \"text\""))
```

This might not be quite the error you expected, but it makes sense: It opened up a new "KeyedDecodingContainer" and when trying to decode the first item it failed to find that key.

Let's create our protocol to opt-in to empty-object checking and then make our `Book` conform to it. I've elected to call it `JSONEmptyRepresentable` to indicate that during decoding this object _may_ be represented by an empty object rather than `null`.

```swift
public protocol JSONEmptyRepresentable {
    associatedtype CodingKeyType: CodingKey
}
```

You may note that it requires an associated type. That's so we can peek in to our potentially empty dictionary when decoding and see if any keys exist and then decode normally.

```swift
extension KeyedDecodingContainer {

    /// Given a nested object that is both optional, and explicitly allowed to be represented as an empty object `{}`, decode the object, or nil if
    /// the underlying representation is `null` or `{}` in the payload
    public func decodeIfPresent<T>(_ type: T.Type, forKey key: KeyedDecodingContainer.Key) throws -> T? where T : Decodable & JSONEmptyRepresentable {
	/// First, check if the key exists at all, and if not return `nil` as is the default behavior
        if contains(key) {
	    // Prepare to decode our nested object normally by opening a nested container
            let container = try nestedContainer(keyedBy: type.CodingKeyType.self, forKey: key)
	    
    	    // If there are no "keys" in the set, the object is empty and we consider it to be `nil`
            if container.allKeys.isEmpty { return nil}
        } else {
            return nil
        }

        return try decode(T.self, forKey: key)
    }
}
```

Now we can add our conformance:

```swift
extension BookCover: JSONEmptyRepresentable {
    typealias CodingKeyType = BookCover.CodingKeys
}
```

You may have noticed earlier we already added the (otherwise unnecessary) `CodingKeys` enum to `BookCover` -- this is why. We need to be able to open a properly keyed nested container to check if the object is in fact empty.

### Blank vs Null String Values

Another case where this can be an issue is missing String values. Some APIs "helpfully" return blank strings rather than a null value which can make it hard to properly handle Strings that you might instead want to make an `enum` out of. We can handle this in much the same way as the empty object.

First we'll create a protocol that lets specific types opt-in to this behavior called `JSONBlankRepresentable`.

```swift
public protocol JSONBlankRepresentable: RawRepresentable {}
```

Then we simply wrap the call to `decodeIfPresent` and add an `isEmpty` check:

```swift
extension KeyedDecodingContainer {

    /// Detect blank strings or nil values and return nil, otherwise
    public func decodeIfPresent<T>(_ type: T.Type, forKey key: KeyedDecodingContainer.Key) throws -> T? where T : Decodable & JSONBlankRepresentable, T.RawValue == String {
        if contains(key) {
            if let stringValue = try decodeIfPresent(String.self, forKey: key), stringValue.isEmpty == false {
                return T.init(rawValue: stringValue)
            }
        }
        return nil
    }
}
```

Here's a full example of how that might look in use: 

```swift
enum Genre: String, Codable {
    case thriller
    case history
}

extension Genre: JSONBlankRepresentable {}

struct Book: Decodable {
    var genre: Genre
    var subgenre: Genre?
}

let jsonData = """
{   "genre": "thriller",
    "subgenre": "",
}
"""
```

Nothing too complicated here. We're just extending protocols with more specific constraints and adding in different behavior. Any of the above can be repurposed to handle much more complex and strange API quirks you might encounter in the real world.


## When your representations differ across endpoints

Another issue that I hope doesn't come up too often, is that APIs that have survived for a long time in the wild can sometimes have "versions" (if you're lucky) or even potentially just have different representations for the same object based on which endpoint is requested. One way to handle this is to "smuggle" some context in to your decoding methods via the Decoder's `userInfo` dictionary. Of course we would prefer to have a more structured contract for these and avoid any "Stringly Typed" interfaces, so we'll want to leverage extensions to add a real interface to the JSONDecoder.

First, we'll need to have a `CodingUserInfoKey` that represents our context object. We'll extend `CodingUserInfoKey` to add in our keys:

```swift
extension CodingUserInfoKey {
    public static let decodingContext: CodingUserInfoKey = CodingUserInfoKey(rawValue: "decodingContext")!
}
```

Next, let's lay out our `DecodingContext` object to provide us with details of which type of response we are meant to decode:

```swift
public struct DecodingContext {
    public var responseType: String
    public init(responseType: String) {
        self.responseType = responseType
    }
}
```

Now we'll expose this as an `Optional` var in our decoder by adding a var and implementing a computed `var` on `Decoder` as follows:

```swift
extension Decoder {
    public var decodingContext: DecodingContext? { return userInfo[.decodingContext] as? DecodingContext }
}
```

And finally, we'll add a `convenience init` to JSONDecoder that can take our new context:

```swift
extension JSONDecoder {
    convenience init(context: DecodingContext) {
        self.init()
        self.userInfo[.decodingContext] = context
    }
}

```

Now you can check with any Decoder if the context object exists, and if so vary your behavior when decoding. Of course, you can also apply this same style of "configuration" extension for anything you might need to know while decoding as well. I'd love to hear about other experiences with "unusual" APIs, and any other workarounds people are finding for dealing with them. If anyone comes up with something cool, send a pull request to the playground repository that I've linked to with this post.
