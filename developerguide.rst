Developers Guide
================

Pre-requisites
--------------

This guide assumes that the :doc:`installation <installation>` instructions have been followed 
and that working odin-control and odin-data modules are available on the filesystem, along with 
all of their dependencies.

Creating a New Detector Project
-------------------------------

Although detector projects can be created using any make system and with any structure, it is
always good practise to try and ensure consistency wherever possible.  For this reason there
is a recommended structure to any new Odin Detector project, which is presented below::

    |--<name>-detector
       |--control
       |--data
          |--cmake
          |--common
             |--include
          |--frameReceiver
             |--include
             |--src
             |--test
          |--frameProcessor
             |--include
             |--src
             |--test


To assist with the creation of a new detector project, there is a script available in the 
odin-data software repository which will create the directory structure and standard build 
files based on the answers to some general questions.  Follow the example below to create
the module for a detector called "dummy".::

    TODO: insert script execution here
    TODO: question 1 answers
    TODO: question 2 answers


One the script has completed it is possible to build both the Python and C++ parts of the 
detector, albeit with no implementation at this stage.  The next steps are to create the
control adapter and the data acquisition plugins.

Odin Control
------------

Developing a New Adapter
************************


Odin Data
---------

Developing a New FrameReceiver Decoder
**************************************


Developing a New FrameProcessor Plugin
**************************************

This is a step by step guide to creating a new Odin Data C++ plugin, complete with testing.
For the duration of this guide we will refer to the plugin as the DummyProcessPlugin.  The guide
assumes that we are starting from an empty project, called the dummy-detector.

Every plugin requires three files.

In the frameProcessor/include directory create the file DummyProcessPlugin.h.

In the frameProcessor/src directory create the file DummyProcessPlugin.cpp.

In the frameProcessor/src directory create the file DummyProcessPluginLib.cpp::

    cd <path to dummy detector>
    cd data/frameProcessor
    touch include/DummyProcessPlugin.h
    touch src/DummyProcessPlugin.cpp
    touch src/DummyProcessPluginLib.cpp

The DummyProcessPluginLib.cpp file is required only by the classloader within the FrameProcessor 
application.  Each plugin that is classloaded into a FrameProcessor must register itself by calling 
a macro, and this resolves any name mangling issues.  However, the macro cannot be built into into 
an application directly as this will result in compilation errors.  By adding the registration 
code into a separate file a developer is free to compile the main plugin class into a main 
application (e.g. for unit testing).

For the DummyDetector the DummyProcessPluginLib.cpp file should contain the following contents

.. code-block:: c++
    :linenos:

    #include "DummyProcessPlugin.h"
    #include "ClassLoader.h"

    namespace FrameProcessor
    {
        /**
        * Registration of this processor through the ClassLoader.  This macro
        * registers the class without needing to worry about name mangling
        */
        REGISTER(FrameProcessorPlugin, DummyProcessPlugin, "DummyProcessPlugin");
    } // namespace FrameReceiver

The file must include the header file for the main plugin and the header file for the ClassLoader 
class.  It then declares the FrameProcessor namespace before calling the REGISTER macro.  The 
macro takes three arguments.

* Base class - always FrameProcessorPlugin
* Child class - in this case the DummyProcessPlugin class
* String name - when the classloading is initiated the name is used as a key to store the object, this should always be the same as the name of the child class and within quotation marks.

The main plugin class must always be a subclass of the FrameProcessorPlugin class, ensuring 
that the method necessary to ineract with other plugins in a chain is implemented.  The table below 
presents all of the methods that can be overridden by the processing plugin with a description of 
what they are used for:

+--------------------+--------+---------------------------------+---------------------------------------------+
|Method Name         |Required|Parameters                       |Description                                  |
+====================+========+=================================+=============================================+
|process_frame       | Yes    |boost::shared_ptr<Frame> frame   | Must be implemented by subclass.            |
|                    |        |                                 | Called whenever a new frame has been pushed |
|                    |        |                                 | from the previous plugin.                   |
+--------------------+--------+---------------------------------+---------------------------------------------+
|configure           | No     |OdinData::IpcMessage& config,    | Optional.  If this plugin has any runtime   |
|                    |        |OdinData::IpcMessage& reply      | parameters that could be set from a client  |
|                    |        |                                 | then this method should be overridden and   |
|                    |        |                                 | used to act upon the parameters.            |
+--------------------+--------+---------------------------------+---------------------------------------------+
|requestConfiguration| No     |OdinData::IpcMessage& reply      | Optional.  If the plugin has overridden the |
|                    |        |                                 | configure method then this method should    |
|                    |        |                                 | also be implemented and it should reply with|
|                    |        |                                 | any configuration items.  This will allow a |
|                    |        |                                 | client to request the current configuration.|
+--------------------+--------+---------------------------------+---------------------------------------------+
|status              | No     |OdinData::IpcMessage& status     | Optional.  If the plugin has any status to  |
|                    |        |                                 | report then this method should be           |
|                    |        |                                 | overridden.                                 |
+--------------------+--------+---------------------------------+---------------------------------------------+
|get_version_major   | Yes    |                                 | Must return the integer value for the major |
|                    |        |                                 | version number of the plugin                |
+--------------------+--------+---------------------------------+---------------------------------------------+
|get_version_minor   | Yes    |                                 | Must return the integer value for the minor |
|                    |        |                                 | version number of the plugin                |
+--------------------+--------+---------------------------------+---------------------------------------------+
|get_version_patch   | Yes    |                                 | Must return the integer value for the patch |
|                    |        |                                 | version number of the plugin                |
+--------------------+--------+---------------------------------+---------------------------------------------+
|get_version_short   | Yes    |                                 | Must return a std::string value for a short |
|                    |        |                                 | version description of the plugin           |
+--------------------+--------+---------------------------------+---------------------------------------------+
|get_version_long    | Yes    |                                 | Must return a std::string value for a long  |
|                    |        |                                 | version description of the plugin           |
+--------------------+--------+---------------------------------+---------------------------------------------+


There is only one method that must be implemented by the plugin class, and this is the process_frame 
method.  The process_frame method is called whenever the previous plugin in the chain pushes its frame.
The method has a single variable which is a shared pointer to a frame object, containing the raw 
data plus any associated meta data.  Note the underlying raw data may still be a block of shared 
memory passed to the FrameProcessor by the FrameReceiver application, or it may be a newly allocated 
block of memory (or even a re-cycled block of memory).  The point is that the access to the frame 
is the same in all cases and so the plugin does not need to worry where the frame has arrived from 
and it does not need to worry about freeing or handling the memory; this is all taken care of by 
the underlying application.

Lets start by creating the class required to compile the plugin, the DummyProcessPlugin class.  First,
the header file (DummyProcessPlugin.h)

.. code-block:: c++
    :lineno-start: 1

    #ifndef FRAMEPROCESSOR_INCLUDE_DUMMYPROCESSPLUGIN_H_
    #define FRAMEPROCESSOR_INCLUDE_DUMMYPROCESSPLUGIN_H_
    
    #include <log4cxx/logger.h>
    #include <log4cxx/basicconfigurator.h>
    #include <log4cxx/propertyconfigurator.h>
    #include <log4cxx/helpers/exception.h>

    using namespace log4cxx;
    using namespace log4cxx::helpers;
    
    #include "FrameProcessorPlugin.h"
    
    namespace FrameProcessor {
    
        class DummyProcessPlugin : public FrameProcessorPlugin {
        public:
            DummyProcessPlugin();
        
            virtual ~DummyProcessPlugin();

            int get_version_major();
            int get_version_minor();
            int get_version_patch();
            std::string get_version_short();
            std::string get_version_long();

        private:
            void process_frame(boost::shared_ptr<Frame> frame);

        };

    } /* namespace FrameProcessor */

    #endif /* FRAMEPROCESSOR_INCLUDE_LATRDPROCESSPLUGIN_H_ */


And now lets create the source file (DummyProcessPlugin.cpp)

.. code-block:: c++
    :lineno-start: 1

    #include "DummyProcessPlugin.h"

    namespace FrameProcessor {

    DummyProcessPlugin::DummyProcessPlugin()
    {
    
    }

    DummyProcessPlugin::~DummyProcessPlugin()
    {

    }

    void DummyProcessPlugin::process_frame(boost::shared_ptr<Frame> frame)
    {

    }

    int DummyProcessPlugin::get_version_major()
    {
        return 0;
    }

    int DummyProcessPlugin::get_version_minor()
    {
        return 1;
    }

    int DummyProcessPlugin::get_version_patch()
    {
        return 0;
    }

    std::string DummyProcessPlugin::get_version_short()
    {
        return std::string("0.1.0");
    }

    std::string DummyProcessPlugin::get_version_long()
    {
        return std::string("0.1.0-testversion");
    }

    } /* namespace FrameProcessor */

Currently the version number is hardcoded in this plugin, but later we will see how 
to automatically get the version number from git (assuming the plugin module is under 
git revision control).
To compile the plugin the following lines should be added to the CMakeLists.txt file 
in the data/frameProcessor/src directory::

    set(CMAKE_INCLUDE_CURRENT_DIR on)
    ADD_DEFINITIONS(-DBOOST_TEST_DYN_LINK)

    # Setup include directories
    include_directories(${FRAMEPROCESSOR_DIR}/include ${ODINDATA_INCLUDE_DIRS}
        ${Boost_INCLUDE_DIRS} ${LOG4CXX_INCLUDE_DIRS}/.. ${ZEROMQ_INCLUDE_DIRS})

    # Add library for dummy process plugin
    add_library(DummyProcessPlugin SHARED DummyProcessPlugin.cpp DummyProcessPluginLib.cpp)

    # Add dummy process plugin library as an installation target
    install(TARGETS DummyProcessPlugin LIBRARY DESTINATION lib)


You should now be able to build the plugin as a shared library, although currently the 
plugin doesn't actually do anything useful.  cd to the project build directory and type make::

    [user@pc build]$ make
    Scanning dependencies of target DummyProcessPlugin
    [ 50%] Building CXX object frameProcessor/src/CMakeFiles/DummyProcessPlugin.dir/DummyProcessPlugin.cpp.o
    [100%] Building CXX object frameProcessor/src/CMakeFiles/DummyProcessPlugin.dir/DummyProcessPluginLib.cpp.o
    Linking CXX shared library ../../lib/libDummyProcessPlugin.so

Now that we have a compiling plugin, lets make it interact with the incoming frame by 
binning the image and then push out the result as a new frame.  The binning mode shall 
be set up as configurable runtime parameter and the status will return the number of 
frames that have been binned since the binning mode was last changed.

The plugin will need an internal member variable to store the binning mode, as well as 
a string name for the configuration parameter.  At this stage lets also add some standard 
logging facilities.  Add these declarations to the header

.. code-block:: c++

    private:
        LoggerPtr logger_;
        static const std::string CONFIG_BINNING_MODE;
        uint32_t binning_mode_;

and then define them in the class source file

.. code-block:: c++

    #include "DummyProcessPlugin.h"

    namespace FrameProcessor {

    const std::string DummyProcessPlugin::CONFIG_BINNING_MODE = "bin";

    DummyProcessPlugin::DummyProcessPlugin() :
        binning_mode_(1)
    {
        // Setup logging for the class
        logger_ = Logger::getLogger("FP.DummyProcessPlugin");
        logger_->setLevel(Level::getAll());
        LOG4CXX_TRACE(logger_, "DummyProcessPlugin constructor.");
    }
    ...

To provide runtime access to the bin mode variable we need to override the configure 
method, and to provide the corresponding query we need to override the requestConfiguration 
method.  The configure method is passed two IpcMessage object references, the config and 
reply.  The method will need to inspect the object to see if it contains the bin mode 
item, and if it does verify the value.  Any success or failure can be set in the reply 
object and this will ultimately return to the client that submitted the configuration. 
The configure method can be implemented as follows (don't forget to add the declaration 
into the header file):

.. code-block:: c++
    :lineno-start: 21

    void DummyProcessPlugin::configure(OdinData::IpcMessage& config, OdinData::IpcMessage& reply)
    {
        // Check for the binning mode
        if (config.has_param(DummyProcessPlugin::CONFIG_BINNING_MODE)) {
            int bin = config.get_param<int>(DummyProcessPlugin::CONFIG_BINNING_MODE);
            LOG4CXX_TRACE(logger_, "Bin mode change request to " << bin);
            // We will support bin modes 1,2,4,8
            if (bin == 1 || bin == 2 || bin == 4 || bin == 8){
                this->binning_mode_ = bin;
                // We can accpet this configuration
                reply.set_msg_type(OdinData::IpcMessage::MsgTypeAck);
                reply.set_msg_val(OdinData::IpcMessage::MsgValCmdConfigure);
            } else {
                // We need to reject this configuration
                reply.set_msg_type(OdinData::IpcMessage::MsgTypeNack);
                reply.set_msg_val(OdinData::IpcMessage::MsgValIllegal);
            }
        }
    }

The implementation above first checks the config object to confirm that it has the 
binning mode parameter (line 24).  If the parameter is present then the value is extracted 
(line 25) and then checked against our allowed values.  If the checks pass then we set the 
private member variable and setup the response as acknowledged (lines 28-32).  If the checks 
fail then the private member variable is not set and the response is set up to notify of 
the failure (lines 35-36).

At this stage it is a good idea to test that the plugin can be successfully loaded into a 
frameProcessor application and that it can be configured through it's control channel.  To 
perform these tests we will create a shell script to run the application and a JSON configuration 
file to load the plugin into the application.  Create a directory outside of the project and 
create a file called stFrameProcessor1.sh.  Give the file executable permission and add the 
following to the file

.. code-block:: bash

    #!/bin/bash

    SCRIPT_DIR="$( cd "$( dirname "$0" )" && pwd )"

    <path to odin-data>/prefix/bin/frameProcessor --ctrl=tcp://0.0.0.0:5004 --json_file=$SCRIPT_DIR/fp1.json -d 5

Replace the <path to odin-data> with the appropriate path to your odin-data installation.  This 
script also assumes that the odin-data module installs into the prefix directory, if this is not 
the case then update the path appropriately.  Important notes from the script above, is that 
the control port has been configured as a TCP socket on port 5004, and the debug level has been 
set very high, to level 5.

Now in the same directory create a file called fp1.json with the following contents

.. code-block:: json
    :lineno-start: 1

    [
    {
        "fr_setup": {
            "fr_ready_cnxn": "tcp://127.0.0.1:5001",
            "fr_release_cnxn": "tcp://127.0.0.1:5002"
        },
        "meta_endpoint": "tcp://*:5008"
    },
    {
        "plugin": {
             "load": {
                 "index": "dummy", 
                 "name": "DummyProcessPlugin", 
                 "library": "<path to dummy detector>/data/prefix/lib/libDummyProcessPlugin.so"
             }
        }
    },
    {
        "plugin": {
             "connect": {
                 "index": "dummy", 
                 "connection": "frame_receiver"
             }
        }
    }
    ]

This JSON configuration file is loaded by the application and executed during the 
initialisation phase.  There are 3 blocks (JSON objects) in an array, and each one 
is executed in turn.  The first block (line 2-8) sets up the connection between the 
frameProcessor application and the frameReceiver application, by specifying the ZeroMQ 
channels, as well as a connection to a meta writer.  More importantly for the current 
test, is the second block (line 9-17).  This tells the frameReceiver application to load 
the DummyProcessPlugin library, and index the plugin as "dummy".  Replace the <path to 
dummy detector> to your specific dummy-detector module path, and update the installation 
directory if you have specified a different installation location.  The third block 
(line 18-25) is used to connect the dummy plugin to the receiving plugin which is always 
present in a frameReceiver application.  The receiving plugin is the part of the application 
which handles incoming buffers from shared memory and ensures they are appropriately 
wrapped into Frame objects.

Once these two files have been created, run the bash script and you should see output 
similar to the following

.. code-block:: bash

   [user@pc bin]$ ./stFrameProcessor1.sh 
   8 [0x7fabca154b00] DEBUG FP.App null - Debug level set to  5
   8 [0x7fabca154b00] DEBUG FP.App null - Setting number of IO threads to 1
   8 [0x7fabca154b00] INFO FP.App null - frameProcessor version 1-2-0dls1-dirty starting up
   9 [0x7fabbb388700] DEBUG FP.FrameProcessorController null - Running IPC thread service
   10 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Constructing FrameProcessorController
   10 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Connecting meta RX channel to endpoint: 
   inproc://meta_rx
   11 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Configuration submitted: 
   {"params":{"ctrl_endpoint":"tcp://0.0.0.0:5004"},
   "msg_type":"illegal","msg_val":"illegal","id":0,"timestamp":"2020-01-30T11:54:24.163659"}
   11 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Connecting control channel to endpoint: 
   tcp://0.0.0.0:5004
   13 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Configuration submitted: 
   {"params":{"fr_setup":{"fr_ready_cnxn":"tcp://127.0.0.1:5001",
   "fr_release_cnxn":"tcp://127.0.0.1:5002"},
   "meta_endpoint":"tcp://*:5008"},"msg_type":"cmd","msg_val":"configure",
   "id":0,"timestamp":"2020-01-30T11:54:24.166187"}
   13 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Connecting meta TX channel to endpoint: 
   tcp://*:5008
   13 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Shared Memory Config: 
   Publisher=tcp://127.0.0.1:5002 Subscriber=tcp://127.0.0.1:5001
   13 [0x7fabca154b00] TRACE FP.SharedMemoryController null - SharedMemoryController constructor.
   13 [0x7fabca154b00] DEBUG FP.SharedMemoryController null - Connecting RX Channel to endpoint: 
   tcp://127.0.0.1:5001
   13 [0x7fabca154b00] DEBUG FP.SharedMemoryController null - Connecting TX Channel to endpoint: 
   tcp://127.0.0.1:5002
   13 [0x7fabca154b00] DEBUG FP.SharedMemoryController null - Registering timer for deferred shared 
   buffer configuration request
   13 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Configuration submitted: 
   {"params":{"plugin":{"load":{"index":"dummy","name":"DummyProcessPlugin",
   "library":"/path/to/dummy-detector/data/prefix/lib/libDummyProcessPlugin.so"}}},"msg_type":"cmd",
   "msg_val":"configure","id":0,"timestamp":"2020-01-30T11:54:24.167001"}
   16 [0x7fabca154b00] TRACE FP.DummyProcessPlugin null - DummyProcessPlugin constructor.
   16 [0x7fabca154b00] DEBUG FP.FrameProcessorController null - Configuration submitted: 
   {"params":{"plugin":{"connect":{"index":"dummy","connection":"frame_receiver"}}},"msg_type":"cmd",
   "msg_val":"configure","id":0,"timestamp":"2020-01-30T11:54:24.170097"}
   16 [0x7fabca154b00] INFO FP.FrameProcessorController null - Running frame processor
   1013 [0x7fabbb388700] DEBUG FP.SharedMemoryController null - Requesting shared buffer 
   configuration from frame receiver
