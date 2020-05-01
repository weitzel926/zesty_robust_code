---
title:  "Decoding JSON with Decoder in Swift"
date:   2020-03-26 18:24:03 -0600
categories: swift 
tags: swift json decodable
header:
   overlay_image: /assets/images/decodersplash.jpg
   overlay_filter: 0.5
---

A deep dive into the Decoder protocol...

To say parsing JSON in Swift is annoying is a bit of an understatement.  For those of us used to Objective-C’s more loose typing, parsing JSON was as simple as making your data models initialize themselves with an NSDictionary.  The strong typing of Swift removes this flexibility.  Recently (well, not SUPER recently - Swift 4, to be precise), a protocol was added which makes JSON parsing easier, with some hard to understand caveats.  

In the process of adding Swift to a large Objective-C app, I was unable to find documentation that describes how this works in one place, so I will attempt to solve that here.  In addition, this system uses many Swift concepts so it is a useful exercise for an Objective-C dev looking to understand Swift better.

Swift is an open source project, and one can look directly at the source code for the JSON APIs.  Most of the relevant bits related to JSON are included in the following file: [JSONEncoder][json-encoder]

These APIs include both decoding (turning JSON into Swift data structures) and encoding (turning your Swift data structures into JSON).  This post discusses decoding.

### JSONDecoder

Apple documentation: [JSON-Decoder][json-decoder-docs]

The JSONDecoder takes JSON objects and turns them into Swift data type instances (and includes some ways to configure how this is accomplished).  The key method to this is:

{% highlight swift %}
func decode<T>(T.type, from: Data) -> T
{% endhighlight %}

To use this, your code would look something like this:

{% highlight swift %}
let decoder = JSONDecoder()
let instanceOfMyDataModel = try decoder.decode(typeOfMyDataModel, from: json)
{% endhighlight %}

This is pretty straightforward, even with the generics.  The JSONDecoder is initialized, after which you can use the decode method to generate an instance of your data model.  It takes the type of your data model and the JSON as a parameter.  The JSON is of type Data, which is just a byte buffer in UTF-8 format.  After it decodes, it returns an instance of your type.  There are plenty of things that can throw errors here, so we need to put this is in a try block.  

### String to Data

Consider the following JSON:

{% highlight swift %}
let json = """
{
   "name": "Mr Soapy Bar Soap",
   "sku": "MrSoapy_Bar_1"
   "description": "Mr Soapy keeps you fresh and clean"
}
""".data(using: .utf8)!
{% endhighlight %}

There are two things to be aware of.  You will often see the “””<Some String>””” literal format for JSON because it allows you to format your JSON in a readable way directly in your code.  This is handy for texts like this or using small chunks of JSON in a playground.  Most of the time, you will be getting your JSON from the network.  Secondly, the data method is converting the string into a UTF-8 encoded Data instance.  It has the following signature:
	
{% highlight swift %}
func data(using encoding: String.Encoding, allowLossyConversion: Bool = false) -> Data?
{% endhighlight %}

If you went and looked at the Swift String struct documentation expecting to find this method, you were probably surprised it wasn’t there.  The data method is actually defined in [StringProtocol][string_protocol_docs], which String (and SubString) conform to.  So, it's there, but tricky to find in the documentation. If you look at the code, you will also see it is in a different place as well.

Open a new playground and add this:

{% highlight swift %}
let string = String("test")
let data = string.data(using: .utf8)!
{% endhighlight %}

You should find that this compiles and runs fine.  Now add the following:

{% highlight swift %}
let nsstring = NSString("test")
let data2 = nsstring.data(using: .utf8)!
{% endhighlight %}

This fails to compile with a type UInt has no member ‘utf8’ error.  What is happening here gives us insight into why this works behind the scenes.  The compile error can be fixed with the following modification:

{% highlight swift %}
let data2 = nsstring.data(using: String.Encoding.utf8.rawValue)!
{% endhighlight %}

So, for an NSString instance, the code is bridged from Objective-C, which gives a different signature because it is an entirely different type.  There is typically no reason to use NSString in Swift code, but this is confusing because some of the String methods you are accustomed to (like the data method) are not documented in the String class, they are in StringProtocol.  

### The JSON spec

One should also be aware that a mapping like what we know on the iOS side as a "dictionary" is called an "object" in the [JSON specification][json_spec].  For the remainder of this document, we will refer to them only as "dictionaries", but it may be helpful to note the difference. 

### Codable & Decodable

Swift provides two protocols to parse (Decodable) and build (Encodable) JSON.  The Codable protocol represents both Encodable & Decodable.  When you use Decodable you will usually end up using Codable.  It is defined as:

{% highlight swift %}
public typealias Codable = Encodable & Decodable
{% endhighlight %}

We now have some JSON and we know which protocol we need to parse it.  We also know the basics of using JSONDecoder.  It is time to look at our data model.  Consider the following playground:

{% highlight swift %}
import Foundation

struct Product : Codable {
   let name: String
   let sku: String
   let price: Double
   let description: String
}

let jsonData = """
{
   "name": "Mr Soapy Bar Soap",
   "sku": "MrSoapy_Bar_1",
   "price": 1.50,
   "description": "Mr Soapy keeps you fresh and clean"
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
let product = try! decoder.decode(Product.self, from: jsonData)
{% endhighlight %}

This is similar to what we have already seen, except that we define a struct called “Product” that conforms the Decodable protocol (by conforming to Codable).  This is our data model.  If we execute this, we will get a valid instance of Product, based on the jsonData instance.

Decodable itself is available in [Codable.swift][swift_codable] and looks like:

{% highlight swift %}
@_implicitly_synthesizes_nested_requirement("CodingKeys")
public protocol Decodable {
   init(from decoder: Decoder) throws
}
{% endhighlight %}

Thus, the only thing you must implement for Decodable is the init(from decoder:) method.  But...wait...we didn’t implement this did we?  If we peer into the [documentation][swift_decodable_init_docs] we can see that a default implementation is provided.  How is that?  Welp, gentle coder, you should not worry yourself about such things.  The compiler actually generates this for you with the Decodable protocol.  Consider it magic, consider it a bunch of C++ code you don’t want to read -- just know that on this one, the compiler has got your back. 

### Problem - JSON Has Poor Naming - Introducing Coding Keys

Perhaps the most common problem that will cause our JSON to map poorly to our data model is the naming of key values inside of our JSON.  Fortunately, Codable has a Coding Keys scheme that can help us use the names we want in our code.  Let’s start by understanding how this works in the default case, and then build on that to customize our property names with respect to JSON.  

There is another piece of compiler sorcery in our playground, and it has to do with how the system knows the names of your JSON keys map to the properties in your data model.  If you modify your data model in your playground to the following code you should see that things still compile and execute as expected:

{% highlight swift %}
struct Product : Codable {   
   enum CodingKeys: String, CodingKey {
      case name = "name"
      case sku = "sku"
      case price = "price"
      case description = "description"
   }
    
   let name: String
   let sku: String
   let price: Double
   let description: String
}
{% endhighlight %}

The CodingKeys enum is a special nested enum that conforms to the CodingKey protocol (We will get to why you may need to know this later).  CodingKeys is not a reserved name, you can name the enum whatever you want, although CodingKeys is very common.  This enum provides the mapping between properties and keys.  Alternatively, when you get to encoding this data model, the encoder will ONLY encode the properties it has coding keys for.  

It should be noted that this all working depends on your properties being of types that conform to Codable.  In Codable.swift, you can see the extensions to types to support Codable, such as Int below (You will find these in Codable.swift):

{% highlight swift %}
extension Int: Codable {
   public init(from decoder: Decoder) throws {
      self = try decoder.singleValueContainer().decode(Int.self)
   }

   public func encode(to encoder: Encoder) throws {
      var container = encoder.singleValueContainer()
      try container.encode(self)
   }
}
{% endhighlight %}

### How do CodingKeys Work?

CodingKeys is just the typical name of the enum you use that conforms to the CodingKey protocol, so a more appropriate question would be how does CodingKey work?  To find out, the best reference is the Swift proposal for Codable which was eventually implemented ([Swift proposal][swift_coding_key]) in Swift 4.

CodingKey itself is a protocol that defines what a type needs to do in order to be used as a key.  The protocol is very simple:

{% highlight swift %}
public protocol CodingKey : CustomDebugStringConvertible, CustomStringConvertible {
   var stringValue: String { get }
   init?(stringValue: String)

   var intValue: Int? { get }
   init?(intValue: Int)
}
{% endhighlight %}

It contains a stringValue and an initializer that takes a string for keys (for collections that have named elements) and an intValue with an Int initializer (for collections that are indexed).  It is always possible (we will work an example below) to customize your own CodingKey instance for your own purposes.  So, how do we get from the enum above to a CodingKey?

The short answer:  the compiler does it for us.  It will generate an enum that conforms to CodingKey.  It will override the stringValue (or intValue) getter to return the correct value based on what you provided in the enum.  The link above for the swift archival serialization proposal will show you a sample generated class, if you are curious.  The decoder, whether you are using the compiler-generated default or you implemented it yourself will now have strongly-typed objects to parse the collections with.  

### Using CodingKeys To Customize Our Parsing

Below is an extreme example, but what if your JSON author decided to use single character key names?  Obviously, you don’t want your data model to have n, s, p and d properties - so you will need to map the poorly named properties into properties that are readable in your code. 

{% highlight swift %}
let jsonData = """
{
   "n": "Mr Soapy Bar Soap",
   "s": "MrSoapy_Bar_1",
   "p": 1.50,
   "d": "Mr Soapy keeps you fresh and clean"
}
""".data(using: .utf8)!
{% endhighlight %}

You can use CodingKeys to fix this.  Consider the following:

{% highlight swift %}
struct Product : Codable {
   enum CodingKeys: String, CodingKey {
      case name = "n"
      case sku = "s"
      case price = "p"
      case description = "d"
   }

   let name: String
   let sku: String
   let price: Double
   let description: String
}

let decoder = JSONDecoder()
let product = try! decoder.decode(Product.self, from: jsonData)
print(product.name)
// Mr Soapy Bar Soap
{% endhighlight %}

The enum defines the mapping, and the correct CodingKey objects are created that result in your data model being able to use nice names to store the poor names in the JSON.

### let vs var

You may have noticed that I have declared all the properties in my data models as let.  Given the nature of the code examples, this is intentional.  In all of the cases I have presented so far, we are just reading in the JSON and storing it to access it later.  Without the need for mutability, there is no reason for them to be mutable.  That said, many data models will be initialized with server-side JSON, modified, and then updated via a RESTful back-end API.  Given this, it is probably more common that your data model properties are var.  That said, whenever you have a property that you don’t intend for anyone to modify, you can enforce that by making it let.

Also note, to have a mutable data structure, you need to decode it to a mutable object as well, as shown below.

{% highlight swift %}
import Foundation
struct Product : Codable {
   enum CodingKeys: String, CodingKey {
      case name = "n"
      case sku = "s"
      case price = "p"
      case description = "d"
   }

   var name: String
   let sku: String
   let price: Double
   let description: String
}

let jsonData = """
{
   "n": "Mr Soapy Bar Soap",
   "s": "MrSoapy_Bar_1",
   "p": 1.50,
   "d": "Mr Soapy keeps you fresh and clean"
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
var product = try! decoder.decode(Product.self, from: jsonData)
print(product.name)
// Mr Soapy Bar Soap
product.name = "test"
print(product.name)
// test
{% endhighlight %}

### Parameters That Might Not Be There

If we consider the JSON we’ve been working with, what happens if the description property is only there sometimes?  For example, what if our playground looks like this:

{% highlight swift %}
import Foundation

class Product : Codable {
   let name: String
   let sku: String
   let price: Double
   let description: String;
}

let jsonData = """
{
   "name": "Mr Soapy Bar Soap",
   "sku": "MrSoapy_Bar_1",
   "price": 1.50
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
let product = try! decoder.decode(Product.self, from: jsonData)

print(product.name)
{% endhighlight %}

This actually generates an error for the key “description” not being found.  We can resolve this in one of two ways:

{% highlight swift %}
var description: String!  // make it optional
var description: String = “”  // provide a default value
{% endhighlight %}

Which should you use?  The drawbacks of both approaches are clear.  If you need to check for the existence of a property to use later in your code, you probably want to make it an optional.  If you are likely to just be passing that back in further RESTful POST/PUT calls, you can probably just have the default value and avoid the unwrapping.  Even with the extra overhead of unwrapping, I’d likely default to the optional unless I am certain I will never have to check to see if the property has actual data in it.

### Class vs Struct

Another big decision you will have to make is whether to make your data model types structs or classes.  Apple suggests that you use value types (struct) instead of reference types when possible.  They provide a rather detailed WWDC talk from 2015 that describes their reasoning for this ([WWDC Value Types vs Reference Types][wwdc_valuetypes_vs_referencetypes]), which I highly recommend viewing.  

One additional factor that will weigh on your decision to create your data models as value types or reference types is your need to interoperate with Objective-C.  You cannot use Swift structs in Objective-C, and this alone may force your hand and cause you to use classes.  Note that if you use either a struct or class, the auto property synthesis will occur unless you overwrite the init(from decoder:).  

### JSON With Nested Array

This is all great, but it relies on our JSON being a well known and flat structure.  What if our JSON is more complex?  Consider the following playground:

{% highlight swift %}
import Foundation

struct ProductCategory : Codable {
   struct Product : Codable {
      let name: String
      let sku: String
      let price: Double
      let description: String
   }
    
   let categoryName: String
   let products:[Product]
}

let jsonData = """
{
   "categoryName" : "soap",
   "products" : [
         {
            "name": "Mr Soapy Bar Soap",
            "sku": "MrSoapy_Bar_1",
            "price": 1.50,
            "description": "Mr Soapy keeps you fresh and clean"
         },
         {
            "name": "Squeeky Clean Liquid Soap",
            "sku": "Squeeky_bottle_1",
            "price": 2.50,
            "description": "Squeeky clean for pourable freshitude"
         }]
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
let productCategory = try! decoder.decode(ProductCategory.self, from: jsonData)

for product in productCategory.products {
   print(product.name)
}
{% endhighlight %}

So, in this example we have a more complex JSON with an array in it.  This array is an array of dictionary objects.  The way that Codable encourages you to deal with this with a nested data model, as shown above.  

That said, you can also separate the two structs like below:

{% highlight swift %}
struct Product : Codable {
   let name: String
   let sku: String
   let price: Double
   let description: String
}

struct ProductCategory : Codable {
   let categoryName: String
   let products:[Product]
}
{% endhighlight %}

Having them nested provides some further context to the reader by associating the inner type with the outer type.  You can see how you would use this outside of Codeable below:

{% highlight swift %}
import Foundation

struct ProductCategory : Codable {
   struct Product : Codable {
      let name: String
      let sku: String
      let price: Double
      let description: String
   }

   let categoryName: String
   let products:[Product]
}

let product = ProductCategory.Product(name: "testProduct", sku: "testSku", price: 0.0, description: "test");

let productCategory = ProductCategory(categoryName: "testCategory", products: [product])
print(productCategory.products[0].name)
// testProduct
{% endhighlight %}

Whether this is a readability advantage to you is something you get to decide for yourself.  Just know the option is there. 

### JSON with Nested Dictionary

In addition to arrays, it is very common to see dictionaries embedded in dictionaries in JSON.  For example, consider this playground:

{% highlight swift %}
import Foundation

struct ProductCategory : Codable {
   struct Product : Codable {
      let name: String
      let sku: String
      let price: Double
      let description: String
   }

   let categoryName: String
   let product:Product
}

let jsonData = """
{
   "categoryName" : "soap",
   "product" :    {
                  "name": "Mr Soapy Bar Soap",
                  "sku": "MrSoapy_Bar_1",
                  "price": 1.50,
                  "description": "Mr Soapy keeps you fresh and clean"
                  }
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
let productCategory = try! decoder.decode(ProductCategory.self, from: jsonData)

print(productCategory.product.name)
// Mr Soapy Bar Soap
{% endhighlight %}

In this case, the product category only contains one product and it is a dictionary.  To seamlessly access this through Decodable, we again use a nested type.  

### Problem - Heavy Nesting

Ideally, you are building your iOS app and your JSON at the same time.  When this is done, it can make sense to have the back-end data model, JSON, and the frontend (iOS, in this case) data model all be the same.  However, this is often not the reality.  The goals of a back-end data model might diverge from the goals of an app’s data model, which can cause you to over-complicate your app’s data model.  For example, consider an API where you place an HTTP GET to return a single specific product (let’s say you pass in the SKU and it returns a product).  What would you want your data model to be like if the API’s data format returned this:

{% highlight swift %}
{
   "elements": [{
      "element": {
               "name": "Mr Soapy Bar Soap",
               "sku": "MrSoapy_Bar_1",
               "price": 1.50,
               "description": "Mr Soapy keeps you fresh and clean"
      }
   }]
}
{% endhighlight %}

In this case, the API writer has decided to add a lot of flexibility, generality and verbosity (we only wanted one product, but they passed a structure capable of handling not only multiple products but multiple types of data).  While this might be desired to enable dynamic additions to the API’s data format, it will likely result in an inconvenient API for us.  To parse this, your data model might look like this:

{% highlight swift %}
struct Product : Codable {
   let name: String
   let sku: String
   let price: Double
   let description: String
}

struct ElementData : Codable {
   let element: Product
}

struct Elements : Codable {
   let elements: [ElementData]
}
{% endhighlight %}

Note that ElementData is un-named in the JSON, because elements are an array of objects and not keyed.  Codable is able to deal with this fine.  That said, this created a structure that is going to make little sense in our code.  We can solve this by ignoring other elements of data structure as follows:

{% highlight swift %}
let decoder = JSONDecoder()
let elements = try! decoder.decode(Elements.self, from: jsonData)
let product = elements.elements[0].element
print(product.name)
// Mr Soapy Bar Soap
{% endhighlight %}

I separated the types instead of nesting them to make sure that instances of Product would be independent.  For this case, this is a fine solution.  However, what happens when you have structures like this all through the JSON?  It is clear there are cases where our data model will be terrible if it mirrors our JSON.

### Flattening Out A JSON Data Model

Another approach to such a problem is to manually take over control of our parsing.  Doing this requires us to parse the JSON ourselves, but we can still leverage Codable to make this happen.

To begin with, we are going to be implementing the required init(from decoder:) initializer from Decodable.  This means we will NOT get the auto-synthesis for free, which will change some basics about our data model:

{% highlight swift %}
class Product : Decodable {
   var name: String?
   var sku: String?
   var price: Double?
   var description: String?
    
   enum CodingKeys : String, CodingKey {
      case elements = "elements"
      case element = "element"
      case name = "name"
      case sku = "sku"
      case price = "price"
      case description = "description"
   }
    
   required init(from decoder: Decoder) throws {
      // TODO
   }
}
{% endhighlight %}

First, notice that we are explicitly conforming to Decodable here.  Because we lost the auto-synthesis, we would need to implement Encodable’s required method and I don’t want to complicate this example with that right now.  

Second, since we lost auto-synthesis we now need to conform to traditional initialization rules.  I chose to make every variable an optional for this purpose.  The init(from decoder:) initializer will place the parsed data in these variables.  

Third, CodingKeys contains all potential keys.  You could nest this structure for additional readability, but for now, I flattened it. 

So, given this, let’s take a detailed look at our implementation of the initializer, one line at  a time.  The general strategy is to use [Decoder protocol][decoder_protocol] to dig all the way into the inner part of our JSON, which we will then extract and initialize our object with.  Note that this code is making the assumption that we always know we only have a single product in the JSON.  

{% highlight swift %}
required init(from decoder: Decoder) throws {
   let elementsContainer = try decoder.container(keyedBy: CodingKeys.self)
}
{% endhighlight %}

We start out by using the container(keyedBy:) method on the Decoder protocol to get a KeyedDecodingContainer.  We use a try before calling this to propagate any possible error up to the caller to deal with, since we can never be guaranteed that our JSON will be valid and have the values we expect.  In this case, we are using a KeyedDecodingContainer because the root dictionary has one element in it with a key of “elements”.  There are three types of containers:

1. **KeyedDecodingContainer** - the data of keyed containers like a dictionary.
2. **SingleValueDecodingContainer** - the data of a container that holds a single primitive.
3. **UnkeyedDecodingContainer** - the data of a container with no keys, such as an array.

Our next step is to get the array of elements:

{% highlight swift %}
required init(from decoder: Decoder) throws {
   let elementsContainer = try decoder.container(keyedBy: CodingKeys.self)
        
   var elementsArrayContainer = try elementsContainer.nestedUnkeyedContainer(forKey: .elements)
        
   if (elementsArrayContainer.count != 1) {
      throw DecodingError.dataCorruptedError(in: elementsArrayContainer, debugDescription: "elements must be an array with one item")
   }
}
{% endhighlight %}

This code gets an **unkeyed** container.  The value of elements in our dictionary is an array of objects, so we can enumerate those with an unkeyed container.  However, before we do that enumeration, we check to be sure that we have exactly one object in the array, else we throw an error.  When digging into a container, there are four methods you will use:

For a **KeyedDecodingContainer** ([Apple documentation][keyed_decoding_container]):

nestedContainer(keyedBy: forKey:) - used when you have the key for the value the container is stored in and the value is keyed container like a dictionary.

nestedUnkeyedContainer(forKey:) - used when you have the key for the value the container is stored in and the value is not keyed, like an array.

For an **UnkeyedDecodingContainer** ([Apple documentation][unkeyed_decoding_container])

nestedContainer(keyedBy:) - used when you want the next container in the current unkeyed container and that container is keyed like a dictionary.

nestedUnkeyedContainer() - used to get the next container in the current unkeyed container, when that container is not keyed, like an array.  

UnkeyedDecodingContainers allow you to iterate them.  Each time you call nestedContainer(keyedBy:) or nestedUnkeyedContainer(), the container increments itself to the next item it stores.  This kind of incrementation can also occur if you are simply decoding values.  Because they iterate like this, they mutate, so we need to store them as var.

Also note that the version of the DecodingError’s dataCorruptedError method we use is the one for an unkeyed container, because elementsArrayContainer is unkeyed.  The compiler will error in a confusing way if you try to use the wrong format.

Thus, the next step in our parser is to iterate the array:

{% highlight swift %}
required init(from decoder: Decoder) throws {
   let elementsContainer = try decoder.container(keyedBy: CodingKeys.self)
        
   var elementsArrayContainer = try elementsContainer.nestedUnkeyedContainer(forKey: .elements)
        
   if (elementsArrayContainer.count != 1) {
      throw DecodingError.dataCorruptedError(in: elementsArrayContainer, debugDescription: "elements must be an array with one item")
   }
        
   while !elementsArrayContainer.isAtEnd {
      let elementDictionaryWrapperContainer = try elementsArrayContainer.nestedContainer(keyedBy: CodingKeys.self)
   }
}
{% endhighlight %}

We use a while loop that checks the condition that the elementsArrayContainer is not at the end.  In our JSON, the array contains a single dictionary.  Based on this, we know we need to get a keyed container for the dictionary, which is elementDictionaryWrappedContainer.  Getting this dictionary also increments our array to the next element, which will cause the while loop to exit since there is only one.  The result is a new KeyedDecodingContainer.  For this one, we need to get the dictionary in the value named with the element key:

{% highlight swift %}
required init(from decoder: Decoder) throws {
   let elementsContainer = try decoder.container(keyedBy: CodingKeys.self)
        
   var elementsArrayContainer = try elementsContainer.nestedUnkeyedContainer(forKey: .elements)
        
   if (elementsArrayContainer.count != 1) {
      throw DecodingError.dataCorruptedError(in: elementsArrayContainer, debugDescription: "elements must be an array with one item")
   }
        
   while !elementsArrayContainer.isAtEnd {
      let elementDictionaryWrapperContainer = try elementsArrayContainer.nestedContainer(keyedBy: CodingKeys.self)
		
      let elementContainer = try elementDictionaryWrapperContainer.nestedContainer(keyedBy: CodingKeys.self, forKey: .element)
	}
}
{% endhighlight %}

Simple enough.  This is another KeyedDecodingContainer, and we request the one with the element CodingKey.  Now we have elementContainer, which includes the values we want to initialize this object with.  Putting it all together:

{% highlight swift %}
required init(from decoder: Decoder) throws {
   let elementsContainer = try decoder.container(keyedBy: CodingKeys.self)
        
   var elementsArrayContainer = try elementsContainer.nestedUnkeyedContainer(forKey: .elements)
        
   if (elementsArrayContainer.count != 1) {
      throw DecodingError.dataCorruptedError(in: elementsArrayContainer, debugDescription: "elements must be an array with one item")
   }
        
   while !elementsArrayContainer.isAtEnd {
      let elementDictionaryWrapperContainer = try elementsArrayContainer.nestedContainer(keyedBy: CodingKeys.self)
		
      let elementContainer = try elementDictionaryWrapperContainer.nestedContainer(keyedBy: CodingKeys.self, forKey: .element)
		
      self.name = try elementContainer.decode(String.self, forKey: .name)
      self.sku = try elementContainer.decode(String.self, forKey: .sku)
      self.price = try elementContainer.decode(Double.self, forKey: .price)
      self.description = try elementContainer.decode(String.self, forKey: .description)
	}
}
{% endhighlight %}

Again, note that every call into the decoding containers uses a try to propagate errors up to the caller.  The decodes are simple, because they are now our foundational types.  As for the code that uses our init, it only changes a little:

{% highlight swift %}
let decoder = JSONDecoder()
let product = try! decoder.decode(Product.self, from: jsonData)
print(product.name!)
// Mr Soapy Bar Soap
{% endhighlight %}

The product is decoded directly from the full JSON that we pass in.  This makes sense, because we are using Product.self as our data model type, instead of Elements.self from our last example.  The data models for Elements and Element are completely avoided.  Notice the use of try!.  Since this is a playground, it is OK to crash if our parsing throws an error.  In reality, we would handle an error like this more elegantly.  Likewise, we use the crash operator to unwrap the optional when printing the name product.  

All of this begs the question, should I do this?  That depends on what you value in an architecture.  I believe the fundamental problem is in the JSON format.  It is making the current call more complicated to solve problems that do not yet exist.  This is generally bad practice.  That said, if you go down a custom road to clean up your data models, you will face the work on decoding (and encoding!) your data model to a sensible format.  When the “cruft” brings no value, such as in the example above, I’m happy to spend the extra resource to clean it up in my codebase.  

### An Inconvenient JSON - Custom Parsing

I would argue that JSON works best when it is a simple document of well-defined key/value types.  As you can see from the above examples, we really need to understand what we are looking for in order to parse it.  Unfortunately, this isn’t always the case either.

Consider the following JSON showing pickup times for a set of SKUs in a store:

{% highlight swift %}
let jsonData = """
{
   "skus": {
      "storePickup": {
         "MrSoapy_Bar_1": {
            "pickupTimes": {
               "2019-12-09T07:00:00.000-0800": {},
               "2019-12-10T07:00:00.000-0800": {}
            }
         },
         "Squeeky_bottle_1": {
            "pickupTimes": {
               "2019-12-09T07:00:00.000-0800": {},
               "2019-12-10T07:00:00.000-0800": {}
            }
         }
      }
   }
}
""".data(using: .utf8)!
{% endhighlight %}

Assume this JSON was returned by a RESTful API where we asked for store pickup times for various SKUs.  There are two uncommon things about this JSON:

First, the value of the “storePickup” element looks like an array, but it is really a dictionary of objects keyed by a sku value that we don’t know ahead of time.  We will have to find a way to dynamically parse those.  

We will assume that the API that returned this JSON is ONLY returning storePickup SKUs, so that part of the JSON is not dynamic and we can parse it directly. 

The second oddity is with “pickupTimes.”  The “pickupTimes” are stored in a dictionary in the same way as the skus in the “storePickup” key’s value, and their values are also keys with data in them.  The values are empty dictionaries for expandability while the keys are dates.  Yikes!

So, we need to start by considering what our data model should look like.  I’m proposing this:

{% highlight swift %}
struct StoreSkus : Decodable {
   let pickupSkus: [StorePickupSkus]
    
   struct StorePickupSkus {
      let sku: String
      let pickupTimes: [Date]
        
      init(sku: String, pickupTimes: [Date]) {
         self.sku = sku
         self.pickupTimes = pickupTimes
      }
   }
}
{% endhighlight %}

So, the main data structure will be StoreSkus, and it will have a property that is an array of StorePickupSkus.  This struct conforms to Codable.  The inner struct is StorePickupSkus.  It contains a String for the sku and an array of Date objects to represent the available pickup times.  It also contains an init method.  Since the properties are not optionals, the init is required.  Also, note that we aren’t implementing Decodable in this struct.  Given that we are doing manual JSON parsing and our data structure can’t match the JSON, it is easiest to do it all in one place since it is small.  

Like with the last example, let’s see what our init(from decoder:) looks like, step by step.

{% highlight swift %}
init(from decoder: Decoder) throws {
   var pickupSkus:[StorePickupSkus] = []
        
   let container = try decoder.container(keyedBy: CodingKeys.self)
   let skusContainer = try container.nestedContainer(keyedBy: CodingKeys.self, forKey: .skus)

   self.pickupSkus = pickupSkus
}
{% endhighlight %}

We start out by creating a mutable version of an array of StorePickupSkus.  We will be building this up as we parse the JSON.  Since this is an initializer, we have to be sure to initialize the property at the end of the initializer with this value we are building up.  Secondly, we need to get the main container.  This is straightforward and we’ve done it before.  Finally, we are going to get a nested container for dictionary contained in the value for the “skus” key.  We also need to add coding keys to the StoreSkus struct, as follows:

{% highlight swift %}
struct StoreSkus : Decodable {
   enum CodingKeys: String, CodingKey {
      case skus
      case storePickup
      case pickupTimes
   }
	
   // ...
}
{% endhighlight %}

Notice that I left off the “= <someString>” part of the keys.  As it turns out, if your string matches the case in the enum, defining the string is not necessary.   
	
Recall when we first looked at the CodingKey protocol how the compiler generates key objects for you based on the “CodingKeys” enum that conforms to CodingKey.  As it turns out, if you need to manipulate the keys themselves, you need to implement a custom CodingKey type:

{% highlight swift %}
struct CustomCodingKey: CodingKey {
   var intValue: Int?
        
   init?(intValue: Int) {
      self.intValue = intValue
      self.stringValue = String(intValue)
   }
        
   var stringValue: String
        
   init?(stringValue: String) {
      self.stringValue = stringValue
   }
}
{% endhighlight %}

This implementation is very simple.  In our case, we need to get named keys from our container, so we are really only concerned with the Int parts of the interface to conform to the protocol.  For the Strings, when a CustomCodingKey is created, we will save the String value so we can access it later.  All that’s left to do is to decode the container with the CustomCodingKey.  

{% highlight swift %}
init(from decoder: Decoder) throws {
   var pickupSkus:[StorePickupSkus] = []
        
   let container = try decoder.container(keyedBy: CodingKeys.self)
   let skusContainer = try container.nestedContainer(keyedBy: CodingKeys.self, forKey: .skus)
	
   let storePickupContainer = try skusContainer.nestedContainer(keyedBy: CustomCodingKey.self, forKey: .storePickup)
   for skuKey in storePickupContainer.allKeys {
      print(skuKey)
   }
	
   self.pickupSkus = pickupSkus
}
// CustomCodingKey(stringValue: "Squeeky_bottle_1", intValue: nil)
// CustomCodingKey(stringValue: "MrSoapy_Bar_1", intValue: nil)
{% endhighlight %}

We create storePickupContainer.  In the sample JSON, the “storePickup” key has a dictionary with two keys in it.  Those keys are named for the sku their value’s describe (also as dictionaries).  The nestedContainer(keyedBy type:, forKey key:) method takes two parameters.  The type is the key type to use with the container, which is our custom type (remember, CustomCodingKey.self is like calling the class method in Objective-C).  The key is the actual key the container is associated with, which is an instance of an object that conforms to CodingKey that references the “storePickup” String like in the JSON (phew!).  

To iterate this, the container has an allKeys accessor we can use to get an array of our CustomCodingKey instances.  If you run the decoder:

{% highlight swift %}
let decoder = JSONDecoder()
let pickupSkus = try! decoder.decode(StoreSkus.self, from: jsonData)
{% endhighlight %}

….you should see print output the two instances of CustomCodingKey.  

The next step is to get the nested container which are named with the skus themselves.  Because we have objects in our JSON that are keyed by a non-dyanmic string “pickupTimes”, we can use the CodingKeys to get at the container we want.  Within that container, we have a list of dates stored as key names.  We need to get those dates for our data model.  This works just like the example above, and we can reuse the CustomCodingKey struct.  

{% highlight swift %}
init(from decoder: Decoder) throws {
   var pickupSkus:[StorePickupSkus] = []
        
   let container = try decoder.container(keyedBy: CodingKeys.self)
   let skusContainer = try container.nestedContainer(keyedBy: CodingKeys.self, forKey: .skus)

   let pickupTimesContainer = try skuContainer.nestedContainer(keyedBy: CustomCodingKey.self, forKey: .pickupTimes)
}
{% endhighlight %}

It is worth noting that we have a second option to accomplish this.  Consider the following code:

{% highlight swift %}
let skuContainer = try storePickupContainer.nestedContainer(keyedBy: CustomCodingKey.self, forKey: skuKey)
let pickupTimesContainer = try skuContainer.nestedContainer(keyedBy: CustomCodingKey.self, forKey: CustomCodingKey(stringValue:"pickupTimes")!)
{% endhighlight %}

In this case, when we retrieve the skuContainer, we use the CustomCodingKey type.  That means in order to get the pickupTimesContainer, we have to create an instance of CustomCodeKey initialized with the “pickupTimes” String we want to use as the key.  This works as well.  The type of custom key used to create the container is the type of key the system expects you to use to build the next container.  My preference is to use the first example for simplicity, especially when you consider we probably shouldn’t force unwrap the CustomCodingKey initialization, which gives us extra error stuff to handle. 

For the next step, we are going to create an empty mutable array of Date objects.  Eventually, this will store the dates we parse from the JSON.  For now, it is a placeholder.  This iteration uses the allKeys accessor on the pickupTimesContainer, just like we saw above.  If you print the key, you can see the code is iterating through both pairs of objects.

{% highlight swift %}
init(from decoder: Decoder) throws {
   var pickupSkus:[StorePickupSkus] = []
        
   let container = try decoder.container(keyedBy: CodingKeys.self)
   let skusContainer = try container.nestedContainer(keyedBy: CodingKeys.self, forKey: .skus)
	
   let pickupTimesContainer = try skuContainer.nestedContainer(keyedBy: CustomCodingKey.self, forKey: .pickupTimes)
	
   var pickupTimes:[Date] = []

   for pickupTimeKey in pickupTimesContainer.allKeys {
      print(pickupTimeKey)
   }
}
// CustomCodingKey(stringValue: "2019-12-09T07:00:00.000-0800", intValue: nil)
// CustomCodingKey(stringValue: "2019-12-10T07:00:00.000-0800", intValue: nil)
// CustomCodingKey(stringValue: "2019-12-09T07:00:00.000-0800", intValue: nil)
// CustomCodingKey(stringValue: "2019-12-10T07:00:00.000-0800", intValue: nil)
{% endhighlight %}

Next, we want to use DateFormatter to convert the date string into a Date object.

{% highlight swift %}
var pickupTimes:[Date] = []

for pickupTimeKey in pickupTimesContainer.allKeys {
   let dateFormatter = DateFormatter()
   dateFormatter.locale = Locale(identifier: "en_US_POSIX")
   dateFormatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZZZZZ"
   let date = dateFormatter.date(from: pickupTimeKey.stringValue)
	
   if (date != nil) {
      pickupTimes.append(date!)  
   } else {
      throw DecodingError.dataCorruptedError(forKey: pickupTimeKey, in: pickupTimesContainer, debugDescription: "Invalid date format")
   }
}
{% endhighlight %}

We simply use an instance of DateFormatter to convert the String into a Date object.  To pass the date String to the date formatter, we use the stringValue accessor on the pickupTimeKey instance of CustomCodingKey to access the String part of the key.  This gives us a Date optional, which we either add to the mutable array, or we throw an error if the format is not what we expect.  When we create the DecodingError, we are using a format of the dataCorruptedError method that works with a keyed collection.  

Finally, we have the data we need to construct our StorePickupSkus object:

{% highlight swift %}
let storePickupSku = StorePickupSkus(sku: skuKey.stringValue, pickupTimes: pickupTimes)
pickupSkus.append(storePickupSku)
{% endhighlight %}

Again, we use the stringValue accessor on our CustomCodingKey class to access the key’s String object and initialize using our custom initializer.  We then add this object to the pickupSkus mutable array we added to the top of the method.

Putting it all together, we end up with an initializer that looks like this:

{% highlight swift %}
init(from decoder: Decoder) throws {
   var pickupSkus:[StorePickupSkus] = []

   let container = try decoder.container(keyedBy: CodingKeys.self)
   let skusContainer = try container.nestedContainer(keyedBy: CodingKeys.self, forKey: .skus)

   let storePickupContainer = try skusContainer.nestedContainer(keyedBy: CustomCodingKey.self, forKey: .storePickup)

   for skuKey in storePickupContainer.allKeys {
      let skuContainer = try storePickupContainer.nestedContainer(keyedBy: CodingKeys.self, forKey: skuKey)

      let pickupTimesContainer = try skuContainer.nestedContainer(keyedBy: CustomCodingKey.self, forKey: .pickupTimes)

      var pickupTimes:[Date] = []

      for pickupTimeKey in pickupTimesContainer.allKeys {
         let dateFormatter = DateFormatter()
         dateFormatter.locale = Locale(identifier: "en_US_POSIX")
         dateFormatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZZZZZ"
         let date = dateFormatter.date(from: pickupTimeKey.stringValue)
         if (date != nil) {
            pickupTimes.append(date!)
         } else {
            // Throw error
            throw DecodingError.dataCorruptedError(forKey: pickupTimeKey, in: pickupTimesContainer, debugDescription: "Invalid date format")
         }
      }

      let storePickupSku = StorePickupSkus(sku: skuKey.stringValue, pickupTimes: pickupTimes)
      pickupSkus.append(storePickupSku)
   }

   self.pickupSkus = pickupSkus
}
{% endhighlight %}

If we print the output of the instance of pickupSkus we created, we can see this works.

{% highlight swift %}
let decoder = JSONDecoder()
let pickupSkus = try! decoder.decode(StoreSkus.self, from: jsonData)

print(pickupSkus)
// StoreSkus(pickupSkus: [__lldb_expr_3.StoreSkus.StorePickupSkus(sku: "MrSoapy_Bar_1", pickupTimes: [2019-12-09 15:00:00 +0000, 2019-12-10 15:00:00 +0000]), __lldb_expr_3.StoreSkus.StorePickupSkus(sku: "Squeeky_bottle_1", pickupTimes: [2019-12-09 15:00:00 +0000, 2019-12-10 15:00:00 +0000])])
{% endhighlight %}

Should you have to do stuff like this?  Hopefully not.  The example presented here is overly complicated with the intent of providing future flexibility.  While understandable why an API writer might want such flexibility, strongly typed languages such as Swift really make it a bad idea.  The data format belonging to your API should be treated as a contract much like your API.  Some dynamism can make things easier for both your clients and your back-end, but it is easy to go too far.  JSON works best as key-value pairs, and your dynamism should respect this to avoid the extra theatrics shown here.  

### Parsing an Unkeyed Array of Dates

Extending a prior concept, consider this simple JSON that contains a list of date strings:

{% highlight swift %}
let jsonData = """
{
   "dates": [
      "2019-12-09T07:00:00.000-0800",
      "2019-12-10T07:00:00.000-0800",
      "2019-12-11T07:00:00.000-0800"
      ]
}
""".data(using: .utf8)!
{% endhighlight %}

We can parse this manually with an unkeyed container. 

{% highlight swift %}
struct DateList : Decodable {
   enum CodingKeys: String, CodingKey {
      case dates
   }
    
   var dates:[Date]
    
   init(from decoder: Decoder) throws {
      var dates:[Date] = []
        
      let container = try decoder.container(keyedBy: CodingKeys.self)
      var datesContainer = try container.nestedUnkeyedContainer(forKey: .dates)
        
      while !datesContainer.isAtEnd {
         let dateFormatter = DateFormatter()
         dateFormatter.locale = Locale(identifier: "en_US_POSIX")
         dateFormatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZZZZZ"
         let dateString = try datesContainer.decode(String.self)
         let date = dateFormatter.date(from: dateString)
         if (date != nil) {
            dates.append(date!)
         } else {
            // Throw error
            throw DecodingError.dataCorruptedError(in: datesContainer, debugDescription: "Invalid date format")
         }
      }
        
      self.dates = dates
   }
}

let decoder = JSONDecoder()
let dates = try! decoder.decode(DateList.self, from: jsonData)
print(dates)
// DateList(dates: [2019-12-09 15:00:00 +0000, 2019-12-10 15:00:00 +0000, 2019-12-11 15:00:00 +0000])
{% endhighlight %}

This is nothing we haven’t seen before, it is just presented slightly differently.  The array doesn’t contain other dictionaries, it contains the data directly.  We use an unkeyed container to iterate over these values.  To pull the data in the current item we are iteration, we simply decode it:

{% highlight swift %}
let dateString = try datesContainer.decode(String.self)
{% endhighlight %}

This can result in an error, so we use a try so the error gets passed up to the caller.  The rest is just building the Date object via a DataFormatter, adding it to a mutable Date array and assigning it to the member property. 

That said, we can actually accomplish this much easier and automatically. 

{% highlight swift %}
struct DateList : Decodable {
   enum CodingKeys: String, CodingKey {
      case dates
   }
    
   var dates:[Date]
}

let decoder = JSONDecoder()

let dateFormatter = DateFormatter()
dateFormatter.locale = Locale(identifier: "en_US_POSIX")
dateFormatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZZZZZ"

decoder.dateDecodingStrategy = .formatted(dateFormatter)

let dates = try! decoder.decode(DateList.self, from: jsonData)
print(dates)
// DateList(dates: [2019-12-09 15:00:00 +0000, 2019-12-10 15:00:00 +0000, 2019-12-11 15:00:00 +0000])
{% endhighlight %}

Because we are using the same date decoding strategy throughout the JSON, we do not need to customize our JSON parsing.  We can set the dateDecodingStrategy in the JSONDecoder to use our DateFormatter, and get rid of the initialization method and the custom parsing.  DateDecodingStrategy provides secondsSince1970, millsecondsSince1970 and ios8601 formats by default.  If you are using those, you don’t even need a custom DateFormatter. 

### Summary

We’ve covered much ground and several examples that should show you how to piece together various techniques to parse most kinds of JSON into data models.  There are a myriad of strategies you can use to parse JSON in Swift I haven’t shown here.  Most of the difficulty will result from how difficult the JSON you have is to deal with.   I encourage you to work with your API writers on JSON standards that are easy to deal with AND reflect what your data model should look like. 

[json-encoder]: https://github.com/apple/swift/blob/master/stdlib/public/Darwin/Foundation/JSONEncoder.swift
[json-decoder-docs]: https://developer.apple.com/documentation/foundation/jsondecoder
[string_protocol_docs]: https://developer.apple.com/documentation/swift/stringprotocol
[swift_codable]: https://github.com/apple/swift/blob/master/stdlib/public/core/Codable.swift
[swift_decodable_init_docs]: https://developer.apple.com/documentation/swift/decodable/2894081-init
[swift_coding_key]: https://github.com/apple/swift-evolution/blob/master/proposals/0166-swift-archival-serialization.md
[wwdc_valuetypes_vs_referencetypes]:https://developer.apple.com/videos/play/wwdc2015/414/
[decoder_protocol]: https://developer.apple.com/documentation/swift/decoder
[keyed_decoding_container]: https://developer.apple.com/documentation/swift/keyeddecodingcontainerprotocol
[unkeyed_decoding_container]: https://developer.apple.com/documentation/swift/unkeyeddecodingcontainer
[json_spec]: https://www.json.org/json-en.html