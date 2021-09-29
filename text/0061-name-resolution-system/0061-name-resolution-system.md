# Name Resolution System: a DNS for the SAFE Network

- Status: proposed
- Type: enhancement
- Start Date: 28/09/2021
- Supersedes: [RFC 0052](../0052-RDF-for-public-name-resolution/0052-RDF-for-public-name-resolution.md) and [RFC 0002](../text/0002-name-service/0002-name-service.md)

## Summary

The Name Resolution System (aka NRS) is a mechanism to look up data related to any name (human readable string like `maidsafe`) on the SAFE network. It's the SAFE network's own variant of the DNS with a couple of differences, one being that is it decentralised.

This proposal looks to scope NRS to the domain/IP (also referred to as: url base) part of an URL in a similar fashion to DNS but taking advantage of the data encoding capabilities of SAFE URLs, enabling data type and other informations to be directly encoded into the URL base. In this proposition, we chose to use vague-to-specific order for domain names and subnames (ie: `google.maps` instead of the current `maps.google`).

## Conventions

-  `Name Resolution System` (NRS) instead of `DNS` to avoid confusion and clarify this is a SAFE network term, and that it relates to `Public Names`.
	- Here the DNS terminology for `domain` is equivalent to a SAFE `Public Name`.
	- What is called `top level domain` aka `tld` in DNS is referred to as `Top Name`
	- What is called `sub domains` in DNS are referred to as `Sub Names`.
- `Xorname` are raw addresses in the SAFE network
- `XorUrl` refers to a url generated from a `Xorname` and with extra information (such as datatype) encoded within it
- `XorUrlBase` refers to the domain/IP part of a XorUrl (which can include information like DataType but excludes path or query variables)
- `NRS url` refer to a human-friendly url with a `Public Name`
- `Url` indicates a URL that is either a `XorUrl` or a NRS url

> Note that the order for top names and subnames is vague-to-specific `safe://topname.then.subnames`

```
safe://<topName>.<subName>/path/to/whatever?var=value
       |-----------------|
            Public Name
```			

```
Example:
  Public Name -->  myname.a
  Top Name    -->  myname
  Sub Names   -->  a
```

```
Example without subnames:
  Public Name -->  myname
  Top Name    -->  myname
  Sub Names   -->  <empty>
```

## Motivation

URLs consist of different parts that are dealt with by different layers of a system.

```
safe://  topname.subname  /path?query=parameters
  ^             ^                 ^
  |             |                 |
protocol      domain       application variables
```

- The first part is the protocol (scheme) part which informs us on the browser/network in which we should use this URL.
- The second part is the domain part of the URL which tells us which application/owner we are dealing with.
- The third part contains specific information for the application to use, it could be anything, that's up to the application:
	- a path to a file
	- a route in a specific application's API
	- ...

This proposal aims to scope NRS to the domain (also referred to as base URL) part of the URL, in an attempt to enforce the separation of concerns in URL managing on SAFE.

The other proposed change is to reverse the traditional order of domains and subdomains to a more logical order that's consistent with the order of the path part of the url.

```
safe://maps.google/mars/surface    (current order)
             vs
safe://google.maps/mars/surface    (proposed order)
```

This order also makes long URLs safer as the owner is easy to find (leftmost part after url scheme). This helps to strengthen trust and security for the SAFE Network users.

> Below, the link in your daily fishing email becoming the remains of a deprecated business model.

```
http://google.secure.access.scam.net/login/google/its/safe/dont/worry?url=google.com/login
          vs
safe://scam.google.secure.access/login/google/its/safe/dont/worry?url=google.com/login
```

## Detailed design

In the same way DNS translates domain names to IP addresses, NRS translates `public names` (human readable names) to `XorUrlBase` (encoded `XorName` along with data type information), ignoring the path and query parameters part of the URL. NRS can also map a `public name` to another `public name`.

```sh
# NRS can map from a human name to a XorUrlBase
safe://<human name>/whatever/path?whatever=var
safe://<XorUrlBase>/whatever/path?whatever=var

# or to another human name
safe://<human name>/whatever/path?whatever=var
safe://<other name>/whatever/path?whatever=var
```

#### XorUrlBase is the base part of a XorUrl

```
safe://  topname.subname  /path?query=parameters
  ^             ^                 ^
  |             |                 |
scheme      base part       application data
```

`Xorname`s are the equivalent of IP addresses in a SAFE url, since XorNames can't be used directly inside a URL they need to be encoded first. The good part here is that SAFE Network Urls can encode additional information, such as Data Type, into the Url base part. This is what makes SAFE Urls self-describing as they know the DataType they are linking to. This is the reason why we chose for this proposal to map to `XorUrlBase` instead of just `Xorname` like a regular DNS would do.

```sh
safe://1234567890/path?var=s  (this is a complete XorUrl)
safe://1234567890             (this is its XorUrlBase)
```

> Just like DNS, the resolution stops if the same name is encountered twice. To prevent infinite recursion for names pointing to themselves.

#### Hash(TopName) for NRS information

For every registered Top Names, NRS creates a `NrsMapContainer` on the network at: `XorName::from_content(top_name.as_bytes())`. This way, when someone wants to access the Top Name `myname`, NRS can just lookup informations stored at the hash of it on the network.

```
hash("topname") -> XorName -> NrsMapContainer
```

Top Names and their subnames are stored together in the Top Name's `NrsMapContainer` which contains versioned map with an entry (`XorUrl`) for every registered subname.

```
"apple.blog"  -> NrsMapContainer for "apple"  -> NrsMap{...} -> XorUrl
"google"      -> NrsMapContainer for "google" -> NrsMap{...} -> XorUrl
"amazon.shop" -> NrsMapContainer for "amazon" -> NrsMap{...} -> XorUrl
```

#### NrsMapContainers are Registers

The `NrsMapContainer` is a `Register` CRDT (aka Merkle DAG) on the network that contains an `NrsMap` structure (more on this below). Registers have a nice property that lets them keep track of older values in a DAG. The entry at the end of the DAG is the current value. Note that there can sometimes be multiple latest entries when concurrent writes to a `Register` are performed.

```
Simple example:

#hash1               #hash2               #hash3
[NrsMap version1] -> [NrsMap version2] -> [NrsMap version3]
```
```
Example with a DAG fork (2 latest versions):

#hash1               #hash2
[NrsMap version1] -> [NrsMap version2]
               \
                \    #hash3
                 --> [NrsMap version3]

How the DAG fork can be resolved:

#hash1               #hash2                 #hash4
[NrsMap version1] -> [NrsMap version2] -> [NrsMap version4]   
            \                              ^
             \    #hash3                  /
              --> [NrsMap version3] -----
```

Registers use hashes to refer to entries in the DAG, this is why versions in safe `Url`s use `VersionHash`.

> Note that on every change in an entry in the NrsMap, a new version is appended to the DAG

#### NrsMap is just a Map

The `NrsMap` maps a `TopName` and its `SubNames` to their respective `XorUrlBase`.

```
| SubName       | Public Name      | XorUrlBase Value |
|---------------|------------------|------------------|
| None          | "example"        | "safe://eg1"     |
| "sub"         | "example.sub"    | "safe://eg2"     |
| "sub.sub"     | "example.sub.sub"| "safe://eg3"     |
```

```rust
pub(crate) type SubName = String;

pub struct NrsMap {
    pub map: BTreeMap<SubName, XorUrlBase>,
}
```

> Example: The NrsMap for `"maidsafe"` stored on the `NrsMapContainer` (a `Register`) at `hash("maidsafe")` containing a couple of subnames.

|Public Name         | SubName (`BtreeMap` Key)| XorUrlBase (`BtreeMap` Value)|
|--------------------|-------------------------|------------------------------|
|`"maidsafe"`        | `""`                    | `"safe://xor1"`              |
|`"maidsafe.sub"`    | `"sub"`                 | `"safe://xor2"`              |
|`"maidsafe.sub.sub"`| `"sub.sub"`             | `"safe://xor3"`              |

## Drawbacks

- NRS cannot be used as an aliasing system (from name to URL) as it cannot map to complete URLs (since it is scoped to the URL base like DNS).
- Subnames have only one layer so everything after the first `'.'` is considered as a single subname `safe://google.maps.mars.surface` has one subname `maps.mars.surface`.

## Alternatives

#### What could NRS also be?

Two different alternative where discussed for NRS :
- Keeping the current URL aliasing system, that merges URLs upon resolution (path, query variables, …)

```sh
# associate name with Url
safe://other-name = safe://42543476847723111324657855432413145465766/over/here
safe://human-name = safe://other-name/other/path?other=var

# resolution that merges the Urls
safe://human-name/some/path?some=var
-> safe://other-name/other/path/some/path?other=var&some=var
-> safe://42543476847723111324657855432413145465766/over/here/other/path/some/path?other=var&some=var
```

- NRS as a URL shortener

```sh
# associate public name with a long Url
safe://human-name = safe://42543476847723111324657855432413145465766/some/path?some=var
```

> The second option could actually be used as a separate app on SAFE network without conflicting with NRS.

#### Subnames order, or no no subnames at all?

One other discussion was wether we wanted subnames or not, and in what order do we wanted them?
- changing the order of subnames can be confusing for users coming from the web
- a different separator character for subnames could be used to avoid confusion
- having no subnames at all avoids that problem altogether

## Unresolved questions

- Versioning with hashes makes URLs long and human unfriendly
- Versioning is currently not encodable/includable in the URL base, one could think of having it as an optional field, one way could be to use the unused port field in SAFE url `safe://urlbase:<version>/`, but this needs more thoughts.
