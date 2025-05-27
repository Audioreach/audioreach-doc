.. _raspberry_pi4:

Raspberry Pi 4
##################################################

1. `Architecture Overview <#Architecture Overview>`__

2. `Create a Yocto Image <#create-a-yocto-image>`__

3. `Setting up your Raspberry Pi <#setting-up-your-raspberry-pi>`__

4. `Running an AudioReach Usecase <#running-an-audioreach-usecase>`__

5. `Troubleshooting <#troubleshooting>`__

This page provides AudioReach Architecture overview on Raspberry Pi platform and walks through steps on how to create a Yocto image that integrates AudioReach,
load that image on your Raspberry Pi4, and then run an AudioReach usecase.

Architecture Overview
=====================
      .. figure:: images/raspberry_pi_reference.png
         :figclass: fig-center
         :scale: 80 %

The above architecture diagram illustrates the playback use-case on a Raspberry
Pi using AudioReach. In this setup, the agmplay test app is utilized to play an
audio clip, and the sound output is rendered through an output device such as
speakers or headphones.

Here, when a graph open request is received by AudioReach Graph Services (ARGS)
from the client, ARGS retrieves the audio graph and calibration data using the
use case handle and calibration handle from Audio Calibration Data Base (ACDB).
It then provides the graph definition and corresponding calibration data to
the AudioReach Engine (ARE) via the Generic Packet Router (GPR) protocol over
a physical or soft data link.

Upon receiving the data, ARE forms an audio graph with processing modules
according to the graph definition. It processes the audio data piped from the
source endpoint to the ALSA Sink endpoint, which is then rendered through a
BCM2835 sound card. Although ARE allows developers to design their use case
graphs and support distributed processing across heterogenous cores, given
that Raspberry Pi lacks DSP, ARE runs on the APPS processor in user space.

Additionally, during the playback use-case, the graph topology can be
visualized in real-time using a PC-based GUI tool called AudioReach
Creator (ARC, also known as QACT).

Create a Yocto image
====================

The first step is to integrate our AudioReach components
into a Yocto build that you can use with Raspberry Pi. To do
this, we will take the Yocto build and use meta-audioreach layer
from the AudioReach Github.

Step 1: Set up your Yocto build
-------------------------------

Please follow the below steps to setup your Yocto build:

   * First check to make sure you have the system requirements for a yocto build. You can check that at the yocto setup instructions here: https://docs.yoctoproject.org/2.0/yocto-project-qs/yocto-project-qs.html

   * Make a directory "yocto" and in that directory, make a directory called "sources".

   * In the "sources" directory, clone the following repositories:

   .. code-block:: bash

      git clone git://git.yoctoproject.org/poky -b scarthgap
      git clone git://git.yoctoproject.org/meta-raspberrypi -b scarthgap
      git clone https://git.openembedded.org/meta-openembedded -b scarthgap

   * Go back to the "yocto" directory and run this command to setup your build environment: **source ./sources/poky/oe-init-build-env**

   * Now in the "yocto/build/conf/local.conf" file, replace the line **MACHINE ?= "<machine>"** with the line **MACHINE ?= "raspberrypi4"**

   * Lastly, we need to edit the build/conf/bblayers.conf file and add the meta layers. Edit the file as shown:

   .. code-block:: bash

      # POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.con
      # changes incompatibly
      POKY_BBLAYERS_CONF_VERSION = "2"

      BBPATH = "${TOPDIR}"
      BBFILES ?= ""

      BBLAYERS ?= " \
         <path_to_build>/yocto/sources/poky/meta \
         <path_to_build>/yocto/sources/poky/meta-poky \
         <path_to_build>/yocto/sources/poky/meta-yocto-bsp \
         <path_to_build>/yocto/sources/meta-raspberrypi \
         <path_to_build>/yocto/sources/meta-openembedded/meta-oe \
         <path_to_build>/yocto/sources/meta-openembedded/meta-multimedia \
         <path_to_build>/yocto/sources/meta-openembedded/meta-networking \
         <path_to_build>/yocto/sources/meta-openembedded/meta-python \
	"

Note: Please make sure that your system has the requirements needed for yocto scarthgap builds. If not, you can download the pre-built "buildtools" for yocto:

   .. code-block:: bash

	  cd <path_to_build>/yocto/sources/poky
	  scripts/install-buildtools

Then run the following commands to setup your build environment to use buildtools:

   .. code-block:: bash

	  cd <path_to_build>/yocto/
	  source ./sources/poky/oe-init-build-env
	  source ./sources/poky/buildtools/environment-setup-x86_64-pokysdk-linux


Step 2: Get AudioReach Meta Layer
---------------------------------

Use the below commands to get the AudioReach meta image:

   .. code-block:: bash

      cd <path_to_build>/yocto/sources
      git clone https://github.com/Audioreach/meta-audioreach.git


Step 3: Add AudioReach to system image
--------------------------------------

The last step is to add AudioReach to the system image to make sure it's compiled.

   * Navigate to **yocto/build/conf/local.conf** and append the below
     line to the file:

		.. code-block:: bash

			IMAGE_INSTALL:append = "audioreach-graphservices tinyalsa audioreach-graphmgr audioreach-engine audioreach-conf"

   * Then add these additional lines to local.conf to enable support for ARE (AudioReach Engine) on APPS processor, as Raspberry Pi does not have DSP:

		.. code-block:: bash

			PACKAGECONFIG:pn-audioreach-graphmgr = "are_on_apps use_default_acdb_path"
			PACKAGECONFIG:pn-audioreach-graphservices = "are_on_apps"

   * Navigate to **yocto/build/conf/bblayers.conf** and under the **BBLAYERS ?= " \\** section, add the AudioReach meta path:

		.. code-block:: bash

			<path_to_build>/yocto/sources/meta-audioreach \

Step 4: Compile the image
-------------------------

Now we can compile the build. Navigate to **yocto/build** directory
and run the command: **bitbake core-image-sato**

* Note: If you get a "umask" error after compiling the build, run the command **umask 022** and try compiling again.
* If you see a "restricted license" error, navigate to the local.conf file and append the below line:

	.. code-block:: bash

		LICENSE_FLAGS_ACCEPTED = "synaptics-killswitch"

If the compilation was successful, you should be able to find the newly generated Yocto image in your workspace.

Navigate to the folder **yocto/build/tmp/deploy/images/raspberrypi4** and unzip the folder **core-image-sato-raspberrypi4.rootfs.wic.bz2**. This will give you the .wic
file that you will use to flash your Raspberry Pi.

Alternatively, you can run the command
**bzip2 -d -f tmp/deploy/images/raspberrypi4/core-image-sato-raspberrypi4.rootfs.wic.bz2**
in your build directory after compiling to unzip the image.

Step 5: Flash the Yocto image
-----------------------------

Now we're going to flash the Yocto image using Raspberry Pi Imager. You can install
this from **raspberrypi.com/software**, or by running **sudo apt install rpi-imager** on your terminal.

* Open the application, and select RaspberryPi4 as the device type.
* Under the Choose OS options, select the "Use custom" option. Make sure you are searching for all file types (by default it doesn't search for .wic files). Then search for your .wic file and select it.
* Under Storage, select the SD card that you want to flash the image onto, and click Flash.

Now you can use your SD card to bootup your Raspberry Pi.


Setting up your Raspberry Pi
============================

First you'll want to setup the hardware on your Raspberry Pi, if not done already.
For this you can follow the steps on the official Raspberry Pi
documentation page here: https://www.raspberrypi.com/documentation/computers/getting-started.html

Follow the steps until the section "Install an operating system".

Configure bootup settings
-------------------------

Next, we will need to complete a few steps to enable the audio and update
the logging settings. You can update the configuration files mentioned
below using "vi" on the terminal; however, it is much easier to just
navigate to these files on the file system in your Raspberry Pi and
update them there. These steps only need to be done once.

To be able to hear the audio output, we need to enable the sound card:

   * Navigate to file **/boot/config.txt**
   * Look for the line **#dtparam=audio=off**
   * Change this line to **dtparam=audio=on**

      * Make sure to uncomment this line while you are updating it.

By default, the system logs printed for running a Raspberry Pi usecase
will be short. We will want to update the settings to make the logs longer:

   * Navigate to **/etc/syslog-startup.conf**
   * Uncomment lines **Rotate size (ROTATESIZE)** and **Rotate Generations (ROTATEGENS)**
   * Set **ROTATESIZE** to 1000000.

      * The rotate size refers to the file size cap before creating a new file to write logs to. We are setting it to a large number to capture as many logs as possible.
   * Set **ROTATEGENS** to 20.

      * This indicates the maximum number of log files that we can generate.
   * Save the file.

Next, you'll want to push a ".wav" audio file to some location in the Raspberry Pi (such as the "/etc" folder).

With this the configuration should be finished. Shut down the Raspberry Pi through
the homescreen or by running the command **shutdown -r -time "now"** through the
terminal so the changes can take effect.

Enable Real-time Calibration Mode
---------------------------------

ARC (AudioReach Creator) is a tool that allows you to see the current graph
configuration while running a usecase, as well as create and modify your own graphs.
These steps are optional, as you don't technically need ARC to run the usecase.

On your Raspberry Pi:

   * Connect the Raspberry Pi to internet using Ethernet or over Wifi.
      * Ethernet
         * Plug an Ethernet cable into the Raspberry Pi’s Ethernet port.
      * Wifi
         * On the top right of the screen click the icon beside the time, and select "Preferences".
         * Find the "Wireless Network" option on the left to choose the network.
   * Open a terminal and run the command **ifconfig** to get your current IP address.
   * Run **ats_gateway <IP address> 5558**
   * Open another terminal and run the command **agm_server**

On your local computer:
   * Install ARC (also known as QACT) on Windows host machine using :ref:`steps_to_install_arc`. You will need at least QACT 8.1
   * Open ARC, and click on "Connection configuration" option.
   * Add the Raspberry Pi as a device by adding entry
      **<Raspberry PI IP address>:5558** under the TCP/IP section
   * Refresh the "Available Devices" list. The IP address of your Raspberry Pi should appear on the list.

      * Note: If it does not come up, make sure the **ats_gateway** and **agm_server** commands are still running.

   * Choose the entry and click connect.

Now when running a usecase, you should be able to see the current usecase
graph on ARC.

Running an AudioReach Usecase
=============================

Now all the setup is finished, and you should finally be able to use the
Raspberry Pi to play audio.

   * Connect your headphones/other audio device to the audio port on your Raspberry Pi.
   * Open a terminal and run the command **agm_server** (if not done already).
   * Open another terminal window and run the below command to start the playback usecase:

       **agmplay /[path_to_audio_file]/<clip_name>.wav -D 100 -d 100 -i PCM_RT_PROXY-RX-2**

If all goes well, you should be able to hear the output through your headphones.
The system logs for the usecase will be saved in /var/log/messages.

Troubleshooting
===============

If you get an error while trying to run the usecase, below are some things you can
try:

Check the sound card
--------------------

On your Raspberry Pi, open the file /proc/asound/cards. You should see a couple
sound card entries in this list. If it says "no sound cards available", you likely
forgot to enable the sound card (see section `Configure bootup settings <#configure-bootup-settings>`__).

Check the sound card ID
-----------------------

If your Raspberry Pi is connected to the monitor, the HDMI-based soundcard might get enumerated in proc/asound/cards, causing the
card ID of the Headphones to change. For this you will need to have ARC installed on a secondary computer (see *Enable
ARC connection*).

   #. Copy the ACDB files from your Raspberry Pi to your local computer. The files
      can be found under /etc/acdbdata

      * Note: I used the program "WinScp" to do this. However, you can also use the "scp" command on your Raspberry Pi terminal to copy these files over to your local computer.

   #. Open up ARC in offline mode (select "Open ACDB File on Disk" option).
      It will prompt you to select a workspace file. Select the workspace file
      that you copied from your Raspberry Pi.

   #. On the top left drop down menu displaying the usecases,
      select any usecase that uses "Headphones".

   #. Double click the "ALSA device sink" module shown below

      .. figure:: images/headphone_screenshot.png
         :figclass: fig-center
         :scale: 80 %

   #. On the menu that comes up, check the card_id. We want the card_id here to
      be the same as the ID that corresponds with the Headphones entry on the
      /proc/asound/cards file on your Raspberry Pi.

      .. figure:: images/alsa_sink_module.png
         :figclass: fig-center
         :scale: 80 %

      If it is not the same, update the value, and click "Set to ACDB" on the
      bottom for the changes to take effect.

   #. On the ARC menu, click "Save" on the top left to update your ACDB files.

   #. Copy the updated ACDB files back to your Raspberry Pi, and shutdown
      the system so the changes can take effect.

   #. Try running the usecase again.


