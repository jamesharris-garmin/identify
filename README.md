identify
========

[![Build Status](https://dev.azure.com/asottile/asottile/_apis/build/status/pre-commit.identify?branchName=master)](https://dev.azure.com/asottile/asottile/_build/latest?definitionId=67&branchName=master)
[![Azure DevOps coverage](https://img.shields.io/azure-devops/coverage/asottile/asottile/67/master.svg)](https://dev.azure.com/asottile/asottile/_build/latest?definitionId=67&branchName=master)
[![pre-commit.ci status](https://results.pre-commit.ci/badge/github/pre-commit/identify/master.svg)](https://results.pre-commit.ci/latest/github/pre-commit/identify/master)
[![PyPI version](https://badge.fury.io/py/identify.svg)](https://pypi.python.org/pypi/identify)

File identification library for Python.

Given a file (or some information about a file), return a set of standardized
tags identifying what the file is.

## Installation

`pip install identify`

## Usage
### With a file on disk

If you have an actual file on disk, you can get the most information possible
(a superset of all other methods):

```python
>>> from identify import identify
>>> identify.tags_from_path('/path/to/file.py')
{'file', 'text', 'python', 'non-executable'}
>>> identify.tags_from_path('/path/to/file-with-shebang')
{'file', 'text', 'shell', 'bash', 'executable'}
>>> identify.tags_from_path('/bin/bash')
{'file', 'binary', 'executable'}
>>> identify.tags_from_path('/path/to/directory')
{'directory'}
>>> identify.tags_from_path('/path/to/symlink')
{'symlink'}
```

When using a file on disk, the checks performed are:

* File type (file, symlink, directory, socket)
* Mode (is it executable?)
* File name (mostly based on extension)
* If executable, the shebang is read and the interpreter interpreted


### If you only have the filename

```python
>>> identify.tags_from_filename('file.py')
{'text', 'python'}
```


### If you only have the interpreter

```python
>>> identify.tags_from_interpreter('python3.5')
{'python', 'python3'}
>>> identify.tags_from_interpreter('bash')
{'shell', 'bash'}
>>> identify.tags_from_interpreter('some-unrecognized-thing')
set()
```

### As a cli

```
$ identify-cli --help
usage: identify-cli [-h] [--filename-only] path

positional arguments:
  path

optional arguments:
  -h, --help       show this help message and exit
  --filename-only
```

```console
$ identify-cli setup.py; echo $?
["file", "non-executable", "python", "text"]
0
$ identify-cli setup.py --filename-only; echo $?
["python", "text"]
0
$ identify-cli wat.wat; echo $?
wat.wat does not exist.
1
$ identify-cli wat.wat --filename-only; echo $?
1
```

### Identifying LICENSE files

`identify` also has an api for determining what type of license is contained
in a file.  This routine is roughly based on the approaches used by
[licensee] (the ruby gem that github uses to figure out the license for a
repo).

The approach that `identify` uses is as follows:

1. Strip the copyright line
2. Normalize all whitespace
3. Return any exact matches
4. Return the closest by edit distance (where edit distance < 5%)

To use the api, install via `pip install identify[license]`

```pycon
>>> from identify import identify
>>> identify.license_id('LICENSE')
'MIT'
```

The return value of the `license_id` function is an [SPDX] id.  Currently
licenses are sourced from [choosealicense.com].

[licensee]: https://github.com/benbalter/licensee
[SPDX]: https://spdx.org/licenses/
[choosealicense.com]: https://github.com/github/choosealicense.com

## How it works

A call to `tags_from_path` does this:

1. What is the type: file, symlink, directory? If it's not file, stop here.
2. Is it executable? Add the appropriate tag.
3. Do we recognize the file extension? If so, add the appropriate tags, stop
   here. These tags would include binary/text.
4. Peek at the first X bytes of the file. Use these to determine whether it is
   binary or text, add the appropriate tag.
5. If identified as text above, try to read and interpret the shebang, and add
   appropriate tags.

By design, this means we don't need to partially read files where we recognize
the file extension.
