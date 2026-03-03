# cracktrader_extras Release Notes

`cracktrader-extras` is now maintained as a standalone repository:

- https://github.com/cracktrader/cracktrader-extras

Core (`cracktrader`) no longer keeps an in-tree mirror of the extras package.

## Release workflow

1. Work in the `cracktrader-extras` repository directly.
2. Run that repository's lint/test checks.
3. Tag and publish from `cracktrader-extras`.

## Core repo changes

When extras API/examples change, update references in core docs (for example,
`docs/plugins_and_extras.md`, `docs/strategy_guide.md`, and related pages) to
point at the latest extras repository paths.
