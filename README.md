# atomic-validation-criteria

These comma-separated values (`CSV`) files contain arguments used to run the tests, and the expected telemetry.  The [atomic-harness](https://github.com/secureworks/atomic-harness) and related tools use the criteria to validate that the EDR software is providing the right telemetry.

## Format
Each CSV file is a collection of Atomic Test Validation Criteria.  They have the following rows.
 - Test header (e.g. `T1101.003,linux,2,some test name`)
 - optional `ARG`,`!!!`,`FYI` rows
 - one or more expected event rows `_E_`
 - lines can be empty
 - lines starting with `#` are comments and are ignored

### Example

```
T1105,linux,6,sftp remote file copy (pull)
ARG,local_file,/tmp/sftpdest-T1105_6
ARG,remote_host,$SERVER[rsync_ssh].addr
ARG,remote_port,$SERVER[rsync_ssh].port
ARG,username,$SERVER[rsync_ssh].username
ARG,remote_path,/tmp/
_E_,Process,cmdline~=sftp -P #{remote_port} #{username}@#{remote_host}:#{remote_path}
_E_,Netflow,TCP:*->#{remote_host}:#{remote_port}
_E_,File,WRITE,path=#{local_file}
```

## Test Header Row
The test header row needs to map directly to an Atomic Test.  It has the following fields
- Technique (e.g. `T1027.001` or `T1003`)
To make it easy for you to get started with GitLab, here's a list of recommended next steps.
- Platform - `linux`,`macos`,`windows`.  Some tests are marked to work for multiple platforms (usually linux and macos together).  You will need separate validation criteria for each platform.
- TestNum - a 1-based index for the test.  The first test for technique `T1XXX` has TestNum == 1, the second will be 2, regardless of platform the tests are on.  If the atomics file for `T1027.001.yaml` contains 3 tests, one for each platform macos,windows,linux. Their numbers will be 1,2,3, respectively.
- TestName - Copy this from the atomic `name` line

## `ARG` row
Every time a single Atomic Test is run, the harness passes the specifics in a JSON file.  If there are `ARG` rows present, the runner `goartrun` will use these values when running the test script.  Obviously, to use an ARG, it must exist in the Atomic Test `yaml` file.

An ARG row has the following columns:
 - `ARG`
 - input_argument name
 - value

 The Atomic yaml files allows for typed values, but here we just treat them as strings.

Apart from static values that are substituted as-is, there are some special values that the harness can provide:
- static value
- `$hostname` : name of the host
- `$ipaddr4` : IPV4 address of host if present
- `$ipaddr6` : IPV6 address of host, if present
- `$username` : The username of current user (before running sudo for the harness)
- `$subnet` : First 3 sections of address.  If `$ipaddr4 == "10.0.0.5`, the `$subnet` will be string `"10.0.0"`
- `$gateway` : This is `$subnet` + ".1".  e.g. `"10.0.0.1"`
- `$netif` : guess of the primary physical network interface

Values from server config file passed to harness:

- `$SERVER[id].addr` : Hostname (if present) or the IPV4 or IPV6 addr of host `id`
- `$SERVER[id].port` : If provided, specifies port number for service
- `$SERVER[id].username` : If needed, username for service
- `$SERVER[id].password` : If needed, password for service

### ERROR / WARNING `!!!` row
Currently, if a test has a `!!!` row, it will be skipped.  Use this when:
 - test needs to be fixed.  For example, when test hangs, needs user inputs, is missing dependency steps,etc.
 - test is destructive and may delete important files
 - test requires a target server that we don't have setup yet
 - if test is manual, it will be skipped, so you shouldn't need this

### `FYI` row
The text on an FYI row will be printed out by the harness when the test runs.  These can serve as reminders or comments.

### `_E_` expected event row
The expected event rows have the following fields
- `_E_`
- Type (`Process,File or FileMod,Netflow,Auth,PTrace,etc.`).  Assume that the type is case-senstive for now
- `SubType` : **ONLY** for `File` or `Netflow`
- one of more `Field Match expressions` separated by commas

**NOTE**: If an expression value contains a space, use double-quotes around the CSV column value.  For example:
```
_E_,File,CHMOD,"path~=testdirwithspaceend /init "
```

### Field Match Expressions
The expressions are very simple and have a `name` `operator` `value` format.

Operators:
 - `=` String equals comparison
 - `~=` String contains
 - `*=` regex match

### Expression variable substitution

The `ARG` values can contain `#{name}` substitution for `ARG` variables.  This makes it easy to copy the commands right from `Atomic` `yaml` files into an `_E_,Process` row.  The values will be substituted before passing the criteria to the `telemetry` tool for validation.

### `_E_,Process` row
The process row is mainly for matching `cmdline` field.

- `cmdline` : created by putting the `Process` args together. **NOTE**: args that contain spaces will be wrapped in double-quote `"like this"`
- `exepath` : path of executable

- **NOTE**: `sh` or `bash` [built-in commands](https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html) like `echo` will not yield `Process` events
- **NOTE**: currently, the agent does not show `pipe` or `file redirection` details.  Therefore, if a user types `echo $SOMEVAR | grep blah | sort | uniq  >> /some/file`, there will be 3 process events seen: `grep blah`, `sort`, and `uniq`.  There will be a `File` event if the path is within the monitored paths, and the process writing the file will be the `shell`, not the `uniq` process.

Example:

```
_E_,Process,cmdline~=sftp -P #{remote_port} #{username}@#{remote_host}:#{remote_path}
```

### `_E_,File` row
The 3rd column (`SubType`) should only contain a single value.  Therefore, if a file is expected to be `CREATE` or `WRITE`, just use `WRITE`.  All `File` rows can match on the `path` field (currently that might be only one).

The Subtypes mostly align with the `telemetry.proto` types:

- `READ`
- `WRITE`
- `CHMOD` : TODO: fields to match (`mode`?)
- `UNLINK` or `DELETE`
- `RENAME`


Example:

```
_E_,File,WRITE,path=#{local_file}
```

### `_E_,Netflow` row

The netflow format is `protocol:local_ip:localport->remote_host:remote_port`.  Use `*` to indicate wildcard.  Since some netflow records may have a dns hostname, the `remote_host` may be a `hostname` or `ipaddr` .
```
_E_,Netflow,TCP:*->#{remote_host}:#{remote_port}
```
