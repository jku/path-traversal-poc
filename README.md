# TUF client metadata path-traversal proof of concept

This repository will fool both python-tuf client implementations to write 
(overwrite) a file outside of the client metadata store when
`get_one_valid_targetinfo()` is called. For this specific example, the
file will be written one level above the metadata directory but likely
any arbitrary filepath (with .json extension) is possible.

### Examples assuming the repository root is hosted at `http://localhost:8000/`:

#### Legacy Updater
```python
# Directories ./metadata/current/ and ./metadata/previous/ must exist.
# Initial root must be in ./metadata/current/root.json.

import tuf.settings
from tuf.client.updater import Updater

tuf.settings.repositories_directory = "./"
repo_mirrors = {
    "mirror1": {
        "url_prefix": "http://localhost:8000",  "metadata_path": "", "targets_path": ""
    }
}
updater = Updater("", repo_mirrors)
updater.refresh()
updater.get_one_valid_targetinfo("any target name here")

# Local metadata dir is ./metadata/current/ but ./metadata/ now contains
# path-traversal-test.json
```

#### ngclient Updater

```python
# Initial root must be in ./root.json.

from tuf.ngclient import Updater
updater = Updater("./", ""http://localhost:8000/", "http://localhost:8000/")
updater.refresh()
updater.get_one_valid_targetinfo("any target name here")

# Local metadata dir is ./ but ../ now contains path-traversal-test.json
```

### More info

Thoughts on the issues TUF/python-tuf has with rolenames being used in filenames/URLS:  [ISSUE-DESCRIPTION.md](ISSUE-DESCRIPTION.md)
Potential solution: [URL-ENCODING.md](URL-ENCODING.md) -- note that fixing the path traversal is easy (_encode rolename before using as filename_), but the document outlines some related future improvements as well
