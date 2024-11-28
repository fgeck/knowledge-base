# IntelliJ

## Run Configurations

## Autoformatting Kotlin Code

- recommended in general: [ktlint](https://github.com/pinterest/ktlint/)
- either use [gradle plugin](https://github.com/JLLeitschuh/ktlint-gradle)
  - used ktlint ruleset version can be found [here](https://github.com/JLLeitschuh/ktlint-gradle):

```kotlin
configure<org.jlleitschuh.gradle.ktlint.KtlintExtension> {
    version.set("1.0.1")
    ...
}
```

- or use [ktlint-intellij-pluign](https://github.com/nbadal/ktlint-intellij-plugin)
  - Used ktlint ruleset version can be configured in plugin settings
  - PRO: can automatically fix all issues on the fly
