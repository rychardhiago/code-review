# PHP Coding Standards

This document defines the default PHP coding standards used when reviewing
changes in PHP applications. It summarizes relevant PHP-FIG recommendations;
the linked specifications remain the authoritative source.

## Core standards

### PSR-1: Basic Coding Standard

Apply the following PSR-1 rules:

- PHP files use `<?php` and `<?=` tags and UTF-8 without a byte order mark.
- A file should either declare symbols or produce side effects, rather than
  doing both.
- Namespaces and class names follow an autoloading standard, preferably PSR-4.
- Class names use `StudlyCaps`; class constants use uppercase names with
  underscores; method names use `camelCase`.
- Property naming is not prescribed by PSR-1; check that the project uses its
  chosen convention consistently.

See [PSR-1](https://www.php-fig.org/psr/psr-1/).

### PSR-12: Extended Coding Style Guide

PSR-12 extends and replaces PSR-2 and requires PSR-1. Apply it as the default
formatting standard, subject to the PHP versions supported by the project.
Among its key rules:

- Use LF line endings, end PHP-only files with a newline, and omit the closing
  `?>` tag in PHP-only files.
- Use four spaces for indentation, no trailing whitespace, and no more than
  one statement per line.
- Keep lines within the 120-character soft limit; aim for 80 characters where
  practical.
- Use lowercase for PHP keywords and types, including short type names such as
  `bool` and `int`.
- Follow the prescribed ordering and spacing for declarations, imports,
  classes, methods, control structures, and multiline argument lists.
- Declare visibility for properties and methods; do not use `var` for
  properties.

Do not report syntax or style that depends on PHP features unavailable in the
project's supported PHP versions as a violation.

See [PSR-12](https://www.php-fig.org/psr/psr-12/).

## Applicable standards

Apply other PSRs when the changed code implements or integrates with the
relevant contract. Do not flag a change merely because it does not use an
unrelated PSR.

- **PSR-4 — Autoloading Standard:** Apply when reviewing namespaces,
  autoloading configuration, or class file placement. Check namespace-prefix
  mappings, directory and file names, and case-sensitive class references.
  See [PSR-4](https://www.php-fig.org/psr/psr-4/).
- **PSR-3 — Logger Interface:** Apply when code implements or consumes the
  `Psr\Log\LoggerInterface` contract. Check log levels, message/context usage,
  and exception context handling. See
  [PSR-3](https://www.php-fig.org/psr/psr-3/).
- **Other accepted PSRs:** Review against another PSR only when the changed
  code uses or claims to implement that standard (for example, PSR-7 HTTP
  messages or PSR-11 containers). Confirm its current status and scope in the
  [PHP-FIG PSR index](https://www.php-fig.org/psr/).

## Project-specific configuration

Project documentation and configuration may define a more specific or
different convention. Respect those documented choices when they do not
conflict with the project's explicit PSR compliance claims. Consider the
project's PHP version and existing formatter or linter configuration before
reporting style findings.
