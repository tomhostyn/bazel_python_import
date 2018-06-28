# bazel_python_import
reproduces behaviour where non declared code is executed

The code has the following structure:
```
.
├── baz.py
├── BUILD
├── evil.py
└── WORKSPACE
```

The BUILD file defines one test target, based on a single file
```
py_test(
    name = "baz.test",
    srcs = [ "baz.py", ],
    main = "baz.py",
)
```
baz.py attempts to import a file not declared in the BUILD file
```
try:
    import evil
except ImportError:
    print ("OK - hermetic!")
```

evil.py looks like this:
```
print ("NOK - not hermetic")
assert (False)
```
Expected result is that the import fails.  However, it succeeds.

```
$ bazel test //:baz.test 
INFO: Build options have changed, discarding analysis cache.
INFO: Analysed target //:baz.test (16 packages loaded).
INFO: Found 1 test target...
FAIL: //:baz.test (see /mnt/disks/disk1/tom_hostyn/cache_instance3/bazel/_bazel_tom_hostyn/4e754f332c3f2483920411a899f8d8e7/execroot/bazel_bug_reproduce/bazel-out/k8-fastbuild/testlogs/baz.test/test.log)
Target //:baz.test up-to-date:
  bazel-bin/baz.test
INFO: Elapsed time: 2.748s, Critical Path: 0.36s
INFO: 2 processes: 1 local, 1 processwrapper-sandbox.
INFO: Build completed, 1 test FAILED, 5 total actions
//:baz.test                                                              FAILED in 0.3s
  /mnt/disks/disk1/tom_hostyn/cache_instance3/bazel/_bazel_tom_hostyn/4e754f332c3f2483920411a899f8d8e7/execroot/bazel_bug_reproduce/bazel-out/k8-fastbuild/testlogs/baz.test/test.log
```
where the log is:

```
exec ${PAGER:-/usr/bin/less} "$0" || exit 1
Executing tests from //:baz.test
-----------------------------------------------------------------------------
NOK - not hermetic
Traceback (most recent call last):
  File "/mnt/disks/disk1/tom_hostyn/cache_instance3/bazel/_bazel_tom_hostyn/4e754f332c3f2483920411a899f8d8e7/sandbox/processwrapper-sandbox/2/execroot/bazel_bug_reproduce/bazel-out/k8-fastbuild/bin/baz.test.runfiles/bazel_bug_reproduce/baz.py", line 4, in <module>
    import evil
  File "/mnt/disks/disk1/tom_hostyn/repos/bazel_python_import/evil.py", line 2, in <module>
    assert (False)
AssertionError
```

it seems evil.py has made it to the execution root:
```
ls $(bazel info execution_root)
bazel-out  baz.py  _bin  BUILD  evil.py  external  README.md  WORKSPACE
```

Furthermore, when we add files to the workspace, they also end up in the execution root

```
$ ls
baz.py  BUILD  evil.py  LICENSE  README.md  WORKSPACE

$ touch XXXXXXX

$ ls
baz.py  BUILD  evil.py  LICENSE  README.md  WORKSPACE  XXXXXXX

$ bazel test //:baz.test 
INFO: Build options have changed, discarding analysis cache.
INFO: Analysed target //:baz.test (14 packages loaded).
INFO: Found 1 test target...
FAIL: //:baz.test (see /mnt/disks/disk1/tom_hostyn/cache_instance3/bazel/_bazel_t
om_hostyn/4e754f332c3f2483920411a899f8d8e7/execroot/bazel_bug_reproduce/bazel-out
/k8-fastbuild/testlogs/baz.test/test.log)
Target //:baz.test up-to-date:
  bazel-bin/baz.test
INFO: Elapsed time: 1.379s, Critical Path: 0.15s
INFO: 2 processes: 1 local, 1 processwrapper-sandbox.
INFO: Build completed, 1 test FAILED, 5 total actions
//:baz.test                                                              FAILED i
n 0.1s
  /mnt/disks/disk1/tom_hostyn/cache_instance3/bazel/_bazel_tom_hostyn/4e754f332c3
f2483920411a899f8d8e7/execroot/bazel_bug_reproduce/bazel-out/k8-fastbuild/testlog
s/baz.test/test.log
INFO: Build completed, 1 test FAILED, 5 total actions

$ ls $(bazel info execution_root
)
bazel-out  _bin   evil.py   LICENSE    _tmp   WORKSPACE
baz.py     BUILD  external  README.md  tools  XXXXXXX
```

This seems to break the hermeticy assumptions ?


