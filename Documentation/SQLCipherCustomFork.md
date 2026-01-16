Custom SQLCipher Fork
=====================

The officially supported fork of GRDB w/SQLCipher is available here: <url to fork>
This fork is maintained by GRDB in collaboration with the SQLCipher team and is the recommended fork to use to enable SQLCipher encryption for GRDB.

If you have requirements that the official fork doesn't support, you're can fork GRDB yourself and include a custom copy of SQLCipher. This guide provides instructions for what is minimally required to get up and running with your fork:

1. [Fork](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo) GRDB
2. Clone the repository on your machine

```sh
git clone <url to fork>
```
3. Build the sqlite amalgamation w/SQLCipher (for example using the SQLCipher documentation for [Apple Source Integration](https://www.zetetic.net/sqlcipher/sqlcipher-apple-community/#source-integration)) to generate `sqlite3.c`/`sqlite3.h` files
4. Open the GRDB Package.swift project in Xcode. Create a new Directory named SQLCipher in the Sources directory and move the `sqlite3.c`/`sqlite3.h` files in the directory structure like so

```sh
SQLCipher
    ├── include
    │   ├── module.modulemap
    │   ├── SQLCipher
    │   │   └── sqlite3.h
    │   └── SQLCipher.h
    └── sqlite3.c
```

Add a `module.modulemap` file in the `include` directory with these contents

```swift
module SQLCipher {
    umbrella header "SQLCipher.h"
    export *
}
```

Add a `SQLCipher.h` umbrella header file in the `include` directory with these contents

```objc
#ifndef SQLCipher_h
#define SQLCipher_h

#import <SQLCipher/sqlite3.h>
#endif /* SQLCipher_h */
```

5. Create a new `GRDBSQLCipher` directory in the `Sources` directory with this structure

```sh
GRDBSQLCipher
│   ├── include
│   │   ├── module.modulemap
│   │   └── SQLCipher_config.h
│   └── SQLCipher_config.c
```

Add a `module.modulemap` file in the `include` directory with these contents


```swift
module GRDBSQLCipher {
    header "SQLCipher_config.h"
    export *
}
```

Add a `SQLCipher_config.h` file in the `include` directory with these contents

```objc
#ifndef grdb_config_h
#define grdb_config_h

#import <SQLCipher/sqlite3.h>

typedef void(*_errorLogCallback)(void *pArg, int iErrCode, const char *zMsg);

/// Wrapper around sqlite3_config(SQLITE_CONFIG_LOG, ...) which is a variadic
/// function that can't be used from Swift.
static inline void _registerErrorLogCallback(_errorLogCallback callback) {
    sqlite3_config(SQLITE_CONFIG_LOG, callback, 0);
}

/// Wrapper around sqlite3_db_config() which is a variadic function that can't
/// be used from Swift.
static inline void _disableDoubleQuotedStringLiterals(sqlite3 *db) {
    sqlite3_db_config(db, SQLITE_DBCONFIG_DQS_DDL, 0, (void *)0);
    sqlite3_db_config(db, SQLITE_DBCONFIG_DQS_DML, 0, (void *)0);
}

/// Wrapper around sqlite3_db_config() which is a variadic function that can't
/// be used from Swift.
static inline void _enableDoubleQuotedStringLiterals(sqlite3 *db) {
    sqlite3_db_config(db, SQLITE_DBCONFIG_DQS_DDL, 1, (void *)0);
    sqlite3_db_config(db, SQLITE_DBCONFIG_DQS_DML, 1, (void *)0);
}
#endif /* grdb_config_h */
```

6. Modify the `Package.swift` to include the two new targets and add them as depdencies for GRDB

```swift
        .target(
            name: "GRDB",
            dependencies: [
                .target(name: "SQLCipher"),
                .target(name: "GRDBSQLCipher")
            ],
            path: "GRDB",
            resources: [.copy("PrivacyInfo.xcprivacy")],
            cSettings: cSettings,
            swiftSettings: swiftSettings),
        .target(
            name: "SQLCipher",
            publicHeadersPath: "include",
            cSettings: cSettings
        ),
        .target(
            name: "GRDBSQLCipher",
            dependencies: [.target(name: "SQLCipher")]
        ),
```

7. Add these swiftSettings and cSettings to `Package.swift` (and whatever other flags desired)

```swift
var swiftSettings: [SwiftSetting] = [
    .define("SQLITE_ENABLE_FTS5"),
    .define("SQLITE_ENABLE_SNAPSHOT"),
    .define("SQLCipher"), // added
    .define("SQLITE_HAS_CODEC") // added
]
```

```swift
var cSettings: [CSetting] = [
    .define("NDEBUG", to: nil),
    .define("SQLCIPHER_CRYPTO_CC", to: nil),
    .define("SQLITE_HAS_CODEC", to: nil),
    .define("SQLITE_TEMP_STORE", to: "2"),
    .define("SQLITE_THREADSAFE", to: "1"),
    .define("SQLITE_EXTRA_INIT", to: "sqlcipher_extra_init"),
    .define("SQLITE_EXTRA_SHUTDOWN", to: "sqlcipher_extra_shutdown"),
    .define("SQLITE_ENABLE_FTS5", to: nil),
    .define("SQLITE_ENABLE_SNAPSHOT", to: nil)
]
```

