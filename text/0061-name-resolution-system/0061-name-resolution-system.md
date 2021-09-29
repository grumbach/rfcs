# NRS

The Name Resolution System (aka NRS) is a mechanism to look up data related to any name (human readable string like `maidsafe`) on the SAFE network. It's the SAFE network's own variant of the DNS with a couple of differences, one being that is it decentralised.

NRS can often be confusing at first, I'll try my best to explain what I understand of it.

This proposal looks to unify the existing descriptions of it, as well as suggesting simplifications to the existing NRS implementation.

## Nomenclature

-  `Name Resolution System` (NRS) instead of `DNS` to avoid confusion and clarify this is a SAFE network term, and that it relates to `Public Names`.
	- Here the DNS terminology for `domain` is equivalent to a SAFE `Public Name`.
	- What is called `top level domain` aka `tld` in DNS is referred to as `Top Name`
	- What is called `sub domains` in DNS are referred to as `Sub Names`.
- `Xorname` are raw addresses in the SAFE network
- `XorUrl` refers to a url generated from a `Xorname` and with extra information (such as datatype) encoded within it
- `NRS url` refer to a human-friendly url with a `Public Name`
- `Url` indicates a URL that is either a `XorUrl` or a NRS url

```
safe://<subName>.<topName>/path/to/whatever?var=value
       |-----------------|
            Public Name
```			

```
Example:
  Public Name -->  a.myname
  Top Name    -->  myname
  Sub Names   -->  a
```

```
Example without subnames:
  Public Name -->  myname
  Top Name    -->  myname
  Sub Names   -->  <empty>
```

## How it works

NRS translates NRS urls (like `myname` or `blog.maidsafe`) into `XorUrl`s.
For every registered Top Names, NRS creates a `NrsMapContainer` on the network at: `XorName::from_content(top_name.as_bytes())`. This way, when someone wants to access the Top Name `myname`, NRS can just lookup informations stored at the hash of it on the network.

```
hash("topname") -> XorName -> NrsMapContainer
```

Top Names and their subnames are stored together in the Top Name's `NrsMapContainer` which contains versioned map with an entry (`XorUrl`) for every registered subname.

```
"blog.apple"  -> NrsMapContainer for "apple"  -> NrsMap{...} -> XorUrl
"google"      -> NrsMapContainer for "google" -> NrsMap{...} -> XorUrl
"shop.amazon" -> NrsMapContainer for "amazon" -> NrsMap{...} -> XorUrl
```

## NrsMapContainers are Registers

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
```

Registers use hashes to refer to entries in the DAG, this is why versions in safe `Url`s use `VersionHash`.

> Note that on every change in an entry in the NrsMap, a new version is appended to the DAG

## NrsMap

The `NrsMap` maps a `TopName` and its `SubNames` to their respective `XorUrl`.

```
| SubName       | Public Name      | XorUrl Value |
|---------------|------------------|--------------|
| None          | "example"        | "safe://eg1" |
| "sub"         | "sub.example"    | "safe://eg2" |
| "sub.sub"     | "sub.sub.example"| "safe://eg3" |
```

The current implementation of NrsMap is an attempt to use the RDF format:

```rust
pub(crate) type SubName = String;
pub(crate) type DefinitionData = BTreeMap<String, String>;

pub enum SubNameRdf {
    Definition(DefinitionData),
    SubName(NrsMap),
}

impl SubNameRdf {
    fn get(&self, key: &str) -> Option<String> {
        match self {
            SubNameRdf::SubName { .. } => Some(self.get(key)?),
            _ => None,
        }
    }
}

// The default for a sub name can be unset (NotSet), reference to the same mapping as
// another existing sub name (ExistingRdf), or just a different mapping (OtherRdf)
pub enum DefaultRdf {
    NotSet,
    ExistingRdf(SubName),
    OtherRdf(DefinitionData),
}

// Each PublicName contains metadata and the link to the target's XOR-URL
pub type SubNamesMap = BTreeMap<SubName, SubNameRdf>;

// To use for mapping sub names to PublicNames
pub struct NrsMap {
    pub sub_names_map: SubNamesMap,
    pub default: DefaultRdf,
}
```

Below is my proposition to simplify the `NrsMap` (and the code using it!).

```rust
pub(crate) type SubName = String;

pub struct NrsMap {
    pub map: BTreeMap<SubName, XorUrl>,
}
```

> Example: The NrsMap for `"maidsafe"` stored on the `NrsMapContainer` (a `Register`) at `hash("maidsafe")` containing a couple of subnames.

|Public Name         | SubName (`BtreeMap` Key)| XorUrl (`BtreeMap` Value)|
|--------------------|-------------------------|--------------------------|
|`"maidsafe"`        | `""`                    | `"safe://xor1"`          |
|`"sub.maidsafe"`    | `"sub"`                 | `"safe://xor2"`          |
|`"sub.sub.maidsafe"`| `"sub.sub"`             | `"safe://xor3"`          |

## NRS API

As the CLI describes it :

```
add       Add a subname to an existing NRS name, or updates its link if it already exists
create    Create a new public name
remove    Remove a subname from an NRS name
```

This is the current NRS API:

```rust
fn nrs_map_container_create(&mut self, name: &str, link: &str, default: bool, hard_link: bool, dry_run: bool)
-> Result<(XorUrl, ProcessedEntries, NrsMap)>; // XorUrl containing a VersionHash

// the input name refers to a Public Name
fn nrs_map_container_add(&self, name: &str, link: &str, default: bool, hard_link: bool, dry_run: bool)
-> Result<(VersionHash, XorUrl, ProcessedEntries, NrsMap)>;

// the input name refers to a Public Name
fn nrs_map_container_remove(&self, name: &str, dry_run: bool)
-> Result<(VersionHash, XorUrl, ProcessedEntries, NrsMap)>;

fn nrs_map_container_get(&self, url: &str)
-> Result<(VersionHash, NrsMap)>;
```

I would like to simplify / refactor it to

```rust
fn nrs_map_container_create(&self, top_name: &str, link: &XorUrl, dry_run: bool)
-> Result<XorUrl>; // XorUrl containing a VersionHash

fn nrs_map_container_associate(&self, public_name: &str, link: &XorUrl, dry_run: bool)
-> Result<XorUrl>; // XorUrl containing a VersionHash

fn nrs_map_container_remove(&self, public_name: &str, dry_run: bool)
-> Result<XorUrl>; // XorUrl containing a VersionHash

fn nrs_map_container_get(&self, public_name: &str)
-> Result<XorUrl>; // XorUrl containing a VersionHash
```

- More rigorous API
- `nrs_map_container_get` doesn't expose the underlying `NrsMap` and does the Job NRS is supposed to do

## Other Points

- What type NRS should return? It currently is `XorUrl` but as we discussed before the distinction between `Url`, `XorUrl`, `NrsUrl` can often be confusing at first, and we tried to converge to a single type `Url`. But having NRS return `Url`s that WOULD themselves need to go through NRS to be resolved seems like a big drawback. @danda also pointed out that using `XorUrl` was probably too much for just a `XorName` + `VersionHash`.


Hope this was useful and helped some better understand NRS! If we like this proposition we could make it an RFC. There are probably mistakes, please help me correct them.

What do you think?
