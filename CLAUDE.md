# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

bcrypt_pbkdf-ruby is a Ruby gem implementing OpenBSD's bcrypt_pbkdf (a variant of PBKDF2 with bcrypt-based PRF). It's primarily used by net-ssh to read password-encrypted Ed25519 keys. The gem consists of a C extension that wraps OpenBSD's bcrypt_pbkdf implementation.

## Development Commands

### Setup
```sh
bundle install
```

### Building the Extension
```sh
bundle exec rake compile       # Compile the C extension
bundle exec rake clean clobber # Clean build artifacts
```

### Testing
```sh
bundle exec rake test          # Run all tests (default task includes compile + test)
bundle exec rake               # Shortcut: compile + test
```

### Documentation
```sh
bundle exec rake rdoc          # Generate RDoc documentation in doc/rdoc/
```

### Cross-Platform Build (Requires Docker)
This gem supports cross-compilation for multiple platforms using rake-compiler-dock.

Supported platforms:
- arm64-darwin
- x64-mingw-ucrt
- x64-mingw32
- x86-mingw32
- x86_64-darwin

Supported Ruby versions: 2.7.0, 3.0.0, 3.1.0, 3.2.0, 3.3.0, 3.4.0

```sh
gem install rake-compiler-dock

# Build for all platforms
bundle exec rake gem:all

# Build for specific platform
bundle exec rake gem:arm64-darwin
bundle exec rake gem:x64-mingw-ucrt
```

### Version Management
```sh
# Bump to final release (removes pre-release suffix or increments minor)
bundle exec rake vbump:final

# Increment pre-release version (rc, alpha, beta, etc.)
bundle exec rake vbump:pre[rc]    # Creates or increments rc version
bundle exec rake vbump:pre[alpha] # Creates or increments alpha version
```

### Release
```sh
bundle exec rake release           # Standard release
bundle exec rake gem:release       # Release native gems for all platforms
```

## Architecture

### C Extension Layer
Located in `ext/mri/`, this contains the native implementation:

- **bcrypt_pbkdf.c**: Core bcrypt_pbkdf algorithm from OpenBSD
- **bcrypt_pbkdf_ext.c**: Ruby C API bindings that expose `BCryptPbkdf::Engine.__bc_crypt_pbkdf` and `BCryptPbkdf::Engine.__bc_crypt_hash`
- **blowfish.c**: Blowfish cipher implementation
- **hash_sha512.c**: SHA-512 hash implementation
- **extconf.rb**: Simple mkmf-based configuration

The C extension defines two internal methods:
- `BCryptPbkdf::Engine.__bc_crypt_pbkdf(pass, salt, keylen, rounds)` - Full key derivation
- `BCryptPbkdf::Engine.__bc_crypt_hash(pass, salt)` - Single bcrypt hash iteration (used internally by tests)

### Ruby Layer
Located in `lib/bcrypt_pbkdf.rb`:

- Loads the compiled C extension based on Ruby version
- Provides public API: `BCryptPbkdf.key(pass, salt, keylen, rounds)`
- This is a thin wrapper around the C extension's `__bc_crypt_pbkdf` method

### Test Suite
Located in `test/bcrypt_pnkdf/engine_test.rb`:

- Uses minitest framework
- Includes a pure Ruby reference implementation of bcrypt_pbkdf for verification
- Tests that native C implementation matches Ruby reference implementation
- Uses test vectors with known password/salt/keylen/rounds combinations

## Key Implementation Details

The algorithm uses a stride-based key generation approach:
1. SHA-512 is used to hash the password and salt
2. bcrypt is used as the PRF (pseudo-random function)
3. Multiple rounds of bcrypt hashing are performed
4. Results are XORed together and arranged with a stride pattern to produce the final key

The pure Ruby reference implementation in the test file demonstrates this algorithm clearly and can be used to understand the expected behavior when modifying the C extension.

## Gemspec Configuration

Version is managed in `bcrypt_pbkdf.gemspec` at line 3. The Rakefile includes version management tasks that automatically update this file.

Extensions are configured to build with `ext/mri/extconf.rb` which uses the standard mkmf approach.
