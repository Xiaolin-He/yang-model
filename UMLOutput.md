# Introduction #

In some circumstances it is useful to be able to visualize YANG models as UML diagrams.
Pyang supports two different ways of doing this:
  * Render png using [PlantUML](http://plantuml.sourceforge.net/): `pyang -f uml`
  * Generate AppleScript that renders a OmniGraffle UML document
    [OmniGraffle](https://www.omnigroup.com/omnigraffle/) : `pyang -f omni`

## Editable UML Graph in OmniGraffle
If you have OmniGraffle installed you can use this plugin to render a UML kind-of diagram in OmniGraffle.
The good thing here is that you can modify the layout, remove parts of the diagram etc to make it more good looking.

Use this in the following way:

1. Start OmniGraffle
2. Generate the AppleScript file from the YANG module
3. Run the AppleScript, this will open a OmniGraffle doc and render the UML diagram

```
$ pyang -f omni ietf-interfaces.yang -o ietf-interfaces.scpt
$ osascript ietf-interfaces.scpt
```
![https://raw.githubusercontent.com/mbj4668/pyang/master/doc/img/ietf-interfaces.png]
(https://raw.githubusercontent.com/mbj4668/pyang/master/doc/img/ietf-interfaces.png)

## PlantUML output ##

The uml plugin to pyang generates a file that can be read by [plantuml](http://plantuml.sourceforge.net/) r7997 (http://sourceforge.net/projects/plantuml/files/plantuml.7997.jar/download).

The following example converts the [ietf-netconf-monitoring](http://tools.ietf.org/html/rfc6022) module into a UML diagram:


```
$ pyang -f uml ietf-netconf-monitoring.yang -o ietf-netconf-monitoring.uml
$ java -jar plantuml.jar ietf-netconf-monitoring.uml 
$ open img/ietf-netconf-monitoring.png
```

On some platforms Java might spit out "java.lang.OutOfMemoryError: Java heap space"

Pass the `-Xmx<SIZE>m` to java like:

```
$ java -Xmx1024m -jar plantuml.jar ietf-netconf-monitoring.uml 
```


Please note that the uml output has numerous options to tweak and simplify the layout. To skip details try the `--uml-no=` parameter. In many cases you like to inline augments and groupings, use the `--uml-inline` parameters.

You can also pass several YANG files to the UML output format, but the files must be given in dependency order. (Multi-file does not work for module/sub-module dependancies)

![https://github.com/mbj4668/pyang/raw/master/doc/img/monitor1.png](https://github.com/mbj4668/pyang/raw/master/doc/img/monitor1.png)


Another example with some parts suppressed:
```
$ pyang -f uml --uml-no=circles,stereotypes,annotation ietf-netconf-monitoring.yang -o ietf-netconf-monitoring.uml
```

![https://github.com/mbj4668/pyang/raw/master/doc/img/monitor2.png](https://github.com/mbj4668/pyang/raw/master/doc/img/monitor2.png)

A multi-file example from the ongoing YANG modeling in [Metro Ethernet Forum](http://metroethernetforum.org/)

![https://github.com/mbj4668/pyang/raw/master/doc/img/mef-8021ag_mef-soam-fm.png](https://github.com/mbj4668/pyang/raw/master/doc/img/mef-8021ag_mef-soam-fm.png)

A complex example based on the [Configuration Data Model for IPFIX and PSAMP](http://www.rfc-editor.org/internet-drafts/draft-ietf-ipfix-configuration-model-08.txt)
![https://github.com/mbj4668/pyang/raw/master/doc/img/ietf-ipfix-psamp.png](https://github.com/mbj4668/pyang/raw/master/doc/img/ietf-ipfix-psamp.png)



There is also a python script to generate a UML package structure for all yangs in a folder.

`uml-utilities/uml-pkg`

Use it in the following way:
```
$ ls *.yang | xargs -n1 pyang -f depend > packages.txt
$ uml-pkg --title=TITLE --inputfile=packages.txt > packages.uml
$ java -jar plantuml.jar packages.uml
$ open img/packages.png


```