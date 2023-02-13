---
layout: post
title:  DynamicTypeSize for UIKit
subtitle: Building a SwiftUI-style modifier for UIView
date:   2023-02-11 02:02:40 -0600
tags:   [uikit]
---

When going from SwiftUI back to UIKit, I miss having certain view modifiers, such as`.dynamicTypeSize()`. In SwiftUI, we can limit the accessibility behavior of a View by setting a value or range of `DynamicTypeSize`. This approach is expressive and safe, since the compiler can check for valid ranges. To apply the same limits to UIView, we set single values of `UIContentSizeCategory` for `.minimumContentSizeCategory` and `.maximumContentSizeCategory`, which requires checking values by hand. The ability to set a range of sizes is very convenient, so let's bring this little piece of SwiftUI style to UIKit.

### Swift Ranges and the Comparable Protocol
The various range types (`Range`, `ClosedRange`, etc.) all conform to the `RangeExpression` protocol, which requires the underlying values to conform to the `Comparable` protocol.

````swift
public protocol RangeExpression<Bound> {
    associatedtype Bound : Comparable
}
````

Since Swift 5.3, comparable conformance can be synthesized for most enums. In SwiftUI, DynamicTypeSize is declared as a conforming enum and the values are available for use in range expressions. The View type has two definitions for a dynamicTypeSize modifier (one for a single value and one for a range of values).

````swift
public func dynamicTypeSize(_ size: DynamicTypeSize) -> some View
public func dynamicTypeSize<T>(_ range: T) -> some View where T : RangeExpression, T.Bound == DynamicTypeSize
````

### Constructing UIContentSizeCategory Ranges
In UIKit, UIContentSizeCategory is declared as a struct without any comparable conformance, so the values cannot be used directly to construct a range expression. Fortunately, it is fairly easy to provide comparable conformance for UIContentSizeCategory, allowing use in ranges. We can extend UIContentSizeCategory to include the two operators necessary for conformance and an array of sizes to enable comparison of the values.

````swift
extension UIContentSizeCategory: Comparable {
    public static func <(lhs: UIContentSizeCategory, rhs: UIContentSizeCategory) -> Bool {
        let leftIndex = orderedSizes.firstIndex(of: lhs) ?? 0
        let rightIndex = orderedSizes.firstIndex(of: rhs) ?? 0
        return leftIndex < rightIndex
    }
  
    public static func ==(lhs: UIContentSizeCategory, rhs: UIContentSizeCategory) -> Bool {
        let leftIndex = orderedSizes.firstIndex(of: lhs) ?? 0
        let rightIndex = orderedSizes.firstIndex(of: rhs) ?? 0
        return leftIndex == rightIndex
    }
  
    static let orderedSizes: [UIContentSizeCategory] =
    [.unspecified, .extraSmall, .small, .medium, .large, .extraLarge, .extraExtraLarge, .extraExtraExtraLarge,
    .accessibilityMedium, .accessibilityLarge, .accessibilityExtraLarge, .accessibilityExtraExtraLarge, .accessibilityExtraExtraExtraLarge]
}
````

All that's left is to extend UIView with functions that mimic SwiftUI's View modifiers. In the second definition, we need to cast and process all of the concrete range types. This is a great place to write some quick unit tests to check our implementation.

````swift
extension UIView {
    func contentSizeCategory(_ size: UIContentSizeCategory?) {
        self.minimumContentSizeCategory = size
        self.maximumContentSizeCategory = size
    }
  
    func contentSizeCategory<T>(_ range: T) where T: RangeExpression, T.Bound == UIContentSizeCategory {
        if let range = range as? ClosedRange<T.Bound> {
            self.minimumContentSizeCategory = range.lowerBound
            self.maximumContentSizeCategory = range.upperBound
        }
        else if let range = range as? PartialRangeFrom<T.Bound> {
            self.minimumContentSizeCategory = range.lowerBound
            self.maximumContentSizeCategory = nil
        }
        else if let range = range as? PartialRangeThrough<T.Bound> {
            self.minimumContentSizeCategory = nil
            self.maximumContentSizeCategory = range.upperBound
        }
        else if let range = range as? PartialRangeUpTo<T.Bound> {
            self.minimumContentSizeCategory = nil
            self.maximumContentSizeCategory = UIContentSizeCategory.orderedSizes.element(before: range.upperBound)
        }
        else if let range = range as? Range<T.Bound> {
            self.minimumContentSizeCategory = range.lowerBound
            self.maximumContentSizeCategory = UIContentSizeCategory.orderedSizes.element(before: range.upperBound)
        }
    }
}
````

In order to process the half-open ranges for `PartialRangeUpTo` and `Range`, we can use a convenience function.

````swift
extension BidirectionalCollection where Iterator.Element: Equatable {
    func element(before item: Element) -> Element? {
        guard let itemIndex = self.firstIndex(of: item), itemIndex != startIndex else { return nil }
        return self[index(before: itemIndex)]
    }
}
````

### Conclusion
In UIKit, we would normally set a range of UIContentSizeCategory like this.

````swift
let view = UIView()
view.minimumContentSizeCategory = .small
view.maximumContentSizeCategory = .accessibilityMedium
````

But now, we can be more concise and expressive, with the added benefit of having our values checked at compile time.

````swift
let view = UIView()
view.contentSizeCategory(.small ... .accessibilityMedium)
````

That is a small amount of work for a big win!