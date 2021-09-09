# Rolename use as part of filenames in TUF

There are several connected issues with using rolenames as filename parts in
TUF implementations. The most pressing one is that the reference clients
(`tuf/client/` and `tuf/ngclient/`) both have a path traversal vulnerability
that in the worst case can result in near-arbitrary file overwrites
on the client device, but I believe we should look at the wider issue as well.


## Path traversal in client file creation

Attacker with ability to affect the rolenames of delegated targets in the
repository can overwrite files ending in ".json" anywhere on client device.
Client is only required to do `get_one_valid_targetinfo()` that ends up
processing the delegation in question. 
https://github.com/jku/path-traversal-poc contains details and an example
repository.
 
The issue is mitigated by a few facts:
 * implementations generally don't allow arbitrary rolename selection (?)
 * the attack does require the ability to A) insert new metadata for the 
   path-traversing role and B) get the role delegated by an existing targets
   metadata
 * the written file content is heavily restricted since it needs to be a valid,
   signed targets file. The file extension is always .json.

It is a real issue however:
 * overwriting almost arbitrary files is ... bad
 * PEP-480-like implementations are likely to lead to rolenames being less
   controlled by repository and more controlled by individual targets
   maintainers

The issue has revealed itself over time so is pretty much documented publicly
already: https://github.com/theupdateframework/python-tuf/issues/1527.


## Wider issue with using rolenames in filenames

Arbitrary rolenames are probably a valuable feature as documented by asraa in
https://github.com/theupdateframework/python-tuf/issues/1527#issuecomment-903871213.
The rest of this document takes this as given but an alternative here would be
to limit rolenames.

Using rolenames as filenames in the specification and the implementations
currently has multiple problems:
* preventing path traversal requires very careful software engineering: this is
  not ideal
* filesystems have requirements for valid filenames (these differ between
  filesystems). These requirements might mean the rolename must be encoded
  somehow. Specifying an encoding that works for everyone may be difficult
* some implementations don't use files at all: requiring specific filenames or
  encodings in this case is weird
* the "meta" dictionary currently uses filenames as keys: This seems like a
  mistake and leads to further problems if the filenames have to be encoded in
  any way -- the encoding could be a client implementation detail but the keys
  in meta cannot be that


### Ideal solution

An ideal solution (that disregards backwards compatibility) might include:
* rolename use in URLs, and any required encoding must be specified in the
  specification
* targetpath use in URLs, and any required encoding must be specified in the
  specification
* behaviour of rolenames in "meta" should be defined (encoded or not?)
* specification should *not* define how client or repository must store
  metadata: this is an implementation detail.
* spec or a "implementer notes" document should give advice on how to store
  local metadata, including a best known method for filename creation
* spec or a "implementer notes" document should give advice on how to store
  target files locally


### Practical solution

1. First, client implementations should defend against path traversal -- possibly
   by preventing rolenames with "/" or by defining an encoding method that can be
   implemented easily and in a mostly backwards-compatible manner.
2. Targetpath handling should be reviewed for similar issues, path traversal
   should be prevented (although in this case we can't just prevent "/")
3. The issue with using rolenames and targetpaths in local paths should described
   in the specification
5. Finally, some sort of path towards the "Ideal solution" should be deviced.
   I have a potential url-encoding based solution which I will document shortly.

