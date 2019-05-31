# Swift Confidence Sheet

## Table of Contents
[Codable String as Value](https://github.com/sufan/spiderpig#codable-string-as-value)  
[Current Visible ViewController](https://github.com/sufan/spiderpig#current-visible-viewcontroller)  
[Exploring View Hierarchy](https://github.com/sufan/spiderpig#exploring-view-heirarchy)  
[Output Object as CSV](https://github.com/sufan/spiderpig#output-object-as-csv)  
[Resign Current Responder](https://github.com/sufan/spiderpig#resign-current-responder)  
[SizeToFit with Aspect Ratio](https://github.com/sufan/spiderpig#sizetofit-with-aspect-ratio)  
[Use HTML Text Formatting in Localized String](https://github.com/sufan/spiderpig#use-html-text-formatting-in-localized-string)  

## Chapter
### [Codable String as Value](#codable-string-as-value)
```swift
struct StringCodable<T: LosslessStringConvertible> : Codable, CustomStringConvertible {

    var value: T

    init(_ value: T) {
        self.value = value
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let string = try container.decode(String.self)

        guard let value = T(string) else {
            throw DecodingError.dataCorruptedError(in: container, debugDescription: "\(string) is not decodable as a \(T.self)")
        }

        self.value = value
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(value.description)
    }

    var description: String {
        return value.description
    }

}

struct StringAsValue: Codable {

    let bytesReceived: StringCodable<Int>

}

```

### [Current Visible ViewController](#current-visible-viewcontroller)
```swift
extension UIApplication {

    func visibleViewController(_ root: UIViewController? = UIApplication.shared.keyWindow?.rootViewController) -> UIViewController? {
        if let navigationControler = root as? UINavigationController {
            return visibleViewController(navigationControler.visibleViewController)
        } else if let tabBar = root as? UITabBarController, let selected = tabBar.selectedViewController {
            return visibleViewController(selected)
        } else if let presented = root?.presentedViewController {
            return visibleViewController(presented)
        }

        return root
    }

}
```

### [Exploring View Heirarchy](#exploring-view-heirarchy)
```swift
(lldb) expr -l objc -O -- [[UIWindow keyWindow] recursiveDescription]
```

### [Output Object as CSV](#output-object-as-csv)
```swift
protocol CSVable {
    func toCSV() -> String
}

extension CSVable {

    var delimiter: String { return "," }

    var allTitles: [String] {
        var titles = [String]()

        let mirror = Mirror(reflecting: self)
        mirror.children.forEach({ (child) in
            if let value = child.value as? CSVable {
                titles.append(contentsOf: value.allTitles)
            } else {
                titles.append(child.label ?? "unknown")
            }
        })

        return titles
    }

    var allValues: [Any] {
        var values = [Any]()

        let mirror = Mirror(reflecting: self)
        mirror.children.forEach({ (child) in
            if let value = child.value as? CSVable {
                values.append(contentsOf: value.allValues)
            } else {
                // unwrapping Any Optional
                let childMirror = Mirror(reflecting: child.value)
                guard childMirror.displayStyle == .optional, let first = childMirror.children.first else {
                    values.append(child.value)
                    return
                }
                values.append(first.value)
            }
        })

        return values
    }

    fileprivate var allValueStrings: [String] {
        return allValues.map { (value) -> String in
            switch value {
            case let item as Date:
                return String(describing: item.epochLong)
            case let item as DateInterval:
                return String(describing: item.start.epochLong) + " " + String(describing: item.end.epochLong)
            default:
                return String(describing: value)
            }
        }
    }

    fileprivate var csvTitles: String {
        return allTitles.joined(separator: delimiter) + "\n"
    }

    private var csvValues: String {
        var string = allValueStrings.joined(separator: delimiter) + "\n"
        string = string.replacingOccurrences(of: "nil", with: " ")
        return string
    }

    func toCSV() -> String {
        return csvTitles + csvValues
    }

}

extension Array: CSVable where Element: CSVable {

    var allTitles: [String] {
        guard let first = first else {
            return []
        }

        return first.allTitles
    }

    var allValues: [Any] {
        return map { $0.allValues }
    }

    private var csvValues: String {
        var data = String()

        forEach { (element) in
            var string = element.allValueStrings.joined(separator: delimiter) + "\n"
            string = string.replacingOccurrences(of: "nil", with: "")
            data.append(string)
        }

        return data
    }

    func toCSV() -> String {
        return csvTitles + csvValues
    }

}
```

### [Resign Current Responder](#resign-current-responder)
```swift
UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to:nil, from:nil, for:nil)
```

### [SizeToFit with Aspect Ratio](#sizetofit-with-aspect-ratio)
```swift
import AVFoundation

extension CGRect {

    func sizeToFit(with aspectRatio: CGSize) -> CGRect {
        return AVMakeRect(aspectRatio: aspectRatio, insideRect: self)
    }

}
```

### [Use HTML Text Formatting in Localized String](#use-html-text-formatting)
```swift
extension NSAttributedString {

    convenience init?(htmlString html: String, font: UIFont? = nil) throws {
        let options: [NSAttributedString.DocumentReadingOptionKey : Any] = [
            .documentType: NSAttributedString.DocumentType.html,
            .characterEncoding: String.Encoding.utf8.rawValue
        ]

        guard let data = html.data(using: .utf8, allowLossyConversion: true),
            let string = try? NSMutableAttributedString(data: data, options: options, documentAttributes: nil) else {
                return nil
        }

        if let font = font {
            let range = NSRange(location: 0, length: string.length)

            string.enumerateAttribute(.font, in: range, options: .longestEffectiveRangeNotRequired) { attributes, range, _ in
                if let htmlFont = attributes as? UIFont {
                    let traits = htmlFont.fontDescriptor.symbolicTraits

                    var descriptor = htmlFont.fontDescriptor.withFamily(font.familyName)
                    if (traits.rawValue & UIFontDescriptor.SymbolicTraits.traitBold.rawValue) != 0 {
                        descriptor = descriptor.withSymbolicTraits(.traitBold)!
                    }

                    if (traits.rawValue & UIFontDescriptor.SymbolicTraits.traitItalic.rawValue) != 0 {
                        descriptor = descriptor.withSymbolicTraits(.traitItalic)!
                    }

                    string.addAttribute(.font, value: UIFont(descriptor: descriptor, size: font.pointSize), range: range)
                }
            }
        }

        self.init(attributedString: string)
    }

}

let htmlString = NSLocalizedString("key", tableName: nil, bundle: Bundle.main, value: "The <u>quick</u> <i>brown</i> <b>fox</b>", comment: "html formatted")
try? NSAttributedString(htmlString: htmlString, font: UIFont())

```
