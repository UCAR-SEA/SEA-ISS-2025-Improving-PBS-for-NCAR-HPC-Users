# Methods

## Designing the `pbsparse` package

The PBS Pro scheduler writes historical *event* records in "accounting logs", which are stored on the PBS server host - typically an isolated system not accessible to users. These logs are structured with one event per line, and each event contains data separated by commas or spaces, depending on whether the information is record metadata or job metadata. An example record follows.

>04/16/2025 09:23:33;Q;4413086.casper-pbs;user=vanderwb group=csgteam account="SCSG0001" project=_pbs_project_default jobname=vncs-default queue=casper ctime=1744817013 qtime=1744817013 etime=0 Resource_List.gpu_type=gp100 Resource_List.mem=10gb Resource_List.ncpus=1 Resource_List.ngpus=1 Resource_List.nodect=1 Resource_List.place=scatter Resource_List.select=1:ncpus=1:ngpus=1:os=opensuse15:ompthreads=1 Resource_List.walltime=04:00:00

In this *queue-job* record, we can see the time, the record type (`Q` for queue-job), the job ID, and then a collection of job metadata which will contain different fields depending on the record type. The following subset of record types are most relevant to users:

* `Q` - a new job has been queued
* `S` - the job has begun execution
* `E` - the job has finished execution and is being removed from the queue
* `R` - the job has failed, and is being re-queued for another attempt

In the original `qhist`, records were processed using purpose-written code, but many use cases are possible (including compiling reports on system utilization). Therefore, we wanted to separate out processing of these records into a new package, which we are calling `pbsparse`.

In the redesigned code, we have moved from a list of dictionaries - with each `dict` containing a record - to defining a `PbsRecord` class with each event being an object of this type. If you have a record in string form such as the above queue-job event, processing into a usable object is simple:

```python
import pbsparse

q_event = pbsparse.PbsRecord(record, process = True)
```

By setting `process = True`, we tell the init routine to convert relevant data into useful forms:  `ncpus` into `int`, start and end times into `datetimes` and so on.

### Lightweight processing with a *generator*

The `pbsparse` package also includes a function, `get_pbs_records`, which allows the user to read in all records from an accounting log file. This operation could become quite expensive in the old `qhist` code as a file would be read in and `dict`-type records would be stored in a `list`. Since the raw accounting logs can grow to O(100)MB in size, and queries can span multiple log files (one log per day) the resulting Python data structures became prohibitively large.

In the redesign, we have altered this code so that instead of returning a list of dictionaries, it returns a *[generator](https://wiki.python.org/moin/Generators)* of `PbsRecord` objects. Many Python coders will be familiar with generators, but fewer coders will have written their own. It is very simple; a stripped down snippet of the `get_pbs_records` code follows.

```python
def get_pbs_records(data_file, process = False, type_filter = None, ...):
    try:
        if reverse:
            cm = ReverseOpen(data_file)
        else:
            cm = open(data_file, "r")
    except FileNotFoundError:
        print("Warning: missing records in time period ({})".format(data_file), file = sys.stderr)

    with cm as records:
        for record in records:
            if not type_filter or record[20] in type_filter:
                event = PbsRecord(record, process, time_divisor = time_divisor)
								
				# match is determined by omitted record-filtering code
                if match:
    		        yield event
```

The generator can then be used as follows.

```python
# Read all "job-end" records in logs from 2025-04-01
jobs = get_pbs_records("/logs/20250401", True, "E")

# Loop through generator and print job ID
for job in jobs:
    print(job.short_id)
```

This code will only read in a single record at a time from the file. The `pbsparse` package also provides a mechanism, via the `ReverseOpen` class, to read in the file in reverse and thus work with event records in the opposite ordering with only minimal added processing and memory expense.

We can see the results of this redesign in comparing memory usage (using a custom tool designed at NCAR called [peak-memusage](https://github.com/NCAR/peak_memusage)) by the old and new `qhist` versions for the same query:

```bash
# Running the old qhist
$ time peak_memusage qhist -p 20241112
casper-login2 used memory in task 0: 293MiB
real  0m6.794s
```

This day featured an exceptionally large number of jobs, but as we can see the memory usage is non-trivial (293MiB). Meanwhile, the new `qhist` uses far less:

```bash
$ time peak_memusage bin/qhist -p 20241112
casper-login2 used memory in task 0: 11MiB
real  0m7.921s
```

We achieved a **96.25%** decrease in memory usage simply by converting a list to a generator!

## Improved formatting and filtering

The old `qhist` implementation allowed for output formatting and data filtering, but there were limitations:

* A custom format could be specified using a Python format string, but all data were first cast to `str`, so only string formatting could be used
* Data could only be filtered by predefined command-line arguments. Additionally, most filters were only testing for equality (e.g., `user == value`)

Both of these limitations are removed in the new `qhist`.

### Type-native formatting

The Python string formatter can accept object attributes as named fields, and attributes are kept in their original type. As a result, the following format can be specified:

```bash
$ qhist -f "{short_id:12.12} {user:10.10} {numcpus:>6d} {end:%d-%H%M} {memory:>8.2f}"
Job ID       User        NCPUs     End  Mem(GB)
------------ ---------- ------ ------- --------
4413086      vanderwb        1 16-1019     1.19
4414946      vanderwb        1 16-1452     0.24
```

There are multiple output modes supported by `qhist` - tabular (as shown above), long-form, csv, and json (new in the redesign) - and custom formats are supported by all modes. For all but tabular output, format is simply specified by listing the desired attributes.

### Free-form filtering

While existing filtering options are still supported, we wanted to provide users with the means to filter on almost any job attribute. A simple implementation would take a user-specified filtering function and evaluate it, but `eval()` is dangerous in that it allows for malicious code injection. Instead, we deconstruct the given function and user the `operator` class to perform the requested filtering operation.

Users can specify their filter via a semicolon-delimited string:

```bash
$ qhist -F "numcpus>1;user==vanderwb" -p 20250401-20250416
Job ID       User       Queue    Nodes  NCPUs NGPUs     End  Mem(GB)   CPU(%)  Elap(h)
------------ ---------- -------- ----- ------ ----- ------- -------- -------- --------
4386652      vanderwb   htc          2      8     0 14-1018     0.48     0.25     0.20
4389304      vanderwb   htc          2      8     0 14-2122     1.42     1.12     6.02
```

The specified filters will be broken down into operator expressions and added to a list of data filters:

```python
data_filters.append([(operator.gt, "numcpus", 1),(operator.eq, "user", "vanderwb")])
```

All filters are then sent to `pbsparse.get_pbs_records()`, so records that don't match the conditions are filtered out and never returned by the generator.

## Parallel installation methods: `pip` and `Makefile`

The old version of `qhist` featured a rudimentary installation approach: simply clone the repository, create a config file (undocumented), and add to your `PATH`. This approach was serviceable for our purposes but did not allow for any complexity, such as adding package requirements. To improve this situation we have implemented two complementary installation methods: using `pip` to install from PyPI and a `Makefile` install directly from a clone of the repository.

### Converting `qhist` to a package distribution

Turning `qhist` into an actual Python package confers many benefits, even if we don't envision it being our primary deployment method on NSF NCAR systems. We can better track versions and dependencies, and we can easily share the tool with other sites running PBS Pro.

This process, as documented in the helpful [Python Packaging tutorial](https://packaging.python.org/en/latest/tutorials/packaging-projects), is very easy. We decided to use `setuptools`  as the build backend to create our wheels. Once we created a packaging environment with the necessary tools, all we need to do was create a pyproject.toml file with the package specifications. A simplified version is shown here:

```ini
[build-system]
requires = ["setuptools >= 77.0","setuptools-scm>=8"]
build-backend = "setuptools.build_meta"

[project]
name = "pbs-qhist"
description = "A utility to query historical PBS Pro job data"
requires-python = ">=3.6"

dependencies = ["pbsparse>=0.2.2"]

[project.scripts]
qhist = "qhist.qhist:main"
```

### Using a `Makefile` for packaging and installation

Since the build commands to create and upload our distributions to PyPI are sequentially dependent, we found a `Makefile` provided the right functionality to direct this process. We defined the following make *targets*:

```Makefile
build:
    python3 -m build

# These commands can only be run successfully by package maintainers
manual-upload: 
    python3 -m twine upload dist/*

test-upload:
    python3 -m twine upload --repository testpypi dist/*
```

We also included an `install` target, which allows for directly installing the package from a clone of the repository. We track `pbsparse` as a Git submodule to handle that dependency when not using `pip`:

```Makefile
make install: lib/pbsparse/Makefile
    mkdir -p $(PREFIX)/bin $(PREFIX)/lib/qhist $(PREFIX)/share
    sed 's|/src|/lib/qhist|' bin/qhist > $(PREFIX)/bin/qhist
    cp -r src/qhist $(PREFIX)/lib/qhist
    cp -r lib/pbsparse/src/pbsparse $(PREFIX)/lib/qhist
    cp -r share $(PREFIX)/share
    chmod +x $(PREFIX)/bin/qhist

lib/pbsparse/Makefile:
    git submodule init
    git submodule update
```

## Adding user documentation

The original version of `qhist` suffered from an all-to-common affliction: a total lack of user documentation. The only breadcrumbs offered to the user came in the form of the `-h/--help` argparse options. Beyond that, our  users were largely on their own (though we did provide *some* documentation of the command in our online HPC documentation).

To remedy this omission, we decided to add two basic forms of documentation that come standard with most programs - the `README.md` and a `man` page. Adding the README was straightforward enough, though we intend to add some automation ensuring that the file is updated for every new version.

We could also manually create the man page, but this is error prone and laborious. Instead, we use a helpful package called [argparse-manpage](https://github.com/praiskup/argparse-manpage), which takes the information contained with the `ArgumentParser` object and outputs a man page (accepting some customization arguments along the way). Again, we leverage our `Makefile` to define a man page creation target:

```Makefile
# Requires packages "pbsparse" and "argparse-manpage" in your Python environment
man:
    argparse-manpage --pyfile src/qhist/qhist.py --author "Written by Brian Vanderwende."   \
        --project-name qhist --function get_parser --version $(VERSION)                     \
        --description "a utility for querying historical PBS records"                       \
        --manual-title "PBS Professional Community Utilities"                               \
        --output share/man/man1/qhist.1
```

## Regression and *live* testing

The lack of any testing mechanism in the previous `qhist` version meant that problems were almost always discovered during actual usage. If a new feature was implemented, parts of the code base rarely used could break in ways not evaluated by inconsistent manual testing.

Even worse, the PBS Pro accounting logs occasionally include fields with unexpected type, non-standard user inputs, or simply truncated records due to a server hiccup. Unless staff or users reported the error during usage for the day, such problems could go unnoticed.

We have added both regression and live testing to ameliorate both of these issues.

### Regression tests with *pytest*

Many regression testing packages/frameworks exist for Python packages. One of the most popular frameworks is [pytest](https://docs.pytest.org/) - used by [numpy](https://github.com/numpy/numpy/tree/main/numpy/tests), for one prominent example - which we use to implement regression testing for `qhist`.

Most tests with *pytest* are written simply using functions containing an assertion. For example, in this test we evaluate whether the time bounds calculation function returns the correct result given a number of days back to search the logs:

```python
def test_get_number_days():
    class NewDatetime(datetime.datetime):
        @classmethod
        def today(cls):
            return cls(2025, 3, 1, 0, 0)

    datetime.datetime = NewDatetime
    period = qhist.get_time_bounds("20250218", "%Y%m%d", days = 4)
    output = " ".join([d.strftime("%Y%m%d") for d in period])
    assert output == "20250225 20250301"
```

Running all tests is simple - simply call `pytest` from the `tests` directory in the repository (the package must be installed in your environment, of course).

### *live* testing using self-hosted runners

In processing the accounting logs, there are many exceptions that must be handled. More of these exceptions are discovered over time as `qhist` encounters exotic/malformed event records. This discovery is a manual process, however, and it can easily break down.

Fortunately, we can rely on GitHub Actions *[self-hosted](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)* runners to perform a nightly test on actual accounting records from our two production systems. The defined testing workflows will run on our systems, but will be triggered by conditions set in the repository. Our staff have designed [tooling](https://github.com/NCAR/cron-github-runner) to keep self-hosted runners alive using cron jobs, ensuring our testing proceeds as intended. The workflow can also be run manually via the web UI using a `workflow_dispatch` trigger.

Our workflow job for testing on Derecho is shown below.

```yaml
  derecho-test:
    name: ⚗️  Test qhist parsing on live Derecho data
    runs-on: derecho-runner
    if: github.event_name == 'schedule' || contains(fromJSON('["both","derecho"]'), inputs.system)

    steps:
    - uses: actions/checkout@v4
    - name: Install qhist with Makefile
      run: make install PREFIX=install
    - name: Test qhist and check status
      run: |
        bin/qhist > /dev/null
      working-directory: install
```

This job also validates out our `Makefile` installation method. In the event that a log cannot be parsed properly, we will have record in the workflow log. Here is an example of a workflow that failed the live testing:

```
Run bin/qhist > /dev/null
Traceback (most recent call last):
	File "/glade/work/vanderwb/repos/qhist/actions-runners/derecho/_work/qhist/qhist/install/bin/qhist", line 10, in <module>
		qhist.main()
...
	File "/glade/work/vanderwb/repos/qhist/actions-runners/derecho/_work/qhist/qhist/install/lib/qhist/qhist/qhist.py", line 80, in get_field
	obj = obj[i]
				~~~^^^
KeyError: 'avgcpu'
Error: Process completed with exit code 1.
```

Such reporting allows us to easily determine which field was problematic. Future automation could create a GitHub issue as soon as a test fails, though we would need to control for periods in which the systems, and thus the runners, are down.