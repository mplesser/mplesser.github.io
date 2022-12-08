---
title: "Overview"
date: 2022-12-08
---
                       
# Introduction
AzCam is a software framework for the acquisition and analysis of image data from scientific imaging systems as well as the control of instrumentation. It is intended to be customized for specific hardware, observational, and data reduction requirements.

AzCam is based on the concept of *tools* which are the interfaces to both hardware and software code.  Examples of tools are *instrument* which controls instrument hardware, *telescope* which interfaces to a telescope, *linearity* which acquires and analyzes images to determine sensor linearity, and *exposure* which controls a scientific observation by interfacing with a variety of other tools. As an example, the *exposure* tool may move a telescope and multiple filter wheels, control a camera shutter, operate the camera by taking an exposure, display the resultant image, and begin data reduction of that image.

AzCam is not appropriate for consumer-level cameras and is not intended to have a common API across all systems. It's primary design principle is to allow interfacing to a wide variety of custom instrumentation which is required to acquire and analyze scientific image data.

## AzCam Links
  - Main links
    - [AzCam documentation site](https://mplesser.github.io/azcam/)
    - [GitHub repos](https://github.com/mplesser)
  - Code details and links
    - [Tools](tools/tools.md)
    - [Classes](classes.md)
    - [Commands](commands.md)
  - Advanced concepts
    - [Advanced](advanced.md)


# Operation

There are three main operational modes of AzCam:

  1. A server, usually called *azcamserver*, which communicates directly or indirectly with system hardware.

  2. A console, usually called *azcamconsole*, which is typically implemented as an IPython command line interface that communicates with *azcamserver* over a socket connection.  It is used to acquire and analyze image data through the command line and python code.

  3. Applications, which are usually client programs that communicate with *azcamserver* over sockets or the web API.

While there are multiple pythonic ways to access the various tools in code, they are always accessible from the *database* object `db`, which can always be accessed as `azcam.db`. For example, when defined, the `controller` tool can be accessed as `db.tools["controller"]` and the `qe` tool can be accessed as `db.tools["qe"]`.  In most environments these tool are mapped directly into the command namespace, so in practice commands are executed directly as `object.method`, e.g. `exposure.expose(2.5, "dark", "dark image")`.

Tools defined on the server side may or may not be available as remote commands from a client. 

In *azcamconsole*, the available tools usually communication from the console to an *azcamserver* application over a socket interface.  These client-side tools may only expose a limited set of methods as compared to the server-side tools. So while the command `exposure.reset` may be available from a client the command `exposure.set_video_gain` may not be.  These less commonly used commands are still accessible, but only with lower level code such as `server.command("controller.set_video_gain 2")` where `server` is a client's server communication tool.

As an specific example, the code below can be used to set the current system wavelength and take an exposure.  For this example, it is assumed here that the *azcam-itl* environment package has been installed (see [Environments](#azcam-environments) below.).

```python
# server-side (azcamserver)
import azcam
import azcam_itl.server
instrument = azcam.db.tools["instrument"]
exposure = azcam.db.tools["exposure"]
instrument.set_wavelength(450)
wavelength = instrument.get_wavelength()
print(f"Current wavelength is {wavelength}")
exposure.expose(2., 'flat', "a 450 nm flat field image")

or

# client-side (azcamconsole)
import azcam
import azam_itl.console
instrument = azcam.db.tools["instrument"]
exposure = azcam.db.tools["exposure"]
instrument.set_wavelength(450)
wavelength = instrument.get_wavelength()
print(f"Current wavelength is {wavelength}")
exposure.expose(2., 'flat', "a 450 nm flat field image")
```

Both *azcamserver* and *azcamconsole* may also be called in a manner similar to:

```python
ipython -m azcam_itl.server -i -- -system LVM
ipython -m azcam_itl.console - -- -system LVM
```

Other examples:
```python
ipython --profile azcamserver  # to start IPython
from azcam_itl import server
from azcam.cli import *
instrument.set_wavelength(450)
exposure.expose(2.5,"flat","a test image")
```

```python
azcam azcam_itl.console -system DESI
```

Example configuration code may be found in the various environment packages with names like `server.py` and `console.py`.

When working in a command line environment, it is often convenient to import commonly used commands into the CLI namespace. This provides direct access to objects and tools such as *db*, *exposure*, *controller*, and various pre-defined shortcuts. To do this, after configuring the environment, execute the commandfrom `from azcam.cli import *`.

And then the code above could be executed as:
```python
from azcam.cli import *
instrument.set_wavelength(450)
exposure.expose(2., 'flat', "a 450 nm flat field image")
```

# Tools

Some of the many supported tools are listed in this section.

## Astronomical Research Cameras Controllers - `azcam.tools.arc`

These tools support Astronomical Research Cameras, Inc. gen1, gen2, and gen3 controllers. See https://www.astro-cam.com/.

### Example Code

The code below is for example only.

#### Controller Setup
```python
import azcam.server
from azcam.tools.arc.controller_arc import ControllerArc
controller = ControllerArc()
controller.timing_board = "arc22"
controller.clock_boards = ["arc32"]
controller.video_boards = ["arc45", "arc45"]
controller.utility_board = None
controller.set_boards()
controller.pci_file = os.path.join(azcam.db.systemfolder, "dspcode", "dsppci3", "pci3.lod")
controller.video_gain = 2
controller.video_speed = 1
```

#### Exposure Setup
```python
import azcam.server
from azcam.tools.arc.exposure_arc import ExposureArc
exposure = ExposureArc()
exposure.filetype = azcam.db.filetypes["MEF"]
exposure.image.filetype = azcam.db.filetypes["MEF"]
exposure.set_remote_imageserver("localhost", 6543)
exposure.image.remote_imageserver_filename = "/data/image.fits"
exposure.image.server_type = "azcam"
exposure.set_remote_imageserver()
```

#### Camera Servers
*Camera servers* are separate executable programs which manage direct interaction with 
controller hardware on some systems. Communication with a camera server takes place over a 
socket via communication protocols defined between *azcam* and a specific camera server program. These 
camera servers are necessary when specialized drivers for the camera hardware are required.  They are 
usually written in C/C++. 

#### DSP Code
The DSP code which runs in the ARC controllers is assembled and linked with
Motorola software tools. These tools are typically installed in the folder `/azcam/motoroladsptools/` on a
Windows machine as required by the batch files which assemble and link the code.

While the AzCam application code for the ARC timing board is typically downloaded during
camera initialization, the boot code must be compatible for this to work properly. Therefore
AzCam-compatible DSP boot code may need to be burned into the timing board EEPROMs before use, depending on configuration. 

The gen3 PCI fiber optic interface boards and the gen3 utility boards use the original ARC code and do not need to be changed. The gen1 and gen2 situations are more complex.

For ARC system, the *xxx.lod* files are downlowded to the boards.

## STA Archon Controller - `azcam.tools.archon`

These tools support STA Archon controllers. See http://www.sta-inc.net/archon/.

### Example Code

The code below is for example only.

#### Controller
```python
import azcam.server
from azcam.tools.archon.controller_archon import ControllerArchon
controller = ControllerArchon()
controller.camserver.port = 4242
controller.camserver.host = "10.0.2.10"
controller.header.set_keyword("DEWAR", "ITL1", "Dewar name")
controller.timing_file = os.path.join(
    azcam.db.systemfolder, "archon_code", "ITL1_STA3800C_Master.acf"
)
```

#### Exposure
```python
import azcam.server
from azcam.tools.archon.exposure_archon import ExposureArchon
exposure = ExposureArchon()
filetype = "MEF"
exposure.fileconverter.set_detector_config(detector_sta3800)
exposure.filetype = azcam.db.filetypes[filetype]
exposure.image.filetype = azcam.db.filetypes[filetype]
exposure.display_image = 1
exposure.image.remote_imageserver_flag = 0
exposure.add_extensions = 1
```

## ASCOM - `azcam.tools.ascom`

These tools support ASCOM cameras. See https://ascom-standards.org/. This code has been used for QHY and ZWO cameras.

### Example Code

The code below is for example only.

#### Controller

```python
import azcam.server
from azcam.tools.ascom.controller_ascom import ControllerASCOM
controller = ControllerASCOM()
```

#### Exposure

```python
import azcam.server
from azcam.tools.ascom.exposure_ascom import ExposureASCOM
exposure = ExposureASCOM()
filetype = "FITS"
exposure.filetype = exposure.filetypes[filetype]
exposure.image.filetype = exposure.filetypes[filetype]
exposure.display_image = 1
exposure.image.remote_imageserver_flag = 0
exposure.set_filename("/data/zwo/asi1294/image.fits")
exposure.display_image = 1
```

## CryoCon Testerature Controller - `azcam.tools.cryocon`

This tools supports Cryogenic Control Systems Inc. (cryo-con) temperature controllers. See http://www.cryocon.com/.

### Example Code

The code below is for example only.

#### Temperature Controller

```python
import azcam.server
from azcam.tools.cryocon.tempcon_cryocon24 import TempConCryoCon24
tempcon = TempConCryoCon24()
tempcon.description = "cryoconqb"
tempcon.host = "10.0.0.44"
tempcon.control_temperature = -100.0
tempcon.init_commands = [
"input A:units C",
"input B:units C",
"input C:units C",
"input A:isenix 2",
"input B:isenix 2",
"input C:isenix 2",
"loop 1:type pid",
"loop 1:range mid",
"loop 1:maxpwr 100",
]
```

## SAO Ds9 Image Display Tool - `azcam.tools.ds9`

This tool supports SAO's ds9 display tool running under Windows. See https://sites.google.com/cfa.harvard.edu/saoimageds9.

See https://github.com/mplesser/azcam-ds9-winsupport for support code which may be helpful when displaying images on Windows computers

The `Display` class defines Azcam's image display interface to SAO's ds9 image display. 
It is usually instantiated as the *display* object for both server and clients.

Depending on system configuration, the *display* object may be available 
directly from the command line, e.g. `display.display("test.fits")`.

Usage Example:

```python
from azcam.tools.ds9.ds9display import Ds9Display
display = Ds9Display()
display.display("test.fits")
rois = display.get_rois(0, "detector")
print(rois)
```

## Focus - `azcam.tools.focus`

This tools controls focus observations used to determine optimal instrument or telescope focus position.

This code is usually executed in the console window although a server-side version is available on some systems.

`focus` is an instance of the *Focus* class.

### Code Examples

`focus.command(parameters...)`

```python
focus.set_pars(1, 30, 10)  
focus.run()
```

### Focus Parameters

Parameters may be changed from the command line as:
`focus.number_exposures=7`
or
`focus.set_pars(1.0, 5, 25, 15)`.

<dl>
  <dt><strong>focus.number_exposures = 7</strong></dt>
  <dd>Number of exposures in focus sequence</dd>

  <dt><strong>focus.focus_step = 30</strong></dt>
  <dd>Number of focus steps between each exposure in a frame</dd>

  <dt><strong>focus.detector_shift = 10</strong></dt>
  <dd>Number of rows to shift detector for each focus step</dd>

  <dt><strong>focus.focus_position</strong></dt>
  <dd>Current focus position</dd>

  <dt><strong>focus.exposure_time = 1.0</strong></dt>
  <dd>Exposure time (seconds)</dd>

  <dt><strong>focus.focus_component = "instrument"</strong></dt>
  <dd>Focus component for motion - "instrument" or "telescope"</dd>

  <dt><strong>focus.focus_type = "absolute"</strong></dt>
  <dd>Focus type, "absolute" or "step"</dd>

  <dt><strong>focus.set_pars_called = 1</strong></dt>
  <dd>Flag to not prompt for focus position</dd>

  <dt><strong>focus.move_delay = 3</strong></dt>
  <dd>Delay in seconds between focus moves</dd>
</dl>

## Remote Image Server - `azcam.tools.sendimage`

This tool supports sending an image to a remote host running an image server which receives the image.

### Usage

```python
from azcam.tools.sendimage import SendImage
sendimage = SendImage()
remote_imageserver_host = "10.0.0.1"
remote_imageserver_port = 6543
sendimage.set_remote_imageserver(remote_imageserver_host, remote_imageserver_port, "azcam")
```

## Magellan Controller - `azcam.tools.mag`

These tools support the OCIW Magellan CCD controllers (ITL version). See http://instrumentation.obs.carnegiescience.edu/ccd/gcam.html.

### Example Code

The code below is for example only.

#### Controller

```python
import azcam.server
from azcam.tools.mag.controller_mag import ControllerMag
controller = ControllerMag()
controller.camserver.set_server("some_machine", 2402)
controller.timing_file = os.path.join(azcam.db.datafolder, "dspcode/gcam_ccd57.s")
```
#### Exposure

```python
import azcam.server
from azcam.tools.mag.exposure_mag import ExposureMag
exposure = ExposureMag()
filetype = "BIN"
exposure.filetype = exposure.filetypes[filetype]
exposure.image.filetype = exposure.filetypes[filetype]
exposure.display_image = 1
exposure.image.remote_imageserver_flag = 0
exposure.set_filename("/azcam/soguider/image.bin")
exposure.test_image = 0
exposure.root = "image"
exposure.display_image = 0
exposure.image.make_lockfile = 1
```

### Camera Servers

*Camera servers* are separate executable programs which manage direct interaction with controller hardware on some systems. Communication with a camera server takes place over a socket via communication protocols defined between *azcam* and a specific camera server program. These camera servers are necessary when specialized drivers for the camera hardware are required.  They are usually written in C/C++. 

### DSP Code

The DSP code which runs in Magellan controllers is assembled and linked with
Motorola software tools. These tools should be installed in the folder `/azcam/motoroladsptools/` on Windows machines, as required by the batch files which assemble and link the code.

For Magellan systems, there is only one DSP file which is downloaded during initialization. 

Note that *xxx.s* files are loaded for the Magellan systems.


## FastAPI - `azcam.tools.fastapi`

This tool implements a fastapi-based web server.  See https://fastapi.tiangolo.com.

### Usage Example

```python
from azcam.tools.fastapi.fastapi_server import WebServer
webserver = WebServer()
webserver.index = f"index_mysystem.html"
webserver.start()
```

## AzCam Web Tools - `azcam.tools.webtools`

These tools implements various browser-based tools which connect to an azcam web server.

### Usage

Open a web browser to http://localhost:2403/XXX where XXX is a toolname, with the appropriate replacements for localhost and the web server port number.

### Browser Tools
 - status - display current exposure status
 - exptool - a simple exposure control tool

## AzCam Environments

Some packages act as *environments* to define code and data files used for specific hardware systems. Examples include:

  * [azcam-90prime](https://github.com/mplesser/azcam-bok) for the UArizona Bok telescope 90prime instrument
  * [azcam-mont4k](https://github.com/mplesser/azcam-mont4k) for the UArizona Mont4k instrument
  * [azcam-vattspec](https://github.com/mplesser/azcam-vattspec) for the VATT VattSpec camera

## AzCam Applications
AzCam *applications* are stand-alone programs which utilize AzCam functionality. The most important application is *azcamserver* which defines the tools for a hardware system. Most but not all applications are clients which connect to an *azcamserver* application. These clients cab be written in any languages.  Some are experimental or still in development. Examples include:

  * [azcam-expstatus](https://github.com/mplesser/azcam-expstatus): a small GUI which displays exposure progress 
  * [azcam-monitor](https://github.com/mplesser/azcam-monitr): an app to monitor and control AzCam processes on the local network
  * [azcam-tool](https://github.com/mplesser/azcam-tool): an exposure control GUI written in National Instruments LabVIEW
  * [azcam-imageserver](https://github.com/mplesser/azcam-imageserver) adds support for remote image servers
  * [azcam-observe](https://github.com/mplesser/azcam-observe) add observing scripts which support a Qt-based GUI and command line interface
    * [azcam-observe code docs can be found here](https://mplesser.github.io/docs/azcam_observe/index.html)

## Help
AzCam is commonly used with IPython.  Help is then available by typing `?xxx`, `xxx?`, `xxx??` or `help(xxx)` where `xxx` is an AzCam class, command, or object instance.

Useful links include:
* IPython <https://ipython.org>
* Python programming language <https://www.python.org>

## Command Structure
The AzCam command structure provides a fairly uniform interface which can be used from the local command line (CLI), a remote socket connection, or the web interface.  An example for taking a 2.5 second *flat field* exposure is:

Local CLI or script example:

```python
exposure.expose(2.5, 'flat', 'an image title')
```

A remote socket connection example:

```python
exposure.expose 2.5 flat "an image title"
```

Web (http) connection example:

```html
http://hostname:2403/api/exposure/expose?exposure_time=2.5&image_type=flat&image_title=an+image+title
```
Web support requires an extension package such as *azcam-fastapi* to be installed with *azcamserver*.  Example URL's are:

```html
http://hostname:2403/status
```
```html
http://hostname:2403/exptool
```

The value *2403* here is the web server port configured by a specific environment.

## Shortcuts
When using IPython, the auto parenthesis mode allows typing commands without 
requiring the normal python syntax of ``command(par1, par2, ...)``. The equivalent 
shortcut/alias syntax is ``command par1 par2``. With IPython in this mode all commands may 
use this syntax.

There are also some simple but useful command line commands which can be optionally installed within
console or server applications as typing shortcuts. Shortcuts are intended for command line use only
and can be found in the `db.shortcuts` dictionary (see `azcam.shortcuts`). Examples include:

  * *sav* - save the current parameters to the parameter file
  * *pp* - toggle the command line printout of client commands and responses command.
  * *gf* - try and go to current image folder.
  * *sf* - try and set the image folder to the current directory.
  * *bf* - browse for a file or folder using a Tcl/Tk GUI.

Autogenerated code documentation for the `azcam` python package can be found [here](http://mplesser.github.io/docs/azcam).

## Scripts
Scripts are python code modules which usually contain one function. They may be loaded automatically during enviroment configuration and can be found in the `db.scripts` dictionary. Scripts defined on the server side are not available as remote commands.

```python
import azcam_scripts
azcam_scripts.load()
get_pressures(2.0, "get_pressures.log", 1)
```

## Configuration Folders
There are two important folders which are defined by most environments:

  * *systemfolder* - the main folder where configuration code is located. It is often the root of the environment's python package.
  * *datafolder* - the root folder where data and parameters are saved. Write access is required. It is often similar to `/data/sytemfolder`.

