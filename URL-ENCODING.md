# URL-encoding as a solution to rolenames-as-filenames issue

## Background:

* Stating that rolenames must be used as filenames/URLs _without specifying
  how_ leads to bugs
* Client cannot rely on repository not being compromised as protection against
  path traversal attacks: even signed metadata should not lead to arbitrary
  file overwrite
* rolenames and targetpaths are both used to form URLs (and local filenames)
  but are fundamentally different as one forms a part of a single path segment
  and the other is a complete URL path with multiple segments and so _cannot be
  encoded_ without this quite drastically changing TUF "API".
* they are also different in that client application never actively chooses to
  download a roles metadata: metadata URLs are in practice controlled by the
  repository.
* URL-encoding encodes _a lot_, and we are mostly just interested in encoding
  "/" and "\\". However, complete URL-encoding is likely easier to implement
  than a partial one.

## Solution:

URL-encoding the rolename when building a local filename will fix this specific
vulnerability, but I'm proposing a few other rules around related issues (for
example: rolenames can currently coax a client into looking for the metadata in a
completely wrong directory on server, as is visible in the POC).

I consider the below recommendations a _mostly_ backwards-compatible solution:
For ascii-text rolenames nothing changes. For other rolenames the
behaviour was in my opinion undefined: I'd like to defines the behaviour in
more detail. Currently the specification does make requirements on how files
should be named: I think these should be changed to recommendations as outlined
below but this does not affect API compatibility.

As mentioned this specific proof-of-concept will be prevented with only the bolded
step below.

### On rolenames:
* In metadata, rolenames are unrestricted strings
* When forming a metadata download url, the rolename MUST be URL-encoded
* **if rolenames are used as filenames on the client, recommendation is to
  URL-encode rolename when building the filename to avoid path-traversal and
  filesystem compatibility issues**. This fixes the vulnerability
* URL-encoding method is the one used for path segments: restricted
  characters, including "/", should be encoded
* snapshot meta-dict is a gray area (as the keys are currently filenames but
  probably should be rolenames): _is the rolename encoded or not?_

### On target paths:
* repository implementation chooses target path encoding: both "a/b"
  and "a%2Fb" are valid (and separate) target paths and can be used as target
  path in metadata
* client MUST NOT URL-encode when forming download url: repository has
  already encoded what needs encoding
* client is responsible for using targetpath as local filepath: Depending on
  the implementation this may be safe (as the client decides the target paths
  it uses). Still, URL-encoding can be used to avoid path-traversal and
  filesystem compatibility issues if target paths are not known to be safe.


## Examples:
Noteworthy behaviour in the examples has been bolded.

Rolename "role1"
 * client download url "https://localhost/metadata/1.role1.json"
 * recommended local filename "role1.json"

Rolename "../../../file"
 * client download url "https://localhost/metadata/**1...%2F..%2F..%2Ffile**.json"
 * recommended local filename "**..%2F..%2F..%2Ffile**.json"

targetname "path/to/file"
 * client download url "https://localhost/targets/path/to/file"
 * Possible local filename (if known to be safe) "path/to/file"
 * Possible local filename "**path%2Fto%2Ffile**"

targetname "a/b?c=d#e"
 * client download url "https://localhost/targets/a/b?c=d#e"
 * Possible local filename "**a%2Fb%3Fc%3Dd%23e**"

targetname "../../../file"
 * client download url "https://localhost/targets/../../../file"
 * Possible local filename "**..%2F..%2F..%2Ffile**"
