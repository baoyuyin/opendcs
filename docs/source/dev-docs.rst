##################
Developer Guidance
##################

This Document is part of the OpenDCS Software Suite for environmental
data acquisition and processing. The project home is:
https://github.com/opendcs/opendcs

See INTENT.md at the project home for information on licensing.

.. contents. Table of Contents
   :depth: 3


Overview
========

The purpose of this document is to describe how different technologies are used for OpenDCS development.
Extra attention is given to testing and using OpenDCS within containers.

The Ant Build
=============

Basics
------

Primarily due to history ant is used to manage the build of OpenDCS. The minimum version of Ant required 1.10.12 due to the use 
of a setting to generate friendlier reports from the junit tests.

The simplest way to verify the build is working is to run the following 'target':

.. code-block:: bash

    ant test

This will download all of the required dependencies, compile the code and run the available unit tests.
An html report of the tests will be available in :code:`build/reports/junit/test/`

The bulk of output from various targets will go into directories under :code:`build`

The basic workflow of the build is to do the following in the order of the following table. NOTE:
See build.xml and common.xml for full details. common.xml lists created paths in properties at 
the top of the file.

+-------------------------+----------------------------------------------------------------------+
|Target                   |Purpose                                                               |
+=========================+======================================================================+
|download-ivy             |Download the required Apache Ivy jars to handle dependency management.|
+-------------------------+----------------------------------------------------------------------+
|init-ivy                 |Setup ant task from ivy jars                                          |
+-------------------------+----------------------------------------------------------------------+
|resolve                  |Retrieve dependencies and organize classpaths for build               |
+-------------------------+----------------------------------------------------------------------+
|compile                  |Compiles the OpenDCS source from `src/main/java` and syncs resource   |
|                         |files into output directories under `build`                           |
+-------------------------+----------------------------------------------------------------------+
|jar                      |Bundles the main OpenDCS compiled code and resources into a .jar file.|
+-------------------------+----------------------------------------------------------------------+
|compile-test             |Compiles the test source code into class files                        |
+-------------------------+----------------------------------------------------------------------+
|test                     |Runs the junit5 test system and generates a report. The exit code of  |
|                         |the build will be 0 on success, and 1 on failure                      |
+-------------------------+----------------------------------------------------------------------+

Additional targets that can be run for testing.

+-------------------------+-------------------------------------------------------------------------+
|Target                   |Purpose                                                                  |
+=========================+=========================================================================+
|gui-test                 |Runs available tests on GUI elements. Requires an active                 |
|                         | desktop/windowing system or something like xvfb (X virtual frame buffer)|
+-------------------------+-------------------------------------------------------------------------+
|integration-test         |Runs tests against an active complete system.                            |
|                         | See `integration_test_infra`_ for more details                          |
+-------------------------+-------------------------------------------------------------------------+

During general development work you can pass :code:`-Dno.docs=true` on the ant commandline to skip doc generation

Other targets of integration-test

+-------------------------+-------------------------------------------------------------------------+
|Target                   |Purpose                                                                  |
+=========================+=========================================================================+
|stage                    |Preps the directory `stage` for creating the installer jar               |
+-------------------------+-------------------------------------------------------------------------+
|opendcs                  |Generates the install jar using izpack                                   |
+-------------------------+-------------------------------------------------------------------------+
|release                  |Generates and signs release artifacts for upload                         |
|                         |requires `-Dgpg.key.id` command line option                              |
+-------------------------+-------------------------------------------------------------------------+

Debugging OpenDCS
-----------------

All of the test tasks above can have :code:`-DdebugPort=<port number>`, where :code:`<port number>` is 
an integer between 1025 and 65534 without the < and > added to the ant command line. Thyen Your IDE can then attach to the JVM running 
the tests.

While we are trying to add more formal unit tests it is still often easier, and generally required while development, to step through
the code while manually testing things.

If you have an installation of OpenDCS already it can also be debugged in a similar way. NOTE: this is
known to work on linux/mac.

After you have an installation otherwise working start applications with the following:

.. code-block:: bash

    # For testing dbedit.
    DECJ_MAXHEAP="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=<port>" dbedit

    # For testing dbedit, but you're trying to figured out an issue during startup.
    DECJ_MAXHEAP="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=<port>" dbedit


You can then have your IDE attach to the JVM and it will stop on break points appropriately.

The following workflow can be used:

.. code-block:: bash

    # <see issue and make tweaks to code>
    ant jar
    cp build/lib/opendcs.jar <your current $DCSTOOL_HOME>/bin
    DECJ_MAXHEAP="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=<port>" <app>


And repeat as required. This works for the GUI and nogui applications.

MBeans
======

We have started implementing JMX MBeans for components within OpenDCS. You can connect to the process
using the jconsole application provided with your JDK to view the information.

CWMS
====

MBeans
------

The cwms connection pool implements the ConnectionPool Mbean. This MBean provides a view into the connections 
outstanding and available. Additional each Connection returned implements a WrappedConnectionMBean that shows
the current lifetime and can show where the connection pool was opened from.

Connection pool
---------------

CwmsDb using a connection pool mechanism. Leaks are a concern, if you working against a CWMS
system you can turn pool tracing on for an application with the following java flags:

.. code-block:: bash

    DECJ_MAXHEAP="-Dcwms.connection.pool.trace=true" routsched ...

With tracing on the WrappedConnectionMBean will show where a connection was created from. This useful for identifing 
what code to fix for connection pool leaks.

Authentication Sources
----------------------

Implementation
++++++++++++++

If the simple file based, or environment variable based credential sources are insufficient it is possible to create and 
load a new source without additional configuration.

To do so implement the following interfaces:

   org.opendcs.spi.authentication.AuthSource

   org.opendcs.spi.authentication.AuthSourceProvider

AuthSource handles actually creating the credentials properties. All current implementations provide "username" and "password"
as that is the only need.


AuthSourceProvider gives the source implementation a name and takes the
configuration string from the user.properties or decodes.properties and instantiates the AuthSource instance.

You must also add a file:

    META-INF/services/org.opendcs.spi.authentication.AuthSourceProvider

that contains the fully qualified class name of your new AuthSource.

Usage
+++++

To acquire the configured credentials the following can be used:

.. code-block:: java

    ...
    String authFileName = DecodesSettings.instance().DbAuthFile;

    try
    {
        Properties credentials = null;
        credentials = AuthSourceService.getFromString(authFileName)
                                        .getCredentials();
        // ... work using the credentials
    }
    catch(AuthException ex)
    {
        String msg = "Cannot read username and password from '"
            + authFileName + "' (run setDecodesUser first): " + ex;
        System.err.println(msg);
        Logger.instance().log(Logger.E_FATAL, msg);
        throw new DatabaseConnectException(msg);
    }
    ...


Code Analysis
-------------

Checkstyle, Spotbugs, and the PMD/CPD tools are available for anaylzing the code.

to run each do the following:

.. code-block:: bash

    # SpotBugs
    ant spotbugs
    # output will be in build/reports/spotbugs/spotbugs.html

    # Checkstyle
    ant Checkstyle
    # output will output to the terminal

    # CPD
    ant cpd
    # output will be in build/reports/pmd/cpd/cpd.txt

Only CPD is fast. checkstyle and SpotBugs are rather slow.

.. _integration_test_infra:

Integration Test infrastructure
===============================

OpenDCS now contains a framework for running integration tests. See the folder `src/test-integration` for the code.
The intent is to be a simple to use "Compatibility Toolkit" where a given implementation is only responsible for identifying
the OpenDCS concepts (DECODES, Timeseries, computations, etc) that it supports and handling instantiation of external resources
and setting up the configuration.

Framework
---------

There is set of code under :code:`org.opendcs.fixtures` that allows configuration and setup to take place and determine if a given 
test should be enabled or not and other per test tasks.

All new integration test classes should derive from :code:`org.opendcs.fixtures.ApptestBase`. This class is marked with the :code:`OpenDCSTestConfigExtension` 
and handles determining which OpenDCS implementation to run, and performing any required "installation and setup steps" needed.

Implementations should derive from :code:`org.opendcs.fixtures.spi.configuration.Configuration` and :code:`org.opendcs.spi.configuration.ConfigurationProvider`
and implement any required setup. All `Configurations` are given a temporary directory to create the `DCSTOOL_USERDIR` contents.
Application logs are all written into this directory.

Currently Implemented are OpenDCS-XML and OpenDCS-Postgres. OpenDCS-Postgres uses the (Testcontainers)[https://java.testcontainers.org] library
which requires docker. OpenDCS-XML only depends on the file system.

To run either use the following command:

.. code-block:: bash

    ant integration-test -Dno.doc=true -Dopendcs.test.engine=OpenDCS-XML
    # or 
    ant integration-test -Dno.doc=true -Dopendcs.test.engine=OpenDCS-Postgres

Adding tests
------------

New classes, or methods to existing classes, should go under :code:`org.opendcs.regression_tests`

Integration tests inherit from :code:AppTestBase. This simplifies access to resources, environment, properties, and methods as described below.

+--------------------------------------------+--------------------------------+
|Member Variable                             |Description                     |
+============================================+================================+
|@SystemStub\                                |variables from \                |
|protected final EnvironmentVariables \      |System.getenv \                 |
|environment = new EnvironmentVariables();   |that applications will see.     |
+--------------------------------------------+--------------------------------+
|@SystemStub\                                |variables from \                |
|protected final SystemProperties \          |System.getProperty \            |
|properties = new SystemProperties();        |that applications will see.     |
+--------------------------------------------+--------------------------------+
|@SystemStub\                                |Used to trap System.exit        |
|protected final SystemExit \                |calls to allow testing          |
|exit = new SystemExit();                    |without aborting the test run   |
+--------------------------------------------+--------------------------------+
|@ConfiguredField                            |Instance of the                 |
|protected Configuration configuration;      |:code:`Configuration` that was  |
|                                            |create for this run. Contains   |
|                                            |reference to user.properties and|
|                                            |other specific information.     |
|                                            |This is provided by default as  |
|                                            |almost all interactions will    |
|                                            |require access to the           |
|                                            |user.properties file            |
+--------------------------------------------+--------------------------------+
 

At the Class and method level the following annotations are available.

+--------------------------------------------+--------------------------------+
|Annotation                                  |Description                     |
+============================================+================================+
|DecodesConfigurationRequired                |List of database import files   |
|                                            |needed for tests to succeed.    |
|                                            |Can be set at the Class level,  |
|                                            |Method level, or both in which  |
|                                            |case the sets will be merged    |
+--------------------------------------------+--------------------------------+


Extension and other Junit information
-------------------------------------

The :code:`OpenDCSTestConfigExtension`, if it knows about a given type, will inject 
an instance of any field annotated with :code:`@ConfiguredField` as seen in the table 
above for the configuration.

The only other injected field is a :code:`TimeSeriesDb` which is Provided by the Configuration
and will already be valid and can be used directly for things like testing DaoObjects or null which 
indicates the implementation under test doesn't use the any of the timeseries database components.
A test may or may not require access to the :code:`TimeSeriesDb` and so is not provided by default.

The sample :code:LoadingAppDaoTestIT uses the :code:@EnableIfSql annotation that extends from Junit's :code:ExecutionCondition
to determine if the test should be run or not.

Additional ExecutionConditions and parameter injection will be added in the future as needed and as
we better identify concepts to map to vs implementation details.


Caveats
-------

OpenDCS supports several implementations, the XML database, the baseline Postgres and Oracle database, two additional Oracle Databases:
USBR's HDB and USACE's CWMS.

Each share the same fundamental concepts. However, portions of the implementation, like Site names
and Data type parameter names (e.g. are we measuring Stage, Elevation, Precipitation, etc) are handle differently.

These tests are intended to be independent of these concerns; however, the current tests getting merged in are for the baseline implementation
which was Derived from CWMS and directly shares naming and data labelling styles. Given a new implementation it is quite 
likely that work will be required to handle this situation. We will address this situation when it happens and you should
not be afraid to reach out in discussions if you are having difficulties.

Containers
==========

Theory of operation
-------------------

Each "application" will have it's own container, derived from a baseline image.
This allows organization while also minimizing downstream disk usage. The base image layer will be shared so each
application will only be a minor additional layer.

Some applications like LRGS, RoutingScheduler, CompProc will have a default CMD and parameters and be suitable for:

.. code-block:: bash

   docker run -d ...

To run as a service.

Other applications, like importts, complocklist, etc, will have an ENTRYPOINT and a user can call it like they normally would except prefixing with:

.. code-block:: bash

   docker run -v `pwd`/decodes.properties:/dcs_user/decodes.properties complocklist

NOTE: this is still a work in progress, we may switch or there will also be support for environment variables. However, the commandline apps will
likely not see common usage in docker directly.

The build
---------

The build is done in multiple stages. 

Stage 1 Build
^^^^^^^^^^^^^

The build uses the openjdk:8-jdk-bullseye image as it was easier to handle some of the basic dependencies. The documentation is not 
generated as it wouldn't be easily accessible anyways.

Stage 2 baseline
^^^^^^^^^^^^^^^^

This setups the basic "OpenDCS" install in /opt/opendcs. We use the openjdk:8-jre-alpine to save space for the final image.
We may experiment in the future with additional image reductions.

The baseline sets up the "DCSTOOL_HOME" directory in /opt/opendcs and alters the bin files with the appropriate full location.

The baseline "env.sh" script, our docker equivalent to opendcs.init, is added here.
The opendcs user, to avoid running as root, and group are added as well as the default entrypoint.

The build/stage directory is copied from the build stage

Stage 3+ lrgs
^^^^^^^^^^^^^

LRGSHOME  and LRGS_ADMIN_PASSWORD ENV variable is registered.
/lrgs_home volume is registered.
The default 16003 port is defined.

The runtime user is set to opendcs:opendcs

CMD is set to lrgs.sh

lrgs.sh handles first time setup, copy default config, initial admin user, and starting LRGS in the foreground.

The lrgs.lock file is currently ignored and docker just kills the process. Currently investigating better ways to 
handle shutdown. Will likely just add a flag to remove the lock file entirely.
