# qgc-tlog-proc
Guide and python scripts to unpack and process telemetry log data collected from QGroundControl.

## Getting Started
First, make sure you have installed either [Python 3.8.10](https://www.python.org/downloads/release/python-3810/) or 
[Python 3.9](https://www.python.org/downloads/release/python-396/) for your system.

Next we are going to setup a basic workspace and python virtual environment for you to work in
. Bring up a terminal in your chosen OS and enter the following:

```
mkdir tlog_processing
cd tlog_processing
python -m pip install virtualenv
virtualenv venv
```

NB: for some platforms the `python -m pip install virtualenv` command might start with `python3`, `python3.8` or `python3.9` instead of `python`.
### Windows Specific ###
Activate the virtual environment as follows:

`venv\Scripts\activate.bat`
### Linux / Mac
Activate the virtual environment as follows:

`source venv/bin/activate`

Now you should see `(venv)` as a prefix to your command line, this indicates that the virtual environment is active.
To deactivate the virtual environment, enter `deactivate`.

### Installing tools
With your virtual environment active, enter the following command to install
some third party libraries that will enable you to read and process .tlog files.

`python -m pip install pymavlink scipy`

### Locate the telemetry log files

The .tlog files are typically located in:

`$USER/Documents/QGroundControl/Telemetry`

Where $USER is the absolute path to your Documents folder, (e.g. `C://Users/DTUAqua`). They are stored based on when
QGroundControl was first opened, and are safely closed (i.e. can be read by other programs) when QGroundControl is closed.

### Using the mavlogdump tool

First get the help documentation for the tool by:

`mavlogdump.py -h`

This lists all the possible options that be passed to `mavlogdump.py`.

#### MATLAB .mat Conversion
To convert the entire log (all data) into a MATLAB friendly .mat format, try the followign:

`mavlogdump.py --format mat --mat_file test.mat your_tlog_file_here.tlog`

Each option is passed with a `-` or `--` preceding the keyword (e.g. `--format`).
The argument for the option is then specified immediately after following a space.
All arguments and options are delimited via spaces when executing a program from the commandline.
The last argument is the path to the .tlog file that you wish to unpack. You should be able to open 
the resulting .mat file with MATLAB.

#### Comma Separated Value .csv Conversion
If you would prefer the data output as a CSV, which can be read by excel, MATLAB, python, or most other programs
then some additional considerations are required.

First, inspect your .tlog file for the specific data that it contains:

`mavlogdump.py --show-types your_tlog_file_here.tlog`

This usually takes a while, but will end up showing something like this:

```
AHRS
MISSION_ACK
GLOBAL_POSITION_INT
VFR_HUD
REQUEST_DATA_STREAM
AHRS3
MISSION_COUNT
DISTANCE_SENSOR

... (SKIPPED FOR BREVITY) ...

SCALED_PRESSURE2
NAV_CONTROLLER_OUTPUT
GPS_RAW_INT
POWER_STATUS
```
These are the data packets available within the .tlog file, and represent a communication
standard known as MAVLINK. Each data packet is structured to contain certain
data, the entire description of which can be looked up on the [MAVLINK Common Message Description](https://mavlink.io/en/messages/common.html).
For example, the `GLOBAL_POSITION_INT` data packet has a description [here](https://mavlink.io/en/messages/common.html#GLOBAL_POSITION_INT).
As described, this message type contains the vehicle's estimated position, heading and speed. Very
useful for determining where the ROV has been!

We can extract this message into a CSV formatted timeseries with the follwoing command:

`mavlogdump.py --format csv --types GLOBAL_POSITION_INT your_tlog_file_here.tlog > test.csv`

The last `> test.csv` takes the output of the program and saves it to a file called `test.csv`.
If you don't include that, then the nicely formatted csv rows are just outputted to the screen and
no file is created.

You can also include multiple message types, just seperate with a comma (no spaces):

`mavlogdump.py --format csv --types GLOBAL_POSITION_INT,DISTANCE_SENSOR your_tlog_file_here.tlog > test.csv`

This unpacks `GLOBAL_POSITION_INT` and `DISTANCE_SENSOR` ([see here](https://mavlink.io/en/messages/common.html#DISTANCE_SENSOR))
messages into the same file. Note that because the messages are logged into the .tlog file
asynchronously (i.e. they are logged when they arrive and are not synced with other messages) you will notice that these
combination csv files have a lot of empty fields on each row. This is because each row is a formatted
representation of a specific message type. You can fill in these empty fields through interpolation,
but watch out for empty fields at the start and at the end of the time series, these will be extrapolated and won't be accurate.

### Notes on special fields

Make sure to check the units specified with the [MAVLINK Common Message Description](https://mavlink.io/en/messages/common.html).
They might be not suitable for calculations or display and might need to be converted.

Each CSV file has a header with the fields of the message specified with .'s as delimiters.

The first column in the CSV file is the timestamp, which is the time at which the message was
received and stored by the logging software, and not the time when it was captured by the ROV sensor.
This is usually fine to use as the lag between capture and logging is only a few milliseconds. Sometimes there can
be lag spikes, so use under caution.
