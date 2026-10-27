# workflow_tool

Patch application tool for the Flutter engine checkout.

## Apply patches

```sh
dart pub get
dart run bin/patch.dart apply ../patches/patches.json ../flutter
```

## Record patches

Record patches by comparing the `unpatched` and `patched` refs in the
checkout:

```sh
dart run bin/patch.dart record ../patches/patches.json ../flutter .
```
