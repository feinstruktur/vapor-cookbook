# Vapor Cookbook

Vapor cookbook contains snippets for problems that I came across when starting to write a backend service recently. The Discord group was very helpful to solve these problems and often I found solutions by searching through its archives. This list of solutions for problems I encountered and found solutions for may prove helpful for others going down that same path, trying to figure out the Vapor APIs.

<!-- TOC -->

- [Vapor Cookbook](#vapor-cookbook)
    - [Requests](#requests)
        - [Download file](#download-file)
        - [Render page from a custom DSL](#render-page-from-a-custom-dsl)
        - [Sending arrays and dictionaries via form](#sending-arrays-and-dictionaries-via-form)
    - [Database](#database)
        - [No ORM batch fetch with filter and sorting](#no-orm-batch-fetch-with-filter-and-sorting)
        - [Batch insert](#batch-insert)
    - [Reading config files](#reading-config-files)
    - [Testing](#testing)

<!-- /TOC -->

## Requests

### Download file

```swift
final class Model: SQLiteModel {
    var id: Int?
    var filename: String
    var data: Data
    ...
}

func download(_ req: Request) throws -> Future<Response> {
    return try req.parameters.next(Model.self).map { obj in
        let headers: HTTPHeaders = [
            "content-disposition": "attachment; filename=\"\(obj.filename)\""
        ]
        let res = HTTPResponse(headers: headers, body: obj.data)
        return req.response(http: res)
    }
}
```

Or via a `Request` extension:

```swift
extension Request {
    func response(file: File, attachment: Bool = true) -> Response {
        var headers = HTTPHeaders()
        headers.contentType = file.contentType

        if attachment {
            headers.add(
                name: .contentDisposition,
                value: "attachment; filename=\"\(file.filename)\""
            )
        }

        let res = HTTPResponse(headers: headers, body: file.data)
        return response(http: res)
    }
}
```

```swift
func download(_ req: Request) throws -> Future<Response> {
    return try req.parameters.next(Model.self).map { obj in
		  let file = File(data: obj.data, filename: obj.filename)
        return req.response(file: file)
    }
}
```

### Render page from a custom DSL

```swift
func render(_ node: HTML.Node) -> String {
    // Renders a DSL's HTML.Node into an HTML string
}

func homePage() -> HTML.Node {
    // Creates the home page in DSL "lingo"
}

router.get { req -> HTTPResponse in
    let node = homePage()
    let html = render(node)
    var res = HTTPResponse(status: .ok, body: html)
    res.contentType = .html
    return res
}
```

### Sending arrays and dictionaries via form

TODO

## Database

### No ORM batch fetch with filter and sorting

```swift
// SQLTable is a built in protocol
extension MyModel: SQLTable {
    static let sqlTableIdentifierString = "my_model"
}

protocol ExportableSQLTable: SQLTable {
    var dateCreated: Date { get }
}

func fetchBatch<T: ExportableSQLTable>(entity: T.Type,
                                       database: PostgreSQLConnection,
                                       batchSize: Int?,
                                       after: Date) -> Future<[T]> {
    return database
        .select()
        .all()
        .from(T.self)
        .where(\T.dateCreated, .greaterThan, after)
        .orderBy(\T.dateCreated)
        .limit(batchSize)
        .all(decoding: T.self)
}
```

### Batch insert

```swift
// SQLTable is a built in protocol
extension MyModel: SQLTable {
    static let sqlTableIdentifierString = "my_model"
}

func insertBatch<T: SQLTable>(rows: [T],
                              database: PostgreSQLConnection) -> Future<[T]> {
    guard rows.count > 0 else { return database.future([T]()) }
    return database
        .insert(into: T.self)
        .run()
        .map(to: [T].self) { rows }
}
```

## Reading config files

This snippet is based on a link someone shared in the Discord channel. Unfortunately I could not find it again to attribute it to the original author.

```swift
import Foundation
import Service

#if os(Linux)
import Glibc
#else
import Darwin
#endif

enum ConfigError: Error {
    case noSuchFile(String)
}

typealias ConfigItems = Dictionary<String, String>

func parseConfig(_ config: String) -> ConfigItems {
    let pairs = config.components(separatedBy: "\n")
        .map { $0.trimmingCharacters(in: CharacterSet.whitespacesAndNewlines) }
        .compactMap { (line: String) -> (key: String, value: String)? in
            guard let equalsIndex = line.index(of: "=") else {
                return nil
            }
            guard !line.starts(with: "#") else { return nil }
            let _line: String
            if let idx = line.firstIndex(of: "#") {
                _line = String(line[line.startIndex..<idx])
            } else {
                _line = line
            }
            let key = String(_line.prefix(through: _line.index(before: equalsIndex)))
                .trimmingCharacters(in: .whitespaces)
            let value = String(_line.suffix(from: _line.index(after: equalsIndex)))
                .trimmingCharacters(in: .whitespaces)
            return (key: key, value: value)
    }
    return ConfigItems(uniqueKeysWithValues: pairs)
}

func readConfig(from path: String) throws -> ConfigItems {
    guard
        let data = FileManager.default.contents(atPath: path),
        let string = String(data: data, encoding: .utf8) else {
            throw ConfigError.noSuchFile(path)
    }
    return parseConfig(string)
}

extension Environment {
    func read(from path: String) throws {
        let config = try readConfig(from: path)
        return config.forEach { setenv($0.key, $0.value, 1) }
    }
}
```

## Testing

TODO
