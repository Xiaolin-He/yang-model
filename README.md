
# Introduction #

This is a tutorial-style guide to the development of [YANG](http://tools.ietf.org/html/rfc6020) data models and corresponding data instances – configurations, state data, RPC requests or replies, and notifications.

-  [Augment](./Augment.md)
-  [Development](./Development.md)
-  [Documentation](./Documentation.md)
-  [Downloads](./Downloads.md)
-  [DsdlDetails](./DsdlDetails.md)
-  [InstanceValidation](./InstanceValidation.md)
-  [JavaScript-Tree](./JavaScript-Tree.md)
-  [Pyang-installation-on-MS-Windows](./Pyang-installation-on-MS-Windows.md)
-  [TreeOutput](./TreeOutput.md)
-  [UMLOutput](./UMLOutput.md)
-  [XmlJson](./XmlJson.md)

# YANG Data Model #

A data model is fully determined by the following information:
  * one or more YANG modules
  * optional _features_ that are supported by the device

Features are defined in YANG modules and allow for making selected parts of the data model schema conditional (see [Section 5.6.2](http://tools.ietf.org/html/rfc6020#section-5.6.2) in RFC 6020 for details).

YANG modules comprising the data model are entered on the pyang command line, and features may be specified using the `--features` option. This option can be used repeatedly – each occurrence defines active (supported) features for one YANG module.

For example, the following command line uses two YANG modules, _foo_ and _bar_, and the only supported feature is `miracle` (defined in _foo_).

```
$ pyang -f tree --features foo:miracle --features bar: foo.yang bar.yang
```

If no `--features` option is present for a module, then all features defined in that module are considered active.

For data models that consist of a larger number of YANG modules and/or features, it may be inconvenient to provide all data model information on the command line. Therefore, it is also possible to specify the data model in a configuration file. The contents of the configuration file are the same as the **hello** message that is sent by the NETCONF server at the start of every session – see RFC 6241, [Section 8.1](http://tools.ietf.org/html/rfc6241#section-8.1), and RFC 6020, [Section 5.6.4](http://tools.ietf.org/html/rfc6020#section-5.6.4).

This command line achieves the same effect as the previous one:

```
$ pyang -f tree --hello hello.xml
```

where the contents of `hello.xml` are as follows:

```
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
    <capability>
      http://example.com/foo?module=foo&amp;features=miracle
    </capability>
    <capability>
      http://example.com/bar?module=bar
    </capability>
  </capabilities>
</hello>
```

See [pyang(1)](http://www.yang-central.org/twiki/pub/Main/YangTools/pyang.1.html) manual page for more information.

# Editing YANG Modules #

YANG modules can be edited in any text editor capable of saving plain text (without formatting information). Add-ons for popular editors are available with varying degrees of support for YANG syntax:
  * [yang.vim](http://www.yang-central.org/twiki/pub/Main/YangTools/yang.vim) – syntax file for Vim,
  * [yang-mode.el](https://github.com/mbj4668/yang-mode) – Emacs mode for YANG, based on CC mode.
  * [yang-npp](https://github.com/vkosuri/lang-yang) - YANG modeling language syntax highlighting for Notepad++

Speaking about Emacs, the amazing [nXML mode](http://www.thaiopensource.com/nxml-mode/) makes a case for editing YANG modules in the alternative XML-based _YIN_ syntax (see RFC 6020, [Section 11](http://tools.ietf.org/html/rfc6020#section-11)). This approach is described [here](https://gitlab.labs.nic.cz/labs/yang-tools/wikis/editing_yang).

## Formatting YANG Modules ##

pyang can be used to consistently format YANG modules.  Use:

```
$ pyang -f yang --yang-canonical --keep-comments foo.yang
```

to get a formatted YANG module.  Note that some lines in the resulting
module may need additional manual tweaking to look good.  For example,
pyang cannot currently break long paths into multiple lines.

pyang can also be used as a linter, and to check the line length:

```
$ pyang --lint --max-line-length 69 foo.yang
```

# Example Data Model #

We will demonstrate pyang functions on a data model of the [Turing Machine](http://en.wikipedia.org/wiki/Turing_machine), defined in the YANG module [turing-machine.yang](https://github.com/mbj4668/pyang/blob/master/doc/tutorial/examples/turing-machine.yang). It is relatively simple but comprehensive in that it covers all document/message types supported in YANG – configuration and state data, RPCs and notifications.

The reader should imagine there is a Turing Machine device that runs a NETCONF server, so that we can use a NETCONF client to configure the machine and communicate with it.

To see an overview of the data model schemas, we can use the `tree` plugin of pyang:

```
$ pyang -f tree turing-machine.yang
module: turing-machine
   +--rw turing-machine
      +--ro state                  state-index
      +--ro head-position          cell-index
      +--ro tape
      |  +--ro cell* [coord]
      |     +--ro coord     cell-index
      |     +--ro symbol?   tape-symbol
      +--rw transition-function
         +--rw delta* [label]
            +--rw label     string
            +--rw input
            |  +--rw state     state-index
            |  +--rw symbol    tape-symbol
            +--rw output
               +--rw state?       state-index
               +--rw symbol?      tape-symbol
               +--rw head-move?   head-dir
rpcs:
   +---x initialize    
   |  +--ro input     
   |     +--ro tape-content?   string
   +---x run           
notifications:
   +---n halted    
      +--ro state    state-index
```

Use “`pyang --tree-help`” to see the explanation of symbols used in the tree representation.

The Turing Machine is represented by several _operational state_ nodes (denoted with `ro`):
  * _state_ – current state of the control unit,
  * _head-position_ – current position of the read/write head,
  * _tape_ – sparse list of cells with their coordinates and contained symbols (blank cells do not appear in the list).

The _configuration_ is a specification of the transition function 𝛿, i.e. a program for the Turing Machine. Each transition rule has two input parameters – the current state of the control unit and the symbol read from the current tape cell – and three output values: the new state, new symbol written to the tape cell, and new position of the head (at every step, it must move one cell to the left or to the right from the current cell).

The data model also defines two RPC methods:
  * **initialize** – puts the control unit into the initial state, the read/write head to its “home” position, and initializes the contents of the tape.
  * **run** – starts Turing Machine's computation.

Finally, there is an event notification, **halted**, which is triggered when the Turing Machine halts. This happens whenever no transition rule is defined for the current state and tape symbol. The notification reports the clock time when the machine halted and the current state.

Further information about the data model can be found in description texts of the [module](https://github.com/mbj4668/pyang/blob/master/doc/tutorial/examples/turing-machine.yang).

# What's Next? #

The following tutorial pages describe how a data model can used, together with pyang, for accomplishing various tasks. The pages are independent and can be read in any order:
  * [InstanceValidation](InstanceValidation) – validation of XML instances,
  * [Augment](Augment) – augmenting the data model,
  * [XmlJson](XmlJson) – schema-aware conversion of XML data to JSON and back.
