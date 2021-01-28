# Introduction to Thoth.Json.Net

## Sample: Parsing an Order

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
          "address1": "1234 Main St.",
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

### Decode!
```fsharp
match orderJson |> Decode.fromString orderDecoder with
| Ok order -> printfn "Order: %A" order
| Error err -> printfn "Error: %s" err
```

### Output
```
Order:
{ AccountName = "accountName"
  OrderId = "1234512345"
  Items = [{ Sku = "Business Cards" }]
  Shipments = [{ Recipient = Some "John Doe"
                 Address = "1234 Main St."
                 Town = "Capitol"
                 PostCode = "12345" }] }
```
