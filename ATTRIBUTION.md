# Attribution

## Standing on giants

The Swift skills in this guide are consolidations of the two most-starred open-source Swift agent
skill packs on GitHub. Both are MIT licensed, and both are excellent standalone:

| Source | Pack | Stars | License | Content verified in |
|---|---|---|---|---|
| Paul Hudson ([@twostraws](https://github.com/twostraws)) | [SwiftUI-Agent-Skill](https://github.com/twostraws/SwiftUI-Agent-Skill), [Swift-Concurrency-Agent-Skill](https://github.com/twostraws/Swift-Concurrency-Agent-Skill), [Swift-Testing-Agent-Skill](https://github.com/twostraws/Swift-Testing-Agent-Skill), [SwiftData-Agent-Skill](https://github.com/twostraws/SwiftData-Agent-Skill) | 4.7k+ | MIT | swiftui, swift-concurrency, swift-data, swift-testing, xcode-build |
| Antoine van der Lee ([@AvdLee](https://github.com/AvdLee)) | [SwiftUI-Agent-Skill](https://github.com/AvdLee/SwiftUI-Agent-Skill), [Swift-Concurrency-Agent-Skill](https://github.com/AvdLee/Swift-Concurrency-Agent-Skill), [Core-Data-Agent-Skill](https://github.com/AvdLee/Core-Data-Agent-Skill), [Swift-Testing-Agent-Skill](https://github.com/AvdLee/Swift-Testing-Agent-Skill), [Xcode-Build-Optimization-Agent-Skill](https://github.com/AvdLee/Xcode-Build-Optimization-Agent-Skill) | 3.5k+ | MIT | swiftui, swift-data, swift-concurrency, swift-testing, xcode-build |

Specific inheritances:

- `swiftui/references/latest-apis.md` is based on Antoine van der Lee's Apple-documentation
  comparison; his attribution line is preserved in the file.
- `swiftui/references/api-refresh.md` is a maintenance workflow adapted from his open-source
  `update-swiftui-apis` skill (MIT), with its scan manifest shipped alongside
  (`api-refresh-scan-manifest.md`).
- Where merged sources genuinely disagreed, the reference files flag the conflict instead of
  presenting a false consensus.

## Also in the family

- **[ios-simulator-skill](https://github.com/conorluddy/ios-simulator-skill)** — Conor Luddy.
  Vendored in this repo as a trimmed adaptation (MIT). His original `LICENSE.md` is included in
  the skill directory; use his repo for the maintained upstream with newer launch flags and the
  full script package.
- **[swift-ios-skills](https://github.com/dpearson2699/swift-ios-skills)** — dpearson2699.
  PolyForm Perimeter 1.0.0 licensed; 100+ framework skills for iOS 26+. Used daily here but
  **not redistributed** — his license forbids competing distributions, so go upstream for it.

## Tooling notes

- `swiftui` deprecation tracking depends on the Sosumi MCP for Apple documentation access
  (`searchAppleDocumentation` and friends).
