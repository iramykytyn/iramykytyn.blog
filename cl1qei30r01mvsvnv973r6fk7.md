## How To Integrate Valgrind into GitHub Actions?

Have you ever got into the situation that after you fixed memory leaks with Valgrind you see ```Definitely lost``` again after a while? Then this CI setup may be useful for you.

You can do it in 3 steps:
- Install Valgrind in your CI environment.
- Run tests with Valgrind and output results to XML.
- Add script to parse XML and report errors.
- You may also want to control when to run it. Running with Valgrind may be time-consuming, so you may want to run it only on Pull Requests to the ```master``` branch. Or you may want to run it only for some specific tests.

These steps are common for every possible CI. Let’s see it on GitHub Actions example.

> 💡 If you did not work with GitHub Actions before, see [Quickstart for GitHub Actions](https://docs.github.com/en/actions/quickstart).


## Add Valgrind install step to ```yml``` file

```
- name: Install Valgrind
  run: |
    sudo apt install valgrind
    echo "Valgrind installed"
```

You can run into the issue that Valgrind version is not compatible with your CI system. Then you will need to build and install another version of Valgrind: [here is an example of how to do it](https://www.claudiokuenzler.com/blog/797/install-upgrade-valgrind-3-13.0-on-ubuntu-14.04-alternatives).


## Update test’s run script to run with Valgrind

```bash
- name: Install Valgrind
  run: |
    sudo apt install valgrind
    echo "Valgrind installed"
```

```--suppressions=./path/to/suppresion/my_app.supp``` - this option is needed only if you have suppression file.
You can generate suppressions for the errors for the libraries, but it’s not a good practice.
It's hard to know if a memory error in the library is caused by a problem in your code or not.
But if it is not possible to fix a leak, and you are sure it’s not your leak, then 1) suppress errors from the library (short-term solution); 2) upgrade the library or connect the community who supports it and ask for a fix.
Example of the first option. Ignore Leak errors in **libcrypto** only, you could put a [suppression](https://valgrind.org/docs/manual/mc-manual.html#mc-manual.suppfiles) like:

```
{
   ignore_libcrypto_conditional_jump_errors
   Memcheck:Leak
   ...
   obj:*/libcrypto.so.*
}
```

into a file and pass it to ```valgrind``` with ```--suppressions=./path/to/suppresion/my_app.supp```.
You can also let Valgrind generate suppression file for you. Use ```-gen-suppressions=yes``` from [Core Command-line Options](https://valgrind.org/docs/manual/manual-core.html#manual-core.options).

```--xml=yes --xml-file=unit_tests_valgrind.xml``` - enable Valgrind’s to report in XML format and save it in ```unit_tests_valgrind.xml```.


## Parse XML output

You can write your own using [ValgrindCI](https://pypi.org/project/ValgrindCI/) tool or write a custom XML parser.

### ValgrindCI tool
```
- name: Valgrind Memory check
        run: |
          pip install ValgrindCI
					echo "Summary report of errors"
					valgrind-ci /path/to/output_file.xml --summary
	        valgrind-ci /path/to/output_file.xml --abort-on-errors # abort on errors
```

### Custom XML parser in Bash
The output XML format is specified in the file ```docs/internals/xml-output-protocol4.txt```  in the source tree for Valgrind 3.5.0 or later. It has ```errorcounts``` or ```error``` tags to report errors:

```
ERROR definition -- common structure
------------------------------------
ERROR defines an error, and is the most complex nonterminal. For all
of the tools, the structure is common, and always conforms to the
following:
<error>
	<unique>HEX64</unique>
	<tid>INT</tid>
	<threadname>NAME</threadname> if set
	<kind>KIND</kind>
	...
	optionally: SUPPRESSION
</error>
* Each error contains a unique, arbitrary 64-bit hex number. This is
used to refer to the error in ERRORCOUNTS nonterminals (see above).
```

It’s enough to check if ```/ERROR``` tag exists to know if at least one error was reported. But it’s more secure to parse all tags and return the exact number of errors and short summary.

```
# Check if the xml has any "</errorcounts>" attributes. If it has, memory errors exists.
sed 's/"/ /g' < $xml_file | grep '</error>' &> /dev/null
if [ $? == 0 ]; then
    printf "%s" "$boldred"
    echo "[  FAILED  ] Some memory checks failed for $xml_file."
    error=-1
		exit $error;
else
    echo "[  SUCCESS  ]"
fi
```

```yml``` step:

```
- name: Valgrind Memory check
  run: |
    echo "Checking Valgrind xmls ..."
    ./check-memory.sh
```

It is good to use a custom script if you need to parse a few XML’s and report separately on each.


## Example of how to control Valgrind run with environment variable

```RUN_MEMORY_CHECK``` - environment variable to control when to run Valgrind memory check.
In the example below, it will run memory check steps only for ```main``` branch or if ```RUN_MEMORY_CHECK``` is added to GitHub Actions Secrets and is set to ```true```


```
name: "Build and run tests with memory check"

on:
  pull_request:
  push:
    branches:
      - main      
			- dev

env:
  RUN_MEMORY_CHECK: ${{ (github.base_ref == 'main') || ((secrets.RUN_MEMORY_CHECK == 'true') }}
jobs:
  build:
    name: "Build"
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: x86_64

    steps:
      - uses: actions/checkout@v2
      - name: Set up environment
        run: |
          sudo apt update
          sudo apt install -y libev-dev
      - name: Install Valgrind
        if: ${{ env.RUN_MEMORY_CHECK == 'true' }}
        run: |
          sudo apt install valgrind
          echo "Valgrind installed"

      - name: Build Tests
        run: |
          # add build script execution here

      - name: Run Tests with Valgrind
        run: |
					valgrind --leak-check=full --suppressions=./path/to/suppresion/my_app.supp --xml=yes --xml-file=unit_tests_valgrind.xml ./unit_tests_binary

      - name: Valgrind Memory check
        if: ${{ env.RUN_MEMORY_CHECK == 'true' }}
        run: |
          echo "Checking Valgrind xml ..."
	        ./check-memory.sh
```
