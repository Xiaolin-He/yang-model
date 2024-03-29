# Introduction #

[RFC 6110](http://tools.ietf.org/html/rfc6110) defines a mapping from YANG to standard [DSDL](http://www.dsdl.org) schemas. Pyang's _dsdl_ plugin, together with several XSLT stylesheet that are also part of pyang distribution, fully implements the mapping procedure. The generated DSDL schemas can be used with generic off-the-shelf XML tools for both syntactic and semantic validation of XML instance documents.

This page explains the use of a shell script, [yang2dsdl](https://github.com/mbj4668/pyang/blob/master/bin/yang2dsdl), which is also provided in pyang distribution as a simple front end to the validation procedure. Developers that need more control or better error diagnostics should read [DsdlDetails](DsdlDetails).

# DSDL Schemas #

Let's apply the **yang2dsdl** script to our data model, [turing-machine.yang](https://github.com/mbj4668/pyang/blob/master/doc/tutorial/examples/turing-machine.yang), and generate a coordinated set of DSDL schemas:

```
$ yang2dsdl -t config turing-machine.yang
== Generating RELAX NG schema './turing-machine-config.rng'
Done.

== Generating Schematron schema './turing-machine-config.sch'
Done.

== Generating DSRL schema './turing-machine-config.dsrl'
Done.
```

As we can see from the output, three schemas are generated:

  * _RELAX NG_ schema (`turing-machine-config.rng`) describes the structure of the instance document (its grammar), and datatypes of leaf data nodes.

  * _Schematron_ schema (`turing-machine-config.sch`) represents semantic rules defined in the YANG data model.

  * _DSRL_ schema (`turing-machine-config.dsrl`) contains instructions that can be used for extending an instance XML document with default values.

The careful reader will notice that the script also created a fourth file, `turing-machine-gdefs-config.rng`, which is a part of the RELAX NG schema.

The command-line option `-t` of **yang2dsdl** defines the _target_ for the schemas. In this case, we instruct the script to generate schemas for the contents of a _configuration datastore_, i.e. configuration data only. Supported targets are:

<dl>
<blockquote><dt><b>data</b></dt>
<dd>datastore contents (configuration and state data) encapsulated in <code>&lt;nc:data&gt;</code> document element;</dd>
<dt><b>config</b></dt>
<dd>configuration datastore contents encapsulated in <code>&lt;nc:config&gt;</code> document element;</dd>
<dt><b>get-reply</b></dt>
<dd>a complete NETCONF message containing a reply to <code>&lt;nc:get&gt;</code> operation;</dd>
<dt><b>get-config-reply</b></dt>
<dd>a complete NETCONF message containing a reply to <code>&lt;nc:get-config&gt;</code> operation;</dd>
<dt><b>rpc</b></dt>
<dd>RPC request;</dd>
<dt><b>rpc-reply</b></dt>
<dd>RPC reply;</dd>
<dt><b>notification</b></dt>
<dd>event notification.</dd>
</dl></blockquote>

We will work with other targets later on in this tutorial.

# Creating an XML Instance #

Quite often, sample instance documents for testing and debugging purposes have to be written manually, in a text editor. Some people are fond of writing every single character in an XML or HTML document with their fingers. The rest of us will want to reduce the drudgery by using an editor that provides some support for editing XML documents.

An excellent choice is Emacs with James Clark's [nXML mode](http://www.thaiopensource.com/nxml-mode/). With the RELAX NG schema we have just generated, we can turn it into a schema-aware editor with autocompletion and on-the-fly validation. We only need to convert the schema into the RELAX NG [compact syntax](http://www.relaxng.org/compact-20021121.html), e.g. by using [Trang](http://www.thaiopensource.com/relaxng/trang.html):

```
trang -I rng -O rnc turing-machine-config.rng turing-machine-config.rnc
```

There are several ways for telling nXML mode how to locate the right schema for an XML document being edited. The easiest option is to rely on file names. In our case it means that the name of the XML file should be `turing-machine-config.xml`.

Now that everything is in place, we can write our first configuration for the Turing Machine, [turing-machine-config.xml](https://github.com/mbj4668/pyang/blob/master/doc/tutorial/examples/turing-machine-config.xml):

```
<?xml version="1.0" encoding="utf-8"?>
<!-- file: turing-machine-config.xml -->
<config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <turing-machine xmlns="http://example.net/turing-machine">
    <transition-function>
      <delta>
	<label>left summand</label>
	<input>
	  <state>0</state>
	  <symbol>1</symbol>
	</input>
      </delta>
      <delta>
	<label>separator</label>
	<input>
	  <state>0</state>
	  <symbol>0</symbol>
	</input>
	<output>
	  <state>1</state>
	  <symbol>1</symbol>
	</output>
      </delta>
      <delta>
	<label>right summand</label>
	<input>
	  <state>1</state>
	  <symbol>1</symbol>
	</input>
      </delta>
      <delta>
	<label>right end</label>
	<input>
	  <state>1</state>
	  <symbol/>
	</input>
	<output>
	  <state>2</state>
	  <head-move>left</head-move>
	</output>
      </delta>
      <delta>
	<label>write separator</label>
	<input>
	  <state>2</state>
	  <symbol>1</symbol>
	</input>
	<output>
	  <state>3</state>
	  <symbol>0</symbol>
	  <head-move>left</head-move>
	</output>
      </delta>
      <delta>
	<label>go home</label>
	<input>
	  <state>3</state>
	  <symbol>1</symbol>
	</input>
	<output>
	  <head-move>left</head-move>
	</output>
      </delta>
      <delta>
	<label>final step</label>
	<input>
	  <state>3</state>
	  <symbol/>
	</input>
	<output>
	  <state>4</state>
	</output>
      </delta>
    </transition-function>
  </turing-machine>
</config>
```

Note that the `<head-move>` element is only present with the value `left` because the opposite alternative, `right`, is the default value of the `head-dir` type.

By the way, this “program” adds two numbers that are written on the tape in the unary notation and separated by the zero symbol.

The schemas for the `config` target expect `<config>` (in the NETCONF namespace) as the document element. This is because YANG allows for multiple roots of the data hierarchy, and so an extra wrapping element is needed in order to make sure that the document is well-formed XML.

We can now use **yang2dsdl** again to verify that the above configuration is a valid instance of the data model:

```
$ yang2dsdl -s -j -b turing-machine -t config -v turing-machine-config.xml
== Using pre-generated schemas

== Validating grammar and datatypes ...
turing-machine-config.xml validates.

== Adding default values... done.

== Validating semantic constraints ...
No errors found.
```

Sure enough, everything is fine. The `-s` option indicates that we want to use the schemas that we generated earlier, and then we also have to provide the `-b` (base name) and `-t` (target) options, so that the script is able to find the schemas. The `-j` option instructs the script to use [Jing](http://www.thaiopensource.com/relaxng/jing.html) as the RELAX NG validator. By default, **xmllint** from the [libxml2](http://www.xmlsoft.org/) toolkit is used. While **xmllint** is more common and also works, its error messages are sometimes rather incomprehensible.

Now, let's smuggle some bugs into the instance document. First, we change the value of the input state element in the first transition rule (`left summand`) from `0` to the string `zero`. This is a type error because the data model says the `state` leaf is `uint16`. Type errors are caught by the RELAX NG validator:

```
$ yang2dsdl -s -j -b turing-machine -t config -v turing-machine-config.xml
== Using pre-generated schemas

== Validating grammar and datatypes ...
/Users/lhotka/sandbox/YANG/Turing/turing-machine-config.xml:9:23: error: character
content of element "state" invalid; must be an integer
```

Next, we will return the `<state>` element to the original correct value of `0`, and introduce a new, more subtle error – create a new entry of the `delta` list with the following content:

```
<delta>
  <label>illegal</label>
  <input>
    <state>0</state>
    <symbol>0</symbol>
  </input>
  <output>
    <head-move>left</head-move>
  </output>
</delta>
```

See what happens:

```
$ yang2dsdl -s -j -b turing-machine -t config -v turing-machine-config.xml
== Using pre-generated schemas

== Validating grammar and datatypes ...
turing-machine-config.xml validates.

== Adding default values... done.

== Validating semantic constraints ...
--- Validity error at "/nc:config/tm:turing-machine/tm:transition-function/tm:delta":
    Violated uniqueness for "tm:input/tm:state tm:input/tm:symbol"
```

Right, the YANG module states in [line 111](https://github.com/mbj4668/pyang/blob/master/doc/tutorial/examples/turing-machine.yang#L111) that the input parameters `state` and `symbol` must be unique among the entries of the `delta` list:

```
unique "input/state input/symbol";
```

However, this semantic constraint is not satisfied because two entries, `separator` and `illegal`, now have identical input parameters.

As an exercise, try to introduce other errors to the instance document and run the same command again.

# Validating RPC requests and replies #

Before starting the computation, we have to initialize the Turing Machine properly. It is easy because our data model defines a custom RPC method exactly for this purpose.

So, after installing the above configuration for adding two numbers, we might want to test it and compute, say, 2 + 3. The following RPC request does the initialization job:

```
<?xml version="1.0" encoding="utf-8"?>
<!-- file: initialize-rpc.xml -->
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
     message-id="1">
  <initialize xmlns="http://example.net/turing-machine">
    <tape-content>110111</tape-content>
  </initialize>
</rpc>
```

The `<tape-content>` element specifies the initial contents of the tape, i.e. numbers 2 and 3 in the unary notation, separated with a zero.

We can validate the above RPC message, too. In this case, we will generate the schemas for RPC replies and perform the validation in one step:

```
$ yang2dsdl -t rpc -v initialize-rpc.xml turing-machine.yang
== Generating RELAX NG schema './turing-machine-rpc.rng'
Done.

== Generating Schematron schema './turing-machine-rpc.sch'
Done.

== Generating DSRL schema './turing-machine-rpc.dsrl'
Done.

== Validating grammar and datatypes ...
initialize-rpc.xml validates

== Adding default values... done.

== Validating semantic constraints ...
No errors found.
```

Note that this time the target is `rpc`.

After starting the computation with the other custom RPC method, `run`, we can wait until we receive the `halted` notification indicating that the Turing Machine is done. Afterwards, we can learn the result by sending a `get` request. The Turing Machine's NETCONF server should reply with the following message:

```
<?xml version="1.0" encoding="utf-8"?>
<!-- file: turing-machine-get-reply.xml -->
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"
	   message-id="3">
  <data>
    <turing-machine xmlns="http://example.net/turing-machine">
      <state>4</state>
      <head-position>0</head-position>
      <tape>
	<cell>
	  <coord>0</coord>
	  <symbol>1</symbol>
	</cell>
	<cell>
	  <coord>1</coord>
	  <symbol>1</symbol>
	</cell>
	<cell>
	  <coord>2</coord>
	  <symbol>1</symbol>
	</cell>
	<cell>
	  <coord>3</coord>
	  <symbol>1</symbol>
	</cell>
	<cell>
	  <coord>4</coord>
	  <symbol>1</symbol>
	</cell>
	<cell>
	  <coord>5</coord>
	  <symbol>0</symbol>
	</cell>
      </tape>
      <transition-function>
	<delta>
	  <label>left summand</label>
	  <input>
	    <state>0</state>
	    <symbol>1</symbol>
	  </input>
	</delta>
	<delta>
	  <label>separator</label>
	  <input>
	    <state>0</state>
	    <symbol>0</symbol>
	  </input>
	  <output>
	    <state>1</state>
	    <symbol>1</symbol>
	  </output>
	</delta>
	<delta>
	  <label>right summand</label>
	  <input>
	    <state>1</state>
	    <symbol>1</symbol>
	  </input>
	</delta>
	<delta>
	  <label>right end</label>
	  <input>
	    <state>1</state>
	    <symbol/>
	  </input>
	  <output>
	    <state>2</state>
	    <head-move>left</head-move>
	  </output>
	</delta>
	<delta>
	  <label>write separator</label>
	  <input>
	    <state>2</state>
	    <symbol>1</symbol>
	  </input>
	  <output>
	    <state>3</state>
	    <symbol>0</symbol>
	    <head-move>left</head-move>
	  </output>
	</delta>
	<delta>
	  <label>go home</label>
	  <input>
	    <state>3</state>
	    <symbol>1</symbol>
	  </input>
	  <output>
	    <head-move>left</head-move>
	  </output>
	</delta>
	<delta>
	  <label>final step</label>
	  <input>
	    <state>3</state>
	    <symbol/>
	  </input>
	  <output>
	    <state>4</state>
	  </output>
	</delta>
      </transition-function>
    </turing-machine>
  </data>
</rpc-reply>
```

We can now read the result from the tape data, and we can also validate the `<get>` reply:

```
$ yang2dsdl -t get-reply -v turing-machine-get-reply.xml turing-machine.yang 
== Generating RELAX NG schema './turing-machine-get-reply.rng'
Done.

== Generating Schematron schema './turing-machine-get-reply.sch'
Done.

== Generating DSRL schema './turing-machine-get-reply.dsrl'
Done.

== Validating grammar and datatypes ...
turing-machine-get-reply.xml validates

== Adding default values... done.

== Validating semantic constraints ...
No errors found.
```



<a href='Hidden comment: 
Local Variables:
mode: fundamental
mode: visual-line
End:
'></a>