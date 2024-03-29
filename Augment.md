# Introduction #

The possibility to augment an existing data model with additional nodes and trees is one of the most powerful features of the YANG language. While standard XML schema languages, such as W3C XML Schema (XSD) or RELAX NG, also support extensible schemas, the data modeller has to arrange the base schema in a special way so as there'd be a chance for adding new elements or attributes in a certain place. In contrast, a YANG module can be augmented almost in any place with nodes defined in another module, and the receiving module needn't be prepared to it in any special way.

# The **augment** Statement #

Data nodes that are intended to augment an existing module have to be defined inside the [augment](http://tools.ietf.org/html/rfc6020#section-7.15) statement. The argument of this statement is a _schema node identifier_, i.e. a path to the target node (usually a container) to which the new nodes are to be added.

YANG also allows the **augment** statement to be used in a submodule for augmenting the main module or other submodules, and also to augment the contents of a grouping when it is used. In the following text, though, we will concentrate on the most common case when one YANG module augments the data hierarchy of another module. In this case, the augmenting nodes keep the namespace of the module where they are defined, so no name conflicts are possible. However, it is a matter of good style to avoid sibling nodes with identical local names.

The **augment** statement can also have **when** as its substatement. The augment is then applied only conditionally, depending on the contents of the receiving module.

# Augmenting the Turing Machine Data Model #

Assume we want to extend the data model of the Turing Machine by adding a second tape. This is accomplished by four **augment** statements contained in the module [second-tape.yang](https://github.com/mbj4668/pyang/blob/master/doc/tutorial/examples/second-tape.yang). The augmented data model looks schematically as follows:
```
$ pyang -f tree turing-machine.yang second-tape.yang
module: turing-machine
   +--rw turing-machine
      +--ro state                  state-index
      +--ro head-position          cell-index
      +--ro tape
      |  +--ro cell* [coord]
      |     +--ro coord     cell-index
      |     +--ro symbol?   tape-symbol
      +--rw transition-function
      |  +--rw delta* [label]
      |     +--rw label     string
      |     +--rw input
      |     |  +--rw state          state-index
      |     |  +--rw symbol         tape-symbol
      |     |  +--rw t2:symbol-2?   tm:tape-symbol
      |     +--rw output
      |        +--rw state?            state-index
      |        +--rw symbol?           tape-symbol
      |        +--rw head-move?        head-dir
      |        +--rw t2:symbol-2?      tm:tape-symbol
      |        +--rw t2:head-move-2?   tm:head-dir
      +--ro t2:head-position-2?    tm:cell-index
      +--ro t2:tape-2
         +--ro t2:cell* [coord]
            +--ro t2:coord     cell-index
            +--ro t2:symbol?   tape-symbol
rpcs:
   +---x initialize    
   |  +--ro input     
   |     +--ro tape-content?        string
   |     +--ro t2:tape-content-2?   string
   +---x run           
notifications:
   +---n halted    
      +--ro state    state-index
```

Let's have a look at each **augment** in turn.

## Operational State Data ##

We essentially need to duplicate the state of the original tape and its read/write head.
```
augment "/tm:turing-machine" {
  description
    "State data for the second tape.";
  leaf head-position-2 {
    config "false";
    type tm:cell-index;
    description
      "Head position of the second tape.";
  }
  container tape-2 {
    description
      "Contents of the second tape.";
    config "false";
    uses tm:tape-cells;
  }
}
```

The target node specified in the argument of the **augment** statement is `/tm:turing-machine`. This means that the nodes defined inside the **augment** statement will appear directly under the `turing-machine` container.

Node that in the definitions of new data nodes, `head-position-2` and `tape-2`, we are reusing a derived type (`cell-index`) and a grouping (`tape-cells`) from the original module [turing-machine.yang](https://github.com/mbj4668/pyang/blob/master/doc/tutorial/examples/turing-machine.yang). This is made possible by importing the _turing-machine_ module in the _second-tape_ module.

In the case of the grouping, it is important to understand what's going on in terms of namespaces. YANG groupings behave as chameleons – data nodes defined in a grouping adopt the namespace of the module in which the grouping is used (via the **uses** statement). Hence, the `cell` list as a child of the `tape-2` container is in the namespace of the _second-tape_ module, and keeps it when it is inserted by the **augment** statement into the data hierarchy of the _turing-machine_ module. In the tree representation above, this is indicated by the `t2` prefix that is prepended to the `cell` node and its subnodes.

## Configuration ##

The transition function of the extended Turing Machine has to take the symbol read from the second tape as a new input parameter:
```
augment
  "/tm:turing-machine/tm:transition-function/tm:delta/tm:input" {
  description
    "A new input parameter.";
  leaf symbol-2 {
    type tm:tape-symbol;
    description
      "Symbol read from the second tape.";
  }
}
```

Similarly, two additional output values have to be returned, namely the symbol to be written to the second tape, and the direction in which the head of the second tape shall move:
```
augment
  "/tm:turing-machine/tm:transition-function/tm:delta/tm:output" {
  description
    "New output parameters.";
  leaf symbol-2 {
    type tm:tape-symbol;
    description
      "Symbol to be written to the second tape. If this leaf is not
       present, the symbol doesn't change.";
  }
  leaf head-move-2 {
    type tm:head-dir;
    description
      "Move the head on the second tape one cell to the left or
       right.";
  }
}
```

In both **augment** statements we are again using derived types defined in the _turing-machine_ module.

## RPC Method ##

Finally, we need to “patch” the `initialize` method so as to be able to initialize the contents of the second tape as well:
```
augment "/tm:initialize/tm:input" {
  description
    "A new RPC input parameter.";
  leaf tape-content-2 {
    type string;
    description
      "Initial content of the second tape.";
  }
}
```

# Extended Configuration #

The following is an instance configuration for the Turing Machine with two tapes. It is a straightforward algorithm for comparing two strings consisting of the symbols `a` and `b`, each written to one tape.
```
<?xml version="1.0" encoding="utf-8"?>
<!-- file: tm-2-tapes-config.xml -->
<config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <turing-machine xmlns="http://example.net/turing-machine"
		  xmlns:t2="http://example.net/turing-machine/tape-2">
    <transition-function>
      <delta>
	<label>'a' on both tapes</label>
	<input>
	  <state>0</state>
	  <symbol>a</symbol>
	  <t2:symbol-2>a</t2:symbol-2>
	</input>
      </delta>
      <delta>
	<label>'b' on both tapes</label>
	<input>
	  <state>0</state>
	  <symbol>b</symbol>
	  <t2:symbol-2>b</t2:symbol-2>
	</input>
      </delta>
      <delta>
	<label>end of string on both tapes</label>
	<input>
	  <state>0</state>
	  <symbol/>
	  <t2:symbol-2/>
	</input>
	<output>
	  <state>1</state>
	</output>
      </delta>
    </transition-function>
  </turing-machine>
</config>
```

If both strings are identical, the Turing machine will halt in the state `1`, otherwise it will halt in the state `0`.

**Exercise:** Write a configuration for the original single-tape Turing Machine that performs the same task.

It is easy to modify the procedure described in [InstanceValidation](InstanceValidation), and validate the above configuration:
```
$ yang2dsdl -j -t config -v tm-2-tapes-config.xml turing-machine.yang second-tape.yang 
== Generating RELAX NG schema './turing-machine_second-tape-config.rng'
Done.

== Generating Schematron schema './turing-machine_second-tape-config.sch'
Done.

== Generating DSRL schema './turing-machine_second-tape-config.dsrl'
Done.

== Validating grammar and datatypes ...
tm-2-tapes-config.xml validates.

== Adding default values... done.

== Validating semantic constraints ...
No errors found.
```

In order to compare strings `abba` and `baba`, we have to initialize the extended Turing Machine with this message:
```
<?xml version="1.0" encoding="utf-8"?>
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
     message-id="42">
  <initialize xmlns="http://example.net/turing-machine"
	      xmlns:t2="http://example.net/turing-machine/tape-2">
    <tape-content>abba</tape-content>
    <t2:tape-content-2>baba</t2:tape-content-2>
  </initialize>
</rpc>
```



<a href='Hidden comment: 
Local Variables:
mode: fundamental
mode: visual-line
End:
'></a>