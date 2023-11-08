---
title: "Fast JSON parsing in Go for OpenRTB"
date: 2020-11-22T00:00:00-03:00
tags: ["go", "json"]
categories: ["development"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableHLJS: false # to disable highlightjs
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
UseHugoToc: true
cover:
    alt: "Fast JSON parsing in Go for OpenRTB" # alt text
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/matipan/matipan.github.io/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

The code used for this test can be found [here](https://github.com/matipan/openrtb).

**TL;DR**: when looking into overall performance and usability, `json-iter` is the clear winner. It gives roughly a 4x improvement over `encoding/json` and 1.2x over the second most performant option. It is also extremely easy to use. You simply need to import it, define a global variable (that you can call `json` to make it even easier) and then use it like you would use `encoding/json`.

Services that serve OpenRTB requests are typically under heavy load and have very strict latency constraints. In this scenario a fast json parser is highly desirable. In the Go ecosystem there are many json parsing libraries that claim to have better performance than `encoding/json`. The following list are the libraries that I will consider for this test:

* [easyjson](https://github.com/mailru/easyjson)
* [jsonparser](https://github.com/buger/jsonparser)
* [simdjson-go](https://github.com/simdjson/simdjson)
* [json-iter](https://github.com/json-iter/go)
* [encoding/json](https://pkg.go.dev/encoding/json)


### Performance

Lets go directly to the performance results. Unmarshaling a standard OpenRTB request with a single `imp` object yields the following numbers:

```
goos: darwin
goarch: amd64
pkg: github.com/matipan/openrtb
BenchmarkRequest_UnmarshalJSON/json-iter-12        	  255331	      4730 ns/op	    1328 B/op	      50 allocs/op
BenchmarkRequest_UnmarshalJSON/easyjson-12         	  197110	      5695 ns/op	    1432 B/op	      28 allocs/op
BenchmarkRequest_UnmarshalJSON/jsonparser-12       	  128836	      9360 ns/op	    5528 B/op	      94 allocs/op
BenchmarkRequest_UnmarshalJSON/encoding/json-12    	   70920	     17180 ns/op	    1752 B/op	      49 allocs/op
BenchmarkRequest_UnmarshalJSON/simdjson-go-12      	   46688	     26658 ns/op	  115388 B/op	      48 allocs/op
PASS
ok  	github.com/matipan/openrtb	7.176s
```

This shows that the fastest option of all is `json-iter`. It is 1.2x compared to the second, `easyjson`, 2x compared to `jsonparser` and about 4x compared to `encoding/json`. Surprisingly, `simdjson` is falling way behind being 6x slower than `json-iter`. I have not done an in depth analysis of why simdjson is this slow. If I do, I will post the results here.

Performance-wise `json-iter` is the clear winner.

### Usability

In terms of usability each library requires something different compared to `encoding/json`:

* [easyjson](https://github.com/mailru/easyjson): requires code generation. I'm ok with this to be honest, but it adds yet another command that all developers need to be aware of when modifying your structures.
* [jsonparser](https://github.com/buger/jsonparser): requires a lot of manual parsing to get something working.
* [simdjson-go](https://github.com/simdjson/simdjson): requires even more manual parsing than jsonparser.
* [json-iter](https://github.com/json-iter/go): requires importing a new library and, if you want to use the fastest option, adding a global variable that has to be used for every unmarshal/marshal call.
* [encoding/json](https://pkg.go.dev/encoding/json): the familiar encode/decode and marshal/unmarshal API.

Below you can see an in depth explanation of how to use each of the libraries shown above. But in my opinion usability-wise `json-iter` wins again.

#### Using json-iter

To use json-iter you can simply import the library and define a global variable using the fastest option:

```go
import jsoniter "github.com/json-iterator/go"

var json = jsoniter.ConfigFastest
```

Once you've done that you can use the familiar Marshal/Unmarshal and Encode/Decode API.

#### Using easyjson

To use `easyjson` you need to first install the CLI they provide:

```sh
go get -u github.com/mailru/easyjson/...
```

With this CLI you can generate code that parses JSON into and out of the structure. First, add the following directive to your structure:

```go
//easyjson:json
type Request struct {
	...
}
```

With that you can run this command that will generate functions that match the signature of `encoding/json.Marshaler` and `encoding/json.Unmarshaler`:

```sh
easyjson -byte -pkg
```

#### Using jsonparser

There are many ways to parse a JSON using the `jsonparser` library. I went with an approach that has a lot of code but most of it is boiler plate, which means that adding new fields is relatively straight forward and does not require a lot of modifications. We start by defining all the fields that the OpenRTB request will have and mapping their IDs to the path were it can be found:

```go
type fieldIdx iota

const (
	fieldDevice fieldIdx = iota
	fieldImp
	fieldApp
	fieldId
	fieldAt
	fieldBcat
	fieldBadv
	fieldBapp
	fieldRegs
)

type rtbFieldDef struct {
	idx  fieldIdx
	path []string
}

var (
	reqFields = []rtbFieldDef{
		{fieldDevice, []string{"device"}},
		{fieldImp, []string{"imp"}},
		{fieldApp, []string{"app"}},
		{fieldId, []string{"id"}},
		{fieldAt, []string{"at"}},
		{fieldBcat, []string{"bcat"}},
		{fieldBadv, []string{"badv"}},
		{fieldBapp, []string{"bapp"}},
		{fieldRegs, []string{"regs"}},
	}

	reqPaths = rtbBuildPaths(reqFields)
)

func rtbBuildPaths(fields []rtbFieldDef) [][]string {
	ret := make([][]string, 0, 10)
	for _, f := range fields {
		ret = append(ret, f.path)
	}
	return ret
}
```

Once we've defined the fields for this top level object we can write the parsing function. This function basically iterates over the JSON one key at a time. For each key it finds it tries to map it to one of the keys we defined in the `reqPaths` variable. If it finds a match then it sets the value to the corresponding fields on the structure:

```go
func (r *Request) UnmarshalJSONReq(b []byte) error {
	jsonparser.EachKey(b, func(idx int, value []byte, vt jsonparser.ValueType, err error) {
		r.setField(idx, value, vt, err)
	}, reqPaths...)

	return nil
}

func (data *Request) setField(idx int, value []byte, _ jsonparser.ValueType, _ error) {
	switch fieldIdx(idx) {
	case fieldDevice:
		data.Device = &Device{}
		data.Device.UnmarshalJSONReq(value)
	case fieldImp:
		data.Imps = []*Imp{}
		jsonparser.ArrayEach(value, func(arrdata []byte, dataType jsonparser.ValueType, offset int, err error) {
			imp := &Imp{}
			if err := imp.UnmarshalJSONReq(value); err != nil {
				return
			}
			data.Imps = append(data.Imps, imp)
		})
	case fieldApp:
		data.App = &App{}
		data.App.UnmarshalJSONReq(value)
	case fieldId:
		data.ID = string(value)
	case fieldAt:
		data.At, _ = strconv.ParseInt(string(value), 10, 64)
	case fieldBcat:
		data.BCat = []string{}
		jsonparser.ArrayEach(value, func(arrdata []byte, dataType jsonparser.ValueType, offset int, err error) {
			data.BCat = append(data.BCat, string(value))
		})
	case fieldBadv:
		data.BAdv = []string{}
		jsonparser.ArrayEach(value, func(arrdata []byte, dataType jsonparser.ValueType, offset int, err error) {
			data.BAdv = append(data.BAdv, string(value))
		})
	case fieldBapp:
		data.BApp = []string{}
		jsonparser.ArrayEach(value, func(arrdata []byte, dataType jsonparser.ValueType, offset int, err error) {
			data.BApp = append(data.BApp, string(value))
		})
	case fieldRegs:
		data.Regs = &Regs{}
		data.Regs.UnmarshalJSONReq(value)
	}
}
```

With this you can start parsing the top level object. However, if you look at the `fieldApp` key for example you can see that we are calling an `UnmarshalJSONReq` function of the `App` object. This structure essentially does the same thing than the top level one. It first defines the list of fields that we care about and their corresponding paths and then implements the function that iterates over each key setting the corresponding values:

```go
const (
	fieldAppName fieldIdx = iota
	fieldAppPubId
	fieldAppBundle
	fieldAppLanguage
	fieldAppId
	fieldExtDevUserId
)

var (
	appFields = []rtbFieldDef{
		{fieldAppName, []string{"name"}},
		{fieldAppPubId, []string{"publisher", "id"}},
		{fieldAppBundle, []string{"bundle"}},
		{fieldAppLanguage, []string{"content", "language"}},
		{fieldAppId, []string{"id"}},
		{fieldExtDevUserId, []string{"ext", "devuserid"}},
	}

	appPaths = rtbBuildPaths(appFields)
)

func (a *App) setField(idx int, value []byte, _ jsonparser.ValueType, _ error) {
	switch fieldIdx(idx) {
	case fieldAppName:
		a.Name = string(value)
	case fieldAppPubId:
		a.Publisher.ID = string(value)
	case fieldAppBundle:
		a.Bundle = string(value)
	case fieldAppLanguage:
		a.Content.Language = string(value)
	case fieldAppId:
		a.ID = string(value)
	case fieldExtDevUserId:
		a.Ext.Devuserid = string(value)
	}
}

func (a *App) UnmarshalJSONReq(b []byte) error {
	a.Publisher = &Publisher{}
	a.Content = &AppContent{}
	a.Ext = &AppExt{}
	jsonparser.EachKey(b, func(idx int, value []byte, vt jsonparser.ValueType, err error) {
		a.setField(idx, value, vt, err)
	}, appPaths...)

	return nil
}
```

If you want to add a new object to the structure you need to implement all this boiler plate. However, if you just want to add a new field to an existing object the change is relatively straight forward. This is why `jsonparser` in terms of usability is worst than `json-iter` but still better than `simdjson-go`.

#### Using simdjson-go

There is a lot of code involved when parsing JSON with simdjson-go. Here I went with an approach that basically requires you to add more parsing code every time you add a new field. We could implement this with reflection like `encoding/json` does but that would hurt the performance even more.

Implementing a parser for `simdjson` requires one to start iterating over the tape that the library generated and map each field according to its type. We start by parsing the top level object and identifying it as a `simdjson.TypeObject`:
```go
func (r *Request) UnmarshalJSONSimd(b []byte) error {
	parsed, err := simdjson.Parse(b, nil)
	if err != nil {
		return err
	}

	var (
		iter = parsed.Iter()
		obj  = &simdjson.Object{}
		tmp  = &simdjson.Iter{}
	)

	for {
		typ := iter.Advance()

		switch typ {
		case simdjson.TypeRoot:
			if typ, tmp, err = iter.Root(tmp); err != nil {
				return err
			}

			switch typ {
			case simdjson.TypeObject:
				if obj, err = tmp.Object(obj); err != nil {
					return err
				}
				return r.parse(tmp, obj)
			}
		default:
			return nil
		}
	}
}
```

Within the `simdjson.TypeObject` switch we can start parsing the OpenRTB request, but the request has many internal objects and each of them can have a different type. This means that for every one of those types we need to parse it separately. To keep this example brief we will only parse two types, `Object` and `Array`:
```go
func (r *Request) parse(tmp *simdjson.Iter, obj *simdjson.Object) error {
	arr := &simdjson.Array{}
	for {
		name, t, err := obj.NextElementBytes(tmp)
		if err != nil {
			return err
		}

		if t == simdjson.TypeNone {
			break
		}

		switch t {
		case simdjson.TypeObject:
			if err := r.parseObject(name, tmp); err != nil {
				return err
			}
		case simdjson.TypeArray:
			if _, err := tmp.Array(arr); err != nil {
				return err
			}
			if err := r.parseArray(name, tmp, arr); err != nil {
				return err
			}
		}
	}

	return nil
}
```

Lets double click on the parsing of an `Object`. Within an OpenRTB request we have many different objects, like `device` and `app`. Each object will require its own parsing function that will map each available key to the corresponding value of the structure. For example, parsing the `device` object would look something like this:
```go
func (d *Device) parse(iter *simdjson.Iter, obj *simdjson.Object) error {
	for {
		name, t, err := obj.NextElementBytes(iter)
		if err != nil {
			return err
		}

		if t == simdjson.TypeNone {
			return nil
		}

		switch t {
		case simdjson.TypeInt:
			n, err := iter.Int()
			if err != nil {
				return err
			}
			switch {
			case bytes.Compare(name, hKey) == 0:
				d.H = n
			case bytes.Compare(name, wKey) == 0:
				d.W = n
			case bytes.Compare(name, dtKey) == 0:
				d.DeviceType = n
			case bytes.Compare(name, ctKey) == 0:
				d.ConnectionType = n
			}
		case simdjson.TypeString:
			b, err := iter.StringBytes()
			if err != nil {
				return err
			}
			switch {
			case bytes.Compare(name, ipKey) == 0:
				d.IP = string(b)
			case bytes.Compare(name, uaKey) == 0:
				d.UA = string(b)
			case bytes.Compare(name, osKey) == 0:
				d.OS = string(b)
			case bytes.Compare(name, osvKey) == 0:
				d.OSV = string(b)
			case bytes.Compare(name, ifaKey) == 0:
				d.IFA = string(b)
			case bytes.Compare(name, hwvKey) == 0:
				d.HWV = string(b)
			case bytes.Compare(name, modelKey) == 0:
				d.Model = string(b)
			case bytes.Compare(name, dntKey) == 0:
				d.DNT = string(b)
			case bytes.Compare(name, langKey) == 0:
				d.Language = string(b)
			}
		}
	}
}
```

In this example you can see how tedious it would be to add a new field or top level object. Which is why from a usability point of view, `simdjson-go` is way behind.

# Conclusion
When looking into overall performance and usability, `json-iter` is the clear winner. It gives roughly a 4x improvement over `encoding/json` and 1.2x over the second most performant option. It is also extremely easy to use. You simply need to import it, define a global variable (that you can call `json` to make it even easier) and then use it like you would use `encoding/json`. On top of that, the community around `json-iter` seems to be really active. And this library is supported on many different languages.
