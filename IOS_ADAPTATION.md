# iOS / SwiftUI Adaptation

This document maps the Bauhaus & Swiss International Typographic Style principles from `SKILL.md` to SwiftUI implementation patterns. Read `SKILL.md` first for design philosophy — this file covers only the platform-specific translation.

## Design Tokens

Define all tokens in a single file (e.g. `BauhausTokens.swift`). No magic numbers anywhere else.

```swift
import SwiftUI

enum BauhausTokens {
    // MARK: - Spacing
    static let margin: CGFloat = 20
    static let gutter: CGFloat = 12

    // MARK: - Type Scale (perfect fourth: 1.333)
    static let textBase: CGFloat = 16
    static let textSm:  CGFloat = textBase / 1.333   // ~12
    static let textLg:  CGFloat = textBase * 1.333   // ~21.3
    static let textXl:  CGFloat = textLg * 1.333     // ~28.4
    static let text2Xl: CGFloat = textXl * 1.333     // ~37.9
    static let text3Xl: CGFloat = text2Xl * 1.333    // ~50.5

    // MARK: - Colors
    static let bgLight       = Color(hex: "F5F5F0")
    static let bgDark        = Color(hex: "1A1A1A")
    static let textPrimary   = Color(hex: "1A1A1A")
    static let textSecondary = Color(hex: "6B6B6B")
    static let accent        = Color(hex: "E63946")  // Bauhaus red
    static let rule          = Color(hex: "D4D4D4")

    // MARK: - Shape
    static let cornerRadius: CGFloat = 2  // max 2px, prefer 0
}
```

`Color(hex:)` extension:

```swift
extension Color {
    init(hex: String) {
        let scanner = Scanner(string: hex)
        var rgb: UInt64 = 0
        scanner.scanHexInt64(&rgb)
        self.init(
            red: Double((rgb >> 16) & 0xFF) / 255,
            green: Double((rgb >> 8) & 0xFF) / 255,
            blue: Double(rgb & 0xFF) / 255
        )
    }
}
```

## Pillar 1 → Grid

SwiftUI `Grid` (iOS 16+) or `LazyVGrid` replaces CSS Grid.

```swift
let columns = Array(
    repeating: GridItem(.flexible(), spacing: BauhausTokens.gutter),
    count: 4
)

LazyVGrid(columns: columns, spacing: BauhausTokens.gutter) {
    ForEach(items) { item in
        CardView(item: item)
    }
}
.padding(BauhausTokens.margin)
```

Asymmetric splits (7:5 ratio):

```swift
GeometryReader { geo in
    HStack(spacing: BauhausTokens.gutter) {
        primaryContent
            .frame(width: geo.size.width * 7/12)
        secondaryContent
    }
}
.padding(BauhausTokens.margin)
```

## Pillar 2 → Typography

SF Pro is iOS's system sans-serif — same role as Inter/Helvetica in the web version. For CJK content, let the system font handle fallback.

```swift
extension Font {
    static func bauhaus(_ size: CGFloat, weight: Font.Weight = .regular) -> Font {
        .system(size: size, weight: weight, design: .default)
    }

    static let headline = bauhaus(BauhausTokens.text2Xl, weight: .bold)
    static let title    = bauhaus(BauhausTokens.textXl, weight: .bold)
    static let body     = bauhaus(BauhausTokens.textBase, weight: .light)
    static let label    = bauhaus(BauhausTokens.textSm, weight: .regular)
    static let caption  = bauhaus(BauhausTokens.textSm, weight: .light)
}
```

Uppercase label modifier:

```swift
struct BauhausLabel: ViewModifier {
    func body(content: Content) -> some View {
        content
            .font(.bauhaus(BauhausTokens.textSm, weight: .medium))
            .textCase(.uppercase)
            .kerning(BauhausTokens.textSm * 0.1)
    }
}

extension View {
    func bauhausLabel() -> some View { modifier(BauhausLabel()) }
}
```

Flush-left alignment (enforce everywhere):

```swift
.multilineTextAlignment(.leading)
.frame(maxWidth: .infinity, alignment: .leading)
```

## Pillar 3 → Color

Use Asset Catalog or `BauhausTokens`. Maximum 3 colors, no gradients, accent used sparingly.

```swift
// ✅ Solid fill
Rectangle().fill(BauhausTokens.accent)

// ❌ Never
LinearGradient(...)
AngularGradient(...)
```

Dark mode: define a paired token set or use adaptive `Color` from Asset Catalog. Keep the palette constrained — dark mode is not an excuse to add colors.

## Pillar 4 → Geometric Abstraction

SwiftUI `Shape`, `Path`, and `Canvas` replace SVG.

```swift
struct AccentGeometry: View {
    var body: some View {
        Canvas { context, size in
            let diameter = size.width * 0.75
            let rect = CGRect(
                x: (size.width - diameter) / 2,
                y: (size.height - diameter) / 2,
                width: diameter, height: diameter
            )
            context.stroke(
                Path(ellipseIn: rect),
                with: .color(BauhausTokens.accent),
                lineWidth: 2
            )

            var line = Path()
            line.move(to: .zero)
            line.addLine(to: CGPoint(x: size.width, y: size.height))
            context.stroke(line, with: .color(BauhausTokens.accent), lineWidth: 1)
        }
    }
}
```

Use `.clipped()` on geometric elements for the partially-cropped tension effect.

## Pillar 5 → Asymmetry

```swift
HStack(spacing: BauhausTokens.gutter) {
    VStack(alignment: .leading) { /* primary */ }
        .frame(maxWidth: .infinity)
    VStack(alignment: .leading) { /* secondary */ }
        .frame(width: 120)
}
```

Anchor headlines to `.leading`. Let whitespace work on the trailing side.

## Motion & Interaction

```swift
// ✅ Restrained easeOut
.animation(.easeOut(duration: 0.3), value: trigger)
.transition(.opacity.combined(with: .move(edge: .bottom)))

// ❌ Never
.animation(.spring(), value: trigger)
.animation(.bouncy, value: trigger)
.transition(.scale)
```

Stagger entries with `delay`:

```swift
ForEach(Array(items.enumerated()), id: \.element.id) { index, item in
    ItemView(item: item)
        .animation(
            .easeOut(duration: 0.3).delay(Double(index) * 0.06),
            value: isVisible
        )
}
```

Duration: 200–400ms. No parallax, no blur transitions, no 3D rotations.

## Anti-Patterns (SwiftUI)

- ❌ `RoundedRectangle(cornerRadius:)` above 4 → use `BauhausTokens.cornerRadius` (2)
- ❌ `.shadow()` of any kind → use `Rectangle().fill(BauhausTokens.rule).frame(height: 1)`
- ❌ `LinearGradient`, `RadialGradient`, `AngularGradient`
- ❌ `.background(.ultraThinMaterial)` or any `Material`
- ❌ `.spring()`, `.bouncy`, `.snappy` animations → use `.easeOut(duration: 0.3)`
- ❌ `.font(.system(.body, design: .serif))` → sans-serif only (`.default`)
- ❌ `.multilineTextAlignment(.center)` on paragraph text
- ❌ SF Symbols with `.renderingMode(.multicolor)` → use `.monochrome` in accent color
- ❌ `TabView` with `.tabViewStyle(.page)` carousel effects
- ❌ `.blur()` modifier for depth

## Dividers

```swift
// ✅ Bauhaus divider
Rectangle()
    .fill(BauhausTokens.rule)
    .frame(height: 1)

// ❌ SwiftUI default Divider (inconsistent styling)
Divider()
```

## Checklist Before Output

1. ☐ All spacing uses `BauhausTokens.margin` / `.gutter` — no magic numbers
2. ☐ Type scale is mathematical (1.333 ratio)
3. ☐ One typeface (SF Pro via `.system`), weight contrast for hierarchy
4. ☐ Maximum 3 colors (bg + text + accent)
5. ☐ Corner radius ≤ 2px
6. ☐ Zero gradients, zero shadows, zero materials
7. ☐ Text is `.leading` aligned
8. ☐ Layout is asymmetric
9. ☐ Animations use `.easeOut`, 200–400ms
10. ☐ Every element justifies its existence
