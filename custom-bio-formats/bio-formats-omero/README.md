Bio-Formats OMERO
=================

A simple POM allowing to update an OMERO server with a new version of
Bio-Formats.

**Warning**: running the command below will replace the JARs under an
existing binary distribution. The current implementation will copy the
dependencies defined by the Bio-Formats components POMs ignoring any exclusion
defined at the OMERO level. Use at your own risk!

To update an OMERO.server, simply run:

     mvn clean package

Additional two properties can be overriden and passed as `-Dkey=value` to
the command above:

- `omero.home` specified the root directory of the OMERO server (default: `OMERO.server`)
- `bioformats.version` specifies the new version of Bio-Formats to replace into the OMERO.server libraries (server and client)

e.g.::

    mvn clean package -Domero.home=~/Downloads/OMERO.server -Dbioformats.version=5.7.0
