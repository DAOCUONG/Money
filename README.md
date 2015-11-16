![](header.png)

[![Build status](https://badge.buildkite.com/265eb9670a2ef6b73eebf37769a8455c402509f71f09c4f51e.svg?branch=development)](https://buildkite.com/blindingskies/money?branch=development)
[![codecov.io](https://codecov.io/github/danthorpe/Money/coverage.svg?branch=development&token=gI70muNOjA)](https://codecov.io/github/danthorpe/Money?branch=development)
[![Cocoapods Compatible](https://img.shields.io/cocoapods/v/Money.svg)](https://img.shields.io/cocoapods/v/Money.svg)
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![Platform](https://img.shields.io/cocoapods/p/Money.svg?style=flat)](http://cocoadocs.org/docsets/Money)

# Money

Money is a Swift framework for iOS, watchOS, tvOS and OS X. It provides types and functionality to represent, calculate and convert money in the 298 [ISO currencies](https://en.wikipedia.org/wiki/ISO_4217). 

## Usage

The Money framework defines the type `Money`, which represents money in the device’s current locale. The following code:

```swift
import Money

let money: Money = 100
print("I'll give \(money) to charity.”)
```

will print out

> I'll give $ 100.00 to charity 

when the region is set to United States

> I'll give £ 100.00 to charity

when the region is set to United Kingdom

> I'll give CN¥ 100.00 to charity

when the region is set to China

You get the idea.

`Money` is `IntegerLiteralConvertible` and  `FloatLiteralConvertible`. Which means values can be initialized using literal `Int`s and `Double`s as shown in these code snippets.

## Specific Currency

Under the hood, `Money` is a `typealias` for `_Money<Currency.Local>` where `Currency.Local` is a specific `CurrencyType` which represents the currency for the current locale. This means that it is strongly typed to the local currency.

In a similar way, there are 298 foreign currency types supported.

```swift
let pounds: GBP = 99.99
let euros: EUR = 149.50

print(“You have \(pounds / 2) and \(euros + 30)”)
```

> You have £ 50.00 and € 179.50

Because the currencies are typed, it means that they cannot be combined together.

```swift
let money = pounds + euros
```
> Binary operator '+' cannot be applied to operands of type 'GBP' (aka '_Money&lt;Currency.GBP&gt;') and 'EUR' (aka '_Money&lt;Currency.EUR&gt;')

Of course, `Money` supports the usual suspects of decimal arithmetic operators, so you can add, subtract, multiply, divide values of the same type, and values with `Int` and `Double` with the expected limitations.

## Minor Units

`Money` can be initialized using the smallest units of currency:

```swift
let dollars = USD(minorUnits: 3250)
let yuen = JPY(minorUnits: 3000)
let bitcoin = BTC(minorUnits: 5000)

print(“You have \(dollars), \(yuen) and \(bitcoin) Satoshis”)
```

> You have $ 32.50, JP¥ 3,000 and 0.00005 Satoshis

## Foreign Currency Exchange (FX)
To represent a foreign exchange transaction, i.e. converting `USD` to `EUR`, use a FX service provider. There is built in support for [Yahoo](https://finance.yahoo.com/currency-converter/#from=USD;to=EUR;amt=1) and [OpenExchangeRates.org](https://openexchangerates.org) services. But it’s possible for consumers to create their own too.

The following code snippet represents a currency exchange using Yahoo’s currency converter.

```swift
Yahoo<USD,EUR>.quote(100) { result in
    if let tx = result.value {
        print("Exchanged \(tx.base) into \(tx.counter) with a rate of \(tx.rate) and \(tx.commission) commission.")
    }
}
```

> Exchanged US$ 100.00 into € 93.09 with a rate of 0.93089 and US$ 0.00 commission.

The result, delivered asynchronously, uses [`Result`](http://github.com/antitypical/Result) to encapsulate either a `FXTransaction` or an `FXError` value. Obviously, in real code - you’d need to check for errors ;)

`FXTransaction` is a generic type which composes the base and counter monies, the rate of the exchange, and any commission the FX service provider charged in the base currency. Currently `FXQuote` only supports percentage based commission.

There is a neat convenience function which just returns the `CounterMoney` as its `Result` value type.

```swift
Yahoo<USD,EUR>.fx(100) { euros in
    print("You got \(euros)")
}
```

> You got .Success(€ 93.09)

### Creating custom FX service providers

Creating a custom FX service provider is straightforward. The protocols `FXLocalProviderType` and `FXRemoteProviderType` define the minimum requirements. The `quote` and `fx` methods are provided via extensions on the protocols.

For a remote FX service provider, i.e. one which will make a network request to get a rate, we can look at the `Yahoo` provider to see how it works.

Firstly, we subclass the generic class `FXRemoteProvider`. The generic types are both constrained to `MoneyType`. The naming conventions follow those of a [currency pair](https://en.wikipedia.org/wiki/Currency_pair).

```swift
public class Yahoo<B: MoneyType, C: MoneyType>: FXRemoteProvider<B, C>, FXRemoteProviderType {
    // etc
}
```

`FXRemoteProvider` provides the typealiases for `BaseMoney` and `CounterMoney` which will be needed to introspect the currency codes.

The protocol requires that we can construct a `NSURLRequest`.

```swift
public static func request() -> NSURLRequest {
  return NSURLRequest(URL: NSURL(string: "https://download.finance.yahoo.com/d/quotes.csv?s=\(BaseMoney.Currency.code)\(CounterMoney.Currency.code)=X&f=nl1")!)
}
```

The last requirement, is that the network result can be mapped into a `Result<FXQuote,FXError>`.

`FXQuote` is a struct, which composes the exchange rate and percentage commission to be used. Both properties are `BankersDecimal` values (see below on the decimal implementation details).

```swift
public static func quoteFromNetworkResult(result: Result<(NSData?, NSURLResponse?), NSError>) -> Result<FXQuote, FXError> {
  return result.analysis(
    ifSuccess: { data, response in
      let rate: BankersDecimal = 1.5 // or whatever	 
      return Result(value: FXQuote(rate: BankersDecimal(floatLiteral: rate)))
    },
    ifFailure: { error in
      return Result(error: .NetworkError(error))
    }
  )
}
```

Note that the provider doesn’t need to perform any networking itself. It is all done by the framework. This is a deliberate architectural design as it makes it much easier to unit test the adaptor code.

## Bitcoin

Money has support for Bitcoin types, the popular `BTC` and the unofficial ISO 4217 currency code `XBT`.

In [November 2015](http://www.coindesk.com/bitcoin-unicode-symbol-approval/), the Unicode consortium accepted U+20BF as the Bitcoin symbol. However, right now that means it is not available in Foundation. Therefore, currently the Bitcoin currency type(s) use Ƀ, which is also a popular symbol and available already within Unicode.

To work with Bitcoin, use the following:

```swift
let bitcoin: BTC = 0.1234_5678
print(“You have \(bitcoin)”)
```
> You have Ƀ0.12345678

Money has support for using [CEX.IO](https://cex.io)’s [trade api](https://cex.io/api) to support quotes of Bitcoin currency exchanges. CEX only supports `USD`, `EUR,` and `RUB` [fiat currencies](https://en.wikipedia.org/wiki/Fiat_money). 

It’s usage is a little bit different for a regular FX. To represent the purchase of Bitcoins use `CEXBuy` like this:

```swift
CEXBuy<USD>.quote(100) { result in
    if let tx = result.value {
        print("\(tx.base) will buy \(tx.counter) at a rate of \(tx.rate) with \(tx.commission)")
    }
}
```
> US$ 100.00 will buy Ƀ0.26219275 at a rate of 0.0026272 with US$ 0.20 commission.

To represent the sale of Bitcoins use `CEXSell` like this:

```swift
CEXSell<EUR>.quote(50) { result in
    if let tx = result.value {
        print("\(tx.base) will sell for \(tx.counter) at a rate of \(tx.rate) with \(tx.commission) commission.")
    }
}
```
> Ƀ50.00 will sell for € 17,541.87 at a rate of 351.5405 with Ƀ0.10 commission.

If trying to buy or sell using a currency not supported by CEX the compiler will prevent your code from compiling.

```swift
CEXSell<GBP>.quote(50) { result in
    // etc
}
```
> Type 'Currency.GBP' does not conform to protocol 'CEXSupportedFiatCurrencyType'

# Creating custom currencies

If your app has its own currency e.g. ⭐️s or 💎s or even 🐝s, you might want to consider making a type for it.

Lets imagine we’re making *Hive.app* - where you compete with your friends to see who can get the biggest hive (measured in number of 🐝s).

To create a custom currency, just conform to `CurrencyType`. 

```swift
protocol HiveCurrencyType: CurrencyType { }

extension Currency {
    final class Bee: HiveCurrencyType {

        static let code: String = "BEES"
        static let symbol: String = "🐝"
        static let scale: Int  = 0
        static let formatter: NSNumberFormatter = {
            let fmtr = NSNumberFormatter()
            fmtr.numberStyle = .CurrencyStyle
            fmtr.maximumFractionDigits = Currency.Bee.scale
            fmtr.currencySymbol = Currency.Bee.symbol
            return fmtr
        }()
    }
}

typealias Bees = _Money<Currency.Bee>
```

Just make sure that your currency code doesn’t clash with a real one - make it more than three characters to be sure.

Now it’s possible to work with your own app’s currency as a proper money type.

```swift
let bees: Bees = 10_000
print(“I have \(bees)”)
```
> I have 🐝10,000

And of course if you have an IAP for purchasing in-app currency, then I’m sure a custom FX provider would be handy.

Take a look at the example project, Custom Money, for an example of a custom local FX provider to exchange your 🐝s.

## Installation
Money builds as a cross platform (iOS, OS X, watchOS) extension compatible framework. It is compatible with [Carthage](https://github.com/carthage/carthage). It is also available via CocoaPods.

```ruby
pod ‘Money’
```

At of writing there are some issues with the CocoaDocs generator for pure Swift 2 projects. This means that the project doesn’t have a page/docs in CocoaPods sites, however they are available through Xcode. 

## Architectural style
Swift is designed to have a focus on safety, enabled primarily through strong typing. This framework fully embraces this ethos and uses generics heavily to achieve this goal. 

At the highest level *currency* is modeled as a protocol, `CurrencyType`. The protocol defines a few static properties like its symbol, and currency code. Therefore *money* is represented as a decimal number with a generic currency. Additionally, we make `CurrencyType` refine the protocol which defines how the decimal number behaves.

Finally, we auto-generate the code which defines all the currencies and money typealiases.

## Implementation Details

Cocoa has two type which can perform decimal arithmetic, these are `NSDecimalNumber` and `NSDecimal`. `NSDecimal` is faster, but is trickier to work with, and doesn’t have support for limiting the scale of the numbers (which is pretty important when working with currencies).

`DecimalNumberType` is a protocol which refines `SignedNumberType` and defines its own functions, `add`, `subtract` etc to support the arithmetic. It is generic over two types, the underlying storage, and the behaviors.

`DecimalNumberType.DecimalStorageType` exists so that conforming types can utilize either `NSDecimalNumber` or `NSDecimal` as their underling storage type.

`DecimalNumberBehavior` is a protocol which exposes a  [`NSDecimalNumberBehaviors`](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Protocols/NSDecimalNumberBehaviors_Protocol/) which should be used in calculations. This includes rounding style, scale, and when to throw exceptions.

### `_Decimal`

Which leads us to `_Decimal<Behavior: DecimalNumberBehavior>` which is a value type implementing `DecimalNumberType` with an `NSDecimalNumber` storage type.

There are two public typealiases for convenience.

```swift
/// `Decimal` with plain decimal number behavior
public typealias Decimal = _Decimal<DecimalNumberBehavior.Plain>

/// `BankersDecimal` with banking decimal number behavior
public typealias BankersDecimal = _Decimal<DecimalNumberBehavior.Bankers>
```

This means, that `Decimal` is more than likely the type to use for most things.

### `_Money`
The `_Money<C: CurrencyType>` type composes a `_Decimal<C>`. Its behavior is provided via its generic `CurrencyType` which refines `DecimalNumberBehavior`. `_Money` also conforms to `DecimalNumberType` which means that it can also be used with the operators.

### Why not use `NSDecimal`?
`NSDecimal` would be a better storage type for `_Decimal`, however it doesn’t have the full `NSDecimalNumberBehaviors` support that `NSDecimalNumber` enjoys. In particular, specifying the scale is problematic. If anyone has any smart ideas, please get in touch. I’ve added an equivalent extension on `NSDecimal` as for `NSDecimalNumber`.

### `ValueCoding`
Both `_Decimal`, `_Money` and `FXTransaction` all conform to [`ValueCoding`](https://github.com/danthorpe/ValueCoding) which means they can be encoded and stored inside archives.


## Author
Daniel Thorpe [@danthorpe](https://twitter.com/danthorpe). 

Feel free to get in contact if you have questions, queries, suggestions, or need help. Especially get in contact via an Issue here or on Twitter if you want to add support for another FX service provider.

I wrote an introductory blog post about money [here](http://danthorpe.me/posts/money.html).

## License

Money is available under the MIT license. See the LICENSE file for more info.

## Disclaimer

Usage of this framework prevents the author, Daniel Thorpe, from being held liable for any losses incurred by the user through their use of the framework.
