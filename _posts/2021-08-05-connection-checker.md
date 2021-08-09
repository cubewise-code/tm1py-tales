---
layout: post title:  "TM1py Connection Checker"
date:   2021-08-05 9:00:00 categories: personal-story
---

# TM1 Connection Checking Tool

I decided to code this small tool to provide a simple way to test various connection parameters and assist with
troubleshooting in case of errors when connecting from TM1py to TM1.

Motivation is to avoid writing the same code (yet easy) when experiencing connection issues and have a simple
diagnostics of exceptional states  (e.g., wrong credentials, wrong port, TM1 not up and running, etc.) already at hand.

![Screenshot - Export via Architect](https://github.com/cubewise-code/tm1py-tales/blob/master/_images/2021-08-05_config.png?raw=true)  
*Sample Configuration*

![Screenshot - Export via Architect](https://github.com/cubewise-code/tm1py-tales/blob/master/_images/2021-08-05_cmd.png?raw=true)  
*Sample Execution*

## About the Tool

The tool was coded as a command line utility with a simple command line interface (CLI), for simplicity I decided to
distribute it as an executable file.

The tool is using a simple JSON configuration file that is storing a dictionary (catalogue) of various connection
configurations. Each configuration has a unique name and is then assigned a dictionary of TM1py connection parameters.

When run, it is expected the user supplies a filename of a JSON file, and a name of connection configuration that should
be tested. The default configuration file name is `test_conn.json`, however alternatively the user might supply a
different JSON configuration file to the CLI. The tool will check existence of the JSON config file, then it will locate
connection configuration using a configuration name supplied to the CLI. It will then use the connection parameters
retrieved from the JSON configuration file and attempt to connect. If an exception occurs during a connection attempt, a
hint about potential cause of the exception is displayed to the console.

## Implementation

Following paragraphs will describe in detail techniques used during the tool implementation. We will focus on basic
building blocks of the tool, which are:

- Command Line Interface - basic user interface to control the tool
- Exceptional states handling - handle various types of connection issues
- Compiling executable file - use Python code to produce executable file for multiple platforms

### Command Line Interface

Command line arguments parsing is a basic feature that is available out of the box in many programming languages.
However, it is usually limited to just basic features as for example "retrieve a second command line argument" or
"return number of supplied command line arguments". Things might get complicated if the command line interface of
intended tool has multiple arguments, some of which will require only certain set of allowed values. Even more
complications would arise if the arguments would be structured as different commands and each *command* requiring
different set of allowed *options*. For example in Windows Powershell you may use a cmdlet called `Get-Date` to display
current system date and time. Let's explore its default behavior:

```powershell
Get-Date
```

```text
Thursday, August 05, 2021 12:19:32
```

However, you may supply various *commands* followed by their *option* values to the CLI to instruct the cmdlet to modify
the default behavior:

```powershell
Get-Date -DisplayHint Date
```

```text
Thursday, August 05, 2021
```

In above example the *command* used with cmdlet `Get-Date` was `-DisplayHint` and its *option* value was `Date`.

Another example how a simple *command* might modify behavior of the same cmdlet:

```powershell
Get-Date -Format "dddd MM/dd/yyyy HH:mm"
```

```text
Thursday 08/05/2021 12:31
```

To demonstrate how just a simple tool might become complicated from CLI point of view, let's combine above examples to
one:

```powershell
Get-Date -DisplayHint Date -Format "dd.MM.yyyy (dddd)"
```

```text
05.08.2021 (Thursday)
```

We may also change order of the *commands* that we have supplied in previous example - but as expected, the result will
be the same:

```powershell
Get-Date -Format "dd.MM.yyyy (dddd)" -DisplayHint Date
```

```text
05.08.2021 (Thursday)
```

As above examples demonstrate, parsing command line with simple functions that return number of command line arguments
and their values would be a complicated task. This is a reason why we should lookup a good command line parser library
to help us with the task.

#### CLICK Command Line Parsing Library

I chose this library for its simplicity of use. Consider following fragment of the code (let's call it `test1.py`):

```python=
import click


# definition of a simple command with 2 options that have default values and help text
@click.command(help='Run a connection test by using a connection configuration file.')
@click.option('--config_file', default='test_conn.json', help='Connection configuration JSON file.', type=click.File('r'))
@click.option('--config_profile', default='test_mode_1', help='Case sensitive name of connection configuration.')
def login(config_file, config_profile):
    # option values usage
    print('config_file={}'.format(config_file))
    print('config_profile={}'.format(config_profile))


if __name__ == '__main__':
    # click initialization - our tool will have only one default command called login, users will not be required to type it to the CLI
    login()

```

Now you may actually see how simple is to define a basic *command* in our tool. On line 5 in our code sample we have
defined a new command `login` by using `@click.command()` decorator. As this is the only command in our tool, we don't
need to expose it to the command line interface (it will be a default command, user will not be required to type it to
the CLI). This is accomplished by a direct call of login() from the main code block (lines 14-16) and usage
of `@click.command()` decorator on the `login` method.

The `@click.command()` decorator will also link the `login` method automatically with CLI, so we don't have to code any
explicit conditions that would be otherwise required to call the method when the command was used on the CLI. All of
this is backed by the CLICK library behind the scenes.

Now try running the code with `--help` argument (this is another neat feature of CLICK as it handles help for us
automatically in proper contextual way):

```
python test1.py --help
```

```
Usage: test1.py [OPTIONS]

  Run a connection test by using a connection configuration file.

Options:
  --config_file FILENAME  Connection configuration JSON file.
  --config_profile TEXT   Case sensitive name of connection configuration.
  --help                  Show this message and exit.
```

You may notice we have introduced two different *options* that belong to the new *command* on lines 11 and 12. These
are `--config_file` to specify name of the JSON configuration file and `--config_profile` to specify name of
configuration profile to test connecting.

CLICK will handle everything for us - from default option value for both options to interchangeability of order in which
they can be typed to the command line interface (see above examples with `Get-Date`). As a bonus, CLICK is able to
handle options that specify file name or path. In first case you don't even need to explicitly open the file, CLICK will
do it for us automatically (and again - CLICK will even test for the file existence, so we don't need to explicitly test
it in our code).

Usage examples of CLI as defined in our sample code above:

- both options supplied - order doesn't matter

```
python test1.py --config_file "connections.json" --config_profile Connection_test
```

or

```
python test1.py --config_profile Connection_test --config_file "connections.json"
```

will both produce:

```
config_file=<_io.TextIOWrapper name='connections.json' mode='r' encoding='UTF-8'>
config_profile=Connection_test
```

- default parameter values

```
python test1.py 
```

will output:

```
config_file=<_io.TextIOWrapper name='test_conn.json' mode='r' encoding='UTF-8'>
config_profile=test_mode_1
```

- non-existant file supplied as an option value:

```
python test1.py --config_file "make_error.json"
```

will output:

```
Usage: test1.py [OPTIONS]
Try 'test1.py --help' for help.

Error: Invalid value for '--config_file': Could not open file: make_error.json: No such file or directory
```

If we wanted more commands to be available in our application, we would need to create a *group* of commands first and
then add the commands to the group by using the `@<group>.command()` decorator - like this (let's call this code
sample `test2.py`):

```python
import click


@click.group()
def our_commands():
    pass


# definition of a simple command with 2 options that have default values
@our_commands.command(help='Run a connection test by using a connection configuration file.')
@click.option('--config_file', default='test_conn.json', help='Connection configuration JSON file.',
              type=click.File('r'))
@click.option('--config_profile', default='test_mode_1', help='Case sensitive name of connection configuration.')
def login(config_file, config_profile):
    # option values usage
    print('config_file={}'.format(config_file))
    print('config_profile={}'.format(config_profile))


@our_commands.command(help='Just a demo command.')
def test():
    print('Test!')


if __name__ == '__main__':
    # click initialization
    our_commands()

```

Try running the code with `--help`:

```
python test2.py --help
```

```
Usage: test2.py [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  login  Run a connection test by using a connection configuration file.
  test   Just a demo command.
```

Now run it with `login --help` and compare with previous example with default command (user had to explicitly
type `login` command to the CLI):

```
python test2.py login --help
```

```
Usage: test2.py login [OPTIONS]

  Run a connection test by using a connection configuration file.

Options:
  --config_file FILENAME  Connection configuration JSON file.
  --config_profile TEXT   Case sensitive name of connection configuration.
  --help                  Show this message and exit.
```

If you want to give a try to CLICK, it is as easy as `pip install -U click`.

#### Other Options to CLICK

Perhaps if you have solved similar task before you came across `argparse` library. I chose CLICK over `argparse` for its
simplicity and clean and concise code. Also when using `argparse` you need to code your own handler that would call
different methods based on commands that were parsed, so the code is longer and thus prone to more errors.

### Handling Exceptional States

Here I must be strict, since the only correct way how to handle any potential errors in your code is to use structured
approach that is build into Python (no discussion allowed). Errors that occur in run-time should always be handled
with `try - except` block. The reason is simple - information about error - wherever it has occurred will be packed into
an object instance descending from `BaseException` class and will be automatically handed over to the point where your
exception handler (`except` block) will catch it. It may catch exactly the instance of exception you want to handle, or
it may be a generic handled that just catches all exception types. The latter is recommended to catch rest of exceptions
not handled previously as a "fallback" handler. Using only one handler is not a good practice. Example of exception
handling is obvious from the code sample below:

```python
...


# Define our own exception class to catch situation when a configuration profile is missing in the config file
class MissingConfigException(Exception):
    pass


...


def login(config_file, config_profile):
    try:
        connection = json.load(config_file)
        if config_profile in connection:
            conn = connection[config_profile]
        else:
            raise MissingConfigException
        print("Using connection config file [{f}], configuration profile [{c}].".format(
            f=config_file.name,
            c=config_profile
        ))
        print("Testing server connection...")
        print()
        with TM1Service(**conn) as tm1:
            print("[OK] Test has passed successfully, connected to [{s}] as [{u}], TM1 product version [{v}].".format(
                s=tm1.server.get_server_name(),
                u=tm1.whoami.friendly_name,
                v=tm1.server.get_product_version()
            ))
    except MissingConfigException:
        print("[FAIL] Required configuration profile [{c}] in [{f}] is missing.".format(
            c=config_profile,
            f=config_file.name
        ))
    except KeyError as e:
        print(
            "[FAIL] Required configuration profile [{c}] in [{f}] is invalid. Check configuration completeness [{e}]".format(
                c=config_profile,
                f=config_file.name,
                e=e
            ))
    except requests.exceptions.InvalidURL as e:
        print("[FAIL] Malformed URL for connection [{e}] (check URL in configuration profile [{c}])".format(
            e=e,
            c=config_profile
        ))
    except requests.exceptions.ConnectionError as e:
        if e.args[0].args[0] == "Connection aborted.":
            print("[FAIL] Connection to server could not be open (check SSL usage)")
        else:
            print("[FAIL] Connection to server could not be open [{}]".format(e))
    except TM1pyRestException as e:
        if e.status_code == 401:
            print("[FAIL] User is not authenticated (check credentials)")
        elif e.status_code == 503:
            print("[FAIL] TM1 server is unavailable (check TM1 service is up and responding)")
        else:
            print("[FAIL] TM1Py exception occurred: {}".format(
                e
            ))
    except Exception as e:
        print("[FAIL] Other exception occurred: {}".format(
            e
        ))
    ...
```

As obvious from the code sample, in all cases except the last one we are handling particular exception types and
displaying appropriate message depending on their class. The last `except` handler is a "fallback" handler that will
display exception that was not caught in previous `except` blocks. The example above also shows how to define our own
exception that is specific to our tool. As other exception states this one is handled in a standard way by
using `except` block followed by a name of the exception class.

### Compiling Executable File

I decided to compile the tool to an executable file as from my experience it is an easiest form for code distribution.
Python code might be distributed as well, but you lose control over versions of libraries that are used in the solution
as new versions are released independently, so it may easily happen that at some point of time the code would become
incompatible with newly released libraries.

Currently, there are several options how to compile an executable file from your Python source code. I chose the
simplest and by far fastest compilation method by using `PyInstaller`.
`PyInstaller` is not a compiler per se, it is rather a packager that will package your code and libraries with a Python
execution environment to a single package - executable file. Nice feature of `PyInstaller` is it allows compiling
executables for all main platforms (Windows, Mac and Linux). Target platform is not selectable, `PyInstaller` will
automatically create an executable for the platform being used on, so it is not possible to compile Mac executable from
a Windows environment.

Usage of `PyInstaller` is simple. First it needs to be installed into the execution environment by
running `pip install pyinstaller`. Once it is installed, creating an executable file from your source code is easy and
can be done one step as below:

```
pyinstaller --onefile --clean --name conn_test.exe "conn_test.py"
```

The output is one executable file that bundles everything your Python code needs to be run on any compatible
environment. With last versions of `PyInstaller` you receive a compact executable file, previous versions generated
slightly bigger files. It is necessary to say as `PyInstaller` is bundling stuff together, running the executable file
has slight overhead in rather seconds that is required for the bundled Python interpreter start.

#### Other Options to PyInstaller

There is an interesting project available called Nuitka that promises to deliver an executable file that doesn't need
any Python interpreter bundled into it. It is a cross-compiler that is able to compile your Python code to a C code and
compile the executable file from it. From my experience it takes a long time to compile a simple Python code, but I have
to say I have experimented with this tool about a year ago - so yes, it is again worth trying, Nuitka is in active
development! That was one of reasons why I decided to go with `PyInstaller`, I needed a stable tool to compile
executable files.

## Links

- CLICK - https://click.palletsprojects.com/en/8.0.x/
- Python exceptions - https://docs.python.org/3/library/exceptions.html
- PyInstaller - https://www.pyinstaller.org/
- Nuitka - https://nuitka.net/

## Final Code

### Python Part

```python
import requests
from TM1py.Exceptions import TM1pyRestException
from TM1py import TM1Service
import click
import json

__author__ = "Petr Buncik"
__copyright__ = "(c) 2021 Apliqo AG"
__credits__ = ["Radu Cantor", "Maurycy Mioduski"]
__version__ = "1.0 Production"


class MissingConfigException(Exception):
    pass


@click.command(help='Run a connection test by using a connection configuration file.')
@click.option('--config_file', default='test_conn.json', help='Connection configuration JSON file.',
              type=click.File('r'))
@click.option('--config_profile', default='test_mode_1', help='Case sensitive name of connection configuration.')
def login(config_file, config_profile):
    print("UX Toolkit Connection Checker")
    print(__copyright__)
    print("Version: {}".format(__version__))
    print()
    try:
        connection = json.load(config_file)
        if config_profile in connection:
            conn = connection[config_profile]
        else:
            raise MissingConfigException
        print("Using connection config file [{f}], configuration profile [{c}].".format(
            f=config_file.name,
            c=config_profile
        ))
        print("Testing server connection...")
        print()
        with TM1Service(**conn) as tm1:
            print("[OK] Test has passed successfully, connected to [{s}] as [{u}], TM1 product version [{v}].".format(
                s=tm1.server.get_server_name(),
                u=tm1.whoami.friendly_name,
                v=tm1.server.get_product_version()
            ))
    except MissingConfigException:
        print("[FAIL] Required configuration profile [{c}] in [{f}] is missing.".format(
            c=config_profile,
            f=config_file.name
        ))
    except KeyError as e:
        print(
            "[FAIL] Required configuration profile [{c}] in [{f}] is invalid. Check configuration completeness [{e}]".format(
                c=config_profile,
                f=config_file.name,
                e=e
            ))
    except requests.exceptions.InvalidURL as e:
        print("[FAIL] Malformed URL for connection [{e}] (check URL in configuration profile [{c}])".format(
            e=e,
            c=config_profile
        ))
    except requests.exceptions.ConnectionError as e:
        if e.args[0].args[0] == "Connection aborted.":
            print("[FAIL] Connection to server could not be open (check SSL usage)")
        else:
            print("[FAIL] Connection to server could not be open [{}]".format(e))
    except TM1pyRestException as e:
        if e.status_code == 401:
            print("[FAIL] User is not authenticated (check credentials)")
        elif e.status_code == 503:
            print("[FAIL] TM1 server is unavailable (check TM1 service is up and responding)")
        else:
            print("[FAIL] TM1Py exception occurred: {}".format(
                e
            ))
    except Exception as e:
        print("[FAIL] Other exception occurred: {}".format(
            e
        ))


if __name__ == '__main__':
    login()

```

### JSON sample configuration file

```JSON
{
  "test_mode_1": {
    "address": "localhost",
    "port": 12345,
    "ssl": true,
    "user": "",
    "password": ""
  },
  "test_mode_2a": {
    "address": "localhost",
    "port": 12345,
    "ssl": true,
    "integrated_login": true
  },
  "test_mode_2b": {
    "address": "localhost",
    "port": 12345,
    "ssl": true,
    "integrated_login": true,
    "integrated_login_host": "",
    "integrated_login_service": "",
    "integrated_login_delegate": true,
    "integrated_login_domain": ""
  },
  "test_mode_5_CAM_SSO": {
    "address": "localhost",
    "port": 12345,
    "ssl": true,
    "namespace": "",
    "gateway": ""
  },
  "test_mode_5_CAM_only": {
    "address": "localhost",
    "port": 12345,
    "ssl": true,
    "user": "",
    "password": "",
    "namespace": ""
  }
}
```

Written
by [![Linkedin](https://i.stack.imgur.com/gVE0j.png) Petr Buncik](https://www.linkedin.com/in/petr-buncik-61b0822/)
