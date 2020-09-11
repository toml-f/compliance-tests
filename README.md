# Compliance tests for the TOML-Fortran library

There is not much to see here, maybe have a look at:

- the [TOML standard](https://toml.io)
- the [toml-test](https://github.com/BurntSushi/toml-test) validator test suite
- the [meson build system](https://mesonbuild.com)


## Use this in your project

To use this in your meson project create a TOML decoder that emits JSON following the guidelines in the validator test suite (assumed to be named `toml2json_exe` in the following snippet).

Than create a `toml-tests.wrap` in your subprojects directory to fetch the tests as alternative to the shipped TOML v0.4 tests in the validator test suite.

```
[wrap-git]
directory = toml-tests
url = https://github.com/toml-f/compliance-tests
revision = head
```

Now you only have to pull the validator and setup the different validation tests.
This example will setup the tests that come with the validator as benchmark and the tests in this repository as normal unit test to make them both available for testing.

```meson
# We will run a TOML decoder validator, in case we find a go installation
go_exe = find_program('go', required: false)
if go_exe.found()
  # Obviously, we don't want to write into the users go-workspace, therefore,
  # we will create our own local workspace by overwriting the GOPATH variable
  validator_env = environment({
    'GOPATH': meson.current_build_dir(),
  })

  compliance_prj = subproject('toml-tests')
  compliance_testdir = compliance_prj.get_variable('compliance_tests')

  # Explicitly specify the validator test directories
  validator_testdir = meson.current_build_dir()/'src'/'github.com'/'BurntSushi'/'toml-test'/'tests'

  # First, we need to fetch the validator tests from upstream
  get_toml_test = run_command(
    go_exe,
    'get',
    'github.com/BurntSushi/toml-test',
    env: validator_env,
  )

  # To make sure the command is actually used before we declare the test we
  # make it dependent on the success of the go get command
  if get_toml_test.returncode() == 0
    validator_exe = find_program(
      'toml-test',
      dirs: meson.current_build_dir()/'bin',
    )
    # Finally, create the decoder test
    test(
      'decoder',
      validator_exe,
      args: ['-testdir', compliance_testdir, toml2json_exe],
      env: validator_env,
    )
    # This is the original BurntSushi/toml-test suite
    benchmark(
      'decoder',
      validator_exe,
      args: ['-testdir', validator_testdir, toml2json_exe],
      env: validator_env,
    )
  endif
endif
```
