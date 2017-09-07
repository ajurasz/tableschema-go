[![Build Status](https://travis-ci.org/frictionlessdata/tableschema-go.svg?branch=master)](https://travis-ci.org/frictionlessdata/tableschema-go) [![Coverage Status](https://coveralls.io/repos/github/frictionlessdata/tableschema-go/badge.svg?branch=master)](https://coveralls.io/github/frictionlessdata/tableschema-go?branch=master) [![Go Report Card](https://goreportcard.com/badge/github.com/frictionlessdata/tableschema-go)](https://goreportcard.com/report/github.com/frictionlessdata/tableschema-go) [![Gitter chat](https://badges.gitter.im/gitterHQ/gitter.png)](https://gitter.im/frictionlessdata/chat) [![GoDoc](https://godoc.org/github.com/frictionlessdata/tableschema-go?status.svg)](https://godoc.org/github.com/frictionlessdata/tableschema-go)

# tableschema-go
A Go library for working with [Table Schema](http://specs.frictionlessdata.io/table-schema/).


# Main Features

* [table](https://godoc.org/github.com/frictionlessdata/tableschema-go/table) package defines [Table](https://godoc.org/github.com/frictionlessdata/tableschema-go/csv#Table) and `Iterator` interfaces, which are used to manipulate and/or explore tabular data;
* [csv](https://godoc.org/github.com/frictionlessdata/tableschema-go/csv) package contains implementation of Table and Iterator interfaces to deal with CSV format;
* [schema](https://github.com/frictionlessdata/tableschema-go/tree/master/schema) package contains classes and funcions for working with table schemas:
     * [Schema](https://godoc.org/github.com/frictionlessdata/tableschema-go/schema#Schema): main entry point, used to validate and deal with schemas
     * [Field](https://godoc.org/github.com/frictionlessdata/tableschema-go/schema#Field): for working with schema fields
     * [Infer](https://godoc.org/github.com/frictionlessdata/tableschema-go/schema#Schema) and [InferImplicitCasting](https://godoc.org/github.com/frictionlessdata/tableschema-go/schema#InferImplicitCasting): for inferring a schema of tabular data
     

# Getting started

## Installation

This package uses [semantic versioning 2.0.0](http://semver.org/). 

### Using dep

```sh
$ dep init
$ dep ensure -add github.com/frictionlessdata/tableschema-go/csv@>=0.1
```

# Examples

Code examples in this readme requires Go 1.8+. You can find more examples in the [examples](https://github.com/frictionlessdata/tableschema-go/tree/master/examples) directory.

```go
package main

import (
	"github.com/frictionlessdata/tableschema-go/csv"
	"github.com/frictionlessdata/tableschema-go/schema"
)

type user struct {
	ID   int
	Age  int
	Name string
}

func main() {
	tab, err := csv.NewTable(csv.FromFile("data_infer_utf8.csv"), csv.SetHeaders("id", "age", "name"))
	if err != nil {
		panic(err)
	}
	sch, err := schema.Infer(tab) // infer the table schema
	if err != nil {
		panic(err)
	}
    sch.SaveToFile("schema.json") // save inferred schema to file
	var users []user
	sch.DecodeTable(tab, &users) // unmarshals the table data into the slice.
}
```
# Documentation

## Table

A table is a core concept in a tabular data world. It represents a data with a metadata (Table Schema). Let's see how we could use it in practice.

Consider we have some local CSV file, `data.csv`:

```csv
city,location
london,"51.50,-0.11"
paris,"48.85,2.30"
rome,N/A
```

To read its contents we use [csv.NewReader](https://godoc.org/github.com/frictionlessdata/tableschema-go/csv#NewReader) to create a table reader and use [csv.FromFile](https://godoc.org/github.com/frictionlessdata/tableschema-go/csv#FromFile) as [Source](https://godoc.org/github.com/frictionlessdata/tableschema-go/csv#Source).

```go
locTable, _ := csv.NewReader(csv.FromFile("data.csv"), csv.LoadHeaders())
locTable.Headers() // ["city", "location"]
iter, _ := locTable.Iter() {    
for iter.Next() {
    fmt.Println(iter.Row())
}
iter.Close()
// [london 51.50,-0.11]
// [paris 48.85,2.30]
// [rome N/A]]
```

So far, locations are string, but it should be geopoints. Also Rome's location is not available but it's also just a N/A string instead of go's zero value. One way to deal with this data is to ask [schema.Infer](https://godoc.org/github.com/frictionlessdata/tableschema-go/schema#Infer) to infer the Table Schema:

```go
locSchema, _ := schema.Infer(tab)
fmt.Printf("%+v", locSchema)
// "fields": [
//     {"name": "city", "type": "string", "format": "default"},
//     {"name": "location", "type": "geopoint", "format": "default"},
// ],
// "missingValues": []
// ...
```

Then we could create a struct and automatically decode the table data into go structs. It is like [json.Unmarshal](https://golang.org/pkg/encoding/json/#Unmarshal), but for table rows. First thing we need is to create the struct which will represent each row.

```go
type Location struct {
    City string
    Location schema.GeoPoint
}
```

Then we are ready to decode the table and process it using Go's values/types.

```go
iter, _ := locTable.Iter() { 
for iter.Next() {
    var loc Location
    locSchema.Decode(iter.Row(), &loc)
    fmt.Println(loc)
}
iter.Close()
// Fails with: "Invalid geopoint:\"N/A\""
```

The problem is that the library does not know that N/A is not an empty value. For those cases, there is a `missingValues` property in Table Schema specification. As a first try we set `missingValues` to N/A in table.Schema.

```go
locSchema.MissingValues = []string{"N/A"}
for iter.Next() {
    var loc Location
    locSchema.Decode(iter.Row(), &loc)
    fmt.Println(loc)
}
iter.Close()
// [{london {51.5 -0.11}} {paris {48.85 2.3}} {rome {0 0}}]
```

And because there are no errors on data reading we could be sure that our data is valid againt our schema. Let's save it:

```go
locSchema.SaveToFile("schema.json")
```