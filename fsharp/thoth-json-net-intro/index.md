# Introduction to Thoth.Json.Net

Thoth.Json.Net has become the first tool I reach for when I need to map incoming json (that I did not author) to types in F#. 

However, the first time I tried to use it, I stumbled because I had never encountered a library like it in all my days as a C# developer. So I wanted to create this quick intro to provide some common examples to get you started (or to refresh your memory). 

## Why Not Use Newtonsoft.Json? 
Newtonsoft is great if you already have a model that closely matches the structure of the incoming json. But if you don't, it can be cumbersome to create mappings via attributes and weird adapter classes. Furthermore, you may not want to litter your domain entities with parsing attributes. 
The typical use case that I like to use Thoth for is when I need to deserialize json from an external source that I didn't create. 

This is where Thoth.Json.Net shines. It allows you to quickly and easily create json "decoder" functions to map your incoming json to your types. 

## Parsing an Order
We will be parsing the first order json sample that appeared in my search results. 
This example is good because it shows how to create decoders for "one-to-many" data structures.

### Sample Order json

```fsharp
let orderJson = """
{
  "destination": {
    "name": "accountName"
  },
  "orderData": {
    "sourceOrderId": "1234512345",
    "items": [
      {
        "sku": "Business Cards",
        "sourceItemId": "1234512346",
        "components": [
          {
            "code": "Content",            
            "fetch": true,
            "path": "http://www.w2psite.com/businessCard.pdf"
          }
        ]
      }
    ],
    "shipments": [
      {
        "shipTo": {
          "name": "John Doe",
          "companyName": "Acme",
          "address1": "1234 Main St",
          "town": "Capitol",
          "postcode": "12345",
          "isoCountry": "US"
        },
        "carrier":{
          "code": "fedex",
          "service": "ground"
        }
      }
    ]
  }
}
"""
```

### Create Order Types

First we will need to create some domain types to decode to. 
Notice that our entities are not a one-to-one mirror of the json structure.  Instead, we will flatten the structure in some areas to simplify our domain model.

```fsharp
type Order = {
    AccountName: string
    OrderId: string
    Items: OrderItem list
    Shipments: Shipment list
}
and OrderItem = {
    Sku: string
}
and Shipment = {
    Recipient: string option
    Address: string
    Town: string
    PostCode: string
}
```

### Create Thoth "Decoders"

Next we will create a Thoth "decoder" function for each entity. 
Here are all three decoders (we will look at each one individually in the next section):

```fsharp
#r "nuget: Thoth.Json.Net"
open Thoth.Json.Net

let orderItemDecoder : Decoder<OrderItem> =
    Decode.object (
        fun get ->
            { OrderItem.Sku = get.Required.Field "sku" Decode.string }
    )

let shipmentDecoder : Decoder<Shipment> = 
    Decode.object (
        fun get -> 
            { Shipment.Recipient = get.Optional.At ["shipTo"; "name"] Decode.string
              Shipment.Address = get.Required.At ["shipTo"; "address1"] Decode.string
              Shipment.Town = get.Required.At ["shipTo"; "town"] Decode.string
              Shipment.PostCode = get.Required.At ["shipTo"; "postcode"] Decode.string }
    )

let orderDecoder : Decoder<Order> = 
    Decode.object (
        fun get ->
            { Order.AccountName = get.Required.At ["destination"; "name"] Decode.string
              Order.OrderId = get.Required.At ["orderData"; "sourceOrderId"] Decode.string
              Order.Items = get.Required.At ["orderData"; "items"] (Decode.list orderItemDecoder)
              Order.Shipments = get.Required.At ["orderData"; "shipments"] (Decode.list shipmentDecoder) }
    )

```

The root entity is the order, so we will create an `orderDecoder` function at the bottom so that it can reference the child decoders.

At the very top we can create our `orderItemDecoder` and `shipmentDecoder` functions that will be used by the `orderDecoder`.

#### Order Item Decoder

```fsharp
let orderItemDecoder : Decoder<OrderItem> =
    Decode.object (
        fun get ->
            { OrderItem.Sku = get.Required.Field "sku" Decode.string }
    )
```

The `orderItemDecoder` is the most simple. This function will decode each item in the "orderData/items" array.
It has a single required field (that means we expect it to always exist in the json). The last parameter, `Decode.string`, is a built-in primitive decoder that will attempt to parse the json field as a `string`. 
You don't have to annotate `orderItemDecoder` with `: Decoder<OrderItem>`, but it can be helpful to add some strong typing while you are creating it.

#### Shipment Decoder

```fsharp
let shipmentDecoder : Decoder<Shipment> = 
    Decode.object (
        fun get -> 
            { Shipment.Recipient = get.Optional.At ["shipTo"; "name"] Decode.string
              Shipment.Address = get.Required.At ["shipTo"; "address1"] Decode.string
              Shipment.Town = get.Required.At ["shipTo"; "town"] Decode.string
              Shipment.PostCode = get.Required.At ["shipTo"; "postcode"] Decode.string }
    )
```

Moving on to the `shipmentDecoder`, notice that instead of using `get.Required.Field`, we have switched to `get.Required.At`.  The difference is that `get.Required.At` takes a list of strings which allows you to drill into a path of multiple levels, whereas `get.Required.Field` just takes a single string that references a field at the current level.
The second thing to notice is that `Shipment.Recipient` uses `get.Optional` instead of `get.Required`. This will return an `option` value, so you must make sure that you are mapping the value into an `option` field. 

#### Order Decoder

```fsharp
let orderDecoder : Decoder<Order> = 
    Decode.object (
        fun get ->
            { Order.AccountName = get.Required.At ["destination"; "name"] Decode.string
              Order.OrderId = get.Required.At ["orderData"; "sourceOrderId"] Decode.string
              Order.Items = get.Required.At ["orderData"; "items"] (Decode.list orderItemDecoder)
              Order.Shipments = get.Required.At ["orderData"; "shipments"] (Decode.list shipmentDecoder) }
    )
```

Finally, we come to the root `orderDecoder`. The most important detail to notice here is that we can populate the `Order.Items` and `Order.Shipments` properties by using the `Decode.list` function in conjunction with our custom decoders above: `(Decode.list orderItemDecoder)` and `(Decode.list shipmentDecoder)`.


### Decode!

Now that we have created our decoders, we can decode our json string.

```fsharp
match orderJson |> Decode.fromString orderDecoder with
| Ok order -> printfn $"Order: {order}"
| Error err -> printfn $"Error: {err}"
```

### Output
```
Order:
{ AccountName = "accountName"
  OrderId = "1234512345"
  Items = [{ Sku = "Business Cards" }]
  Shipments = [{ Recipient = Some "John Doe"
                 Address = "1234 Main St"
                 Town = "Capitol"
                 PostCode = "12345" }] }
```

### Diagnosing Errors
One huge benefit of using Thoth.Json.Net is that the `Decode.fromString` function returns a `Result` with either your decoded payload or a very helpful error message.

For example, if the json is missing the "sku" field that is marked as `Required` in our decoder, we will get a very informative error message:
```
Error at: `$.orderData.items.[0]`
Expecting an object with a field named `sku` but instead got:
{
    "sourceItemId": "1234512346",
    "components": [
        {
            "code": "Content",
            "fetch": true,
            "path": "http://www.w2psite.com/businessCard.pdf"
        }
    ]
}
```

## Anonymous Records
You can also decode directly to an anonymous record!

```fsharp
let json = """ { "FName": "John", "LName": "Doe" } """

let personDecoder = 
    Decode.object <|
        fun get -> 
            {| FirstName = get.Required.Field "FName" Decode.string
               LastName = get.Required.Field "LName" Decode.string |}    

match json |> Decode.fromString personDecoder with
| Ok person -> printfn $"Person: {person}"
| Error err -> printfn $"Error: {err}"
```

This yields the following output:
```
Person:
{ FirstName = "John"
  LastName = "Doe" }
```

## Conclusion
Anytime you need to "cherry-pick" data from a json structure to be custom mapped into your own types, you should reach for Thoth. 

Thanks to Maxime Mangel for creating this library. 
Visit his official page here for documentation. Maxime has been very gracious and quick to help when I've had questions.
[Thoth.Json.Net Official Page](https://thoth-org.github.io/Thoth.Json/)


