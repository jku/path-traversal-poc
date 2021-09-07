# Rolename use as part of filenames in TUF

There are several connected issues with using rolenames as filename parts in
TUF implementations. The most pressing one is that the reference clients
(tuf/client/ and tuf/ngclient/) both have a path traversal vulnerability, but
I believe we should look at the wider issue as well.


## Path traversal in client file creation

Attacker with ability to affect the rolenames of delegated targets in the
repository can overwrite files ending in ".json" anywhere on client device.
Client is only required to do `get_one_valid_targetinfo()` that ends up
processing the delegation in question. 
 
The issue is mitigated by a few facts:
 * implementations generally don't allow arbitrary rolename selection (?)
 * the attack does require the ability to A) insert new metadata for the role
   and B) get the role delegated by an existing targets metadata
 * the written file content is heavily restricted since it needs to be a valid,
   signed targets file. The file extension is always .json.

It is a real issue however:
 * overwriting almost arbitrary files is ... bad
 * PEP-480 like implementations are likely to lead to rolenames being less
   controlled by repository and more controlled by individual targets
   maintainers

The issue has revealed itself over time so is pretty much documented publicly
already: https://github.com/theupdateframework/python-tuf/issues/1527. The
specific vulnerability is documented in a private repository:
https://github.com/jku/path-traversal-poc


## Wider issue with using rolenames in filenames

Arbitrary rolenames are probably a valuable feature as documented by asraa in
https://github.com/theupdateframework/python-tuf/issues/1527#issuecomment-903871213.
The rest of this document takes this as given but an alternative here is to limit
rolenames.

Using rolenames as filenames in the specification and the implementations has
multiple problems:
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
* "meta" should use rolename, not filename, as key
* specification should not specify how client or repository must store
  metadata: this is an implementation detail.
* A "implementer notes" document should give advice on how to store metadata
* The advice should specify a best known method for filename creation (TBD):
  Some options for the best known method could be:
  * use rolename hash as filename
  * use a URL-encoded rolename as filename (this requires specifying some
    details about the encoding)


### Practical solution

First, client implementations should defend against path traversal -- possibly
by preventing rolenames with "/". The issues around rolenames as filenames
should probably be mentioned in the spec.

Then, some sort of path towards the "Ideal solution" should be deviced. A
fairly backwards compatible, but ugly, solution might be to keep using "meta"
as is but document that the keys are not "real" filenames but just 
rolename+extension (which is already true since "role1.json" is a valid key
for meta but the non-versioned file might not even exist in a
_consistent_snapshot_ repository).
