# Introduction #

This page gives technical details about the mapping of YANG modules to DSDL schemas and instance validation. It is a recommended reading for developers who intend to perform validation in their code. A good knowledge of the concepts of XML schema languages, in particular RELAX NG and Schematron, is assumed.

The [yang2dsdl](https://github.com/mbj4668/pyang/blob/master/bin/yang2dsdl) script that is a part of pyang distribution is an example of how the complete validation workflow can be integrated into a single application. Note, however, that the script was designed to be simple and doesn't use all the information provided by the low-level tools.

# Mapping YANG to DSDL Schemas #

[RFC 6110](http://tools.ietf.org/html/rfc6110) defines the standard mapping of a YANG data model to [DSDL](http://www.dsdl.org) schemas. That document, as well as its implementation in pyang, divide the mapping algorithm into two major steps:
  1. Mapping a YANG data model to the _hybrid schema_.
  1. Generating DSDL schemas (RELAX NG, DSRL and Schematron) from the hybrid schema.

## Hybrid Schema ##

The hybrid schema is an XML document which captures all aspects of the data model. It uses the RELAX NG syntax for defining grammatical and datatype constraints. However, it is _not_ a RELAX NG schema because, depending on the data model, it may contain schemas for multiple targets (document types). RELAX NG elements are annotated with additional information about semantic constraints and default values. Annotations are attributes or elements from the XML namespace with URI `urn:ietf:params:xml:ns:netmod:dsdl-annotations:1`.

In pyang, the hybrid schema is generated using the `dsdl` plugin, for example:

```
pyang -f dsdl -o turing-machine.dsdl turing-machine.yang
```

The output XML is unformatted, so a pretty-printer, such as “`xmllint --format`” might be useful for inspecting the hybrid schema.

## DSDL Schemas ##

From the hybrid schema, standard-compliant DSDL Schemas can be generated. In pyang, this second step is implemented via XSLT1, using the following stylesheets (found in the `xslt` subdirectory):

  * [gen-relaxng.xsl](https://github.com/mbj4668/pyang/blob/master/xslt/gen-relaxng.xsl) – generates the RELAX NG schema,
  * [gen-dsrl.xsl](https://github.com/mbj4668/pyang/blob/master/xslt/gen-dsrl.xsl) – generates the DSRL schema,
  * [gen-schematron.xsl](https://github.com/mbj4668/pyang/blob/master/xslt/gen-schematron.xsl) – generates the Schematron schema.

The stylesheet generating RELAX NG actually has to be applied twice. The following example uses the ubiquitous **xsltproc** tool, but other XSLT processors should work, too:
```
$ xsltproc -o turing-machine-gdefs-config.rng --stringparam target config \
> --stringparam gdefs-only 1 \
> $PYANG_XSLT_DIR/gen-relaxng.xsl turing-machine.dsdl
$ xsltproc -o turing-machine-config.rng --stringparam target config \
> $PYANG_XSLT_DIR/gen-relaxng.xsl turing-machine.dsdl
```

In the first invocation, with the `gdefs-only` parameter set to the value of `1`, the XSLT processor only extracts _global definitions_ (named patterns in RELAX NG terms). This step is necessary because these global definitions have to support the so-called “chameleon” design, in which a definition adopts the XML namespace of the grammar in which the definition is referenced (for a detailed explanation, see [Section 11.5](http://books.xmlschemata.org/relaxng/relax-CHP-11-SECT-5.html) in E. van der Vlist: _RELAX NG_, O'Reilly, 2003).

The second invocation then generates the actual schema for the selected target (`config` in our example). In it, the sub-schema of each YANG module inhabits a second-level grammar, and every such grammar then includes the schema with global definitions that was generated in the first invocation.

Care has to be taken when selecting names of output files into which the schemas are written, so as to allow the main schema to locate the schema file with global definitions. The rules for naming the files is explained in the manual page [yang2dsdl(1)](http://www.yang-central.org/twiki/pub/Main/YangTools/yang2dsdl.1.html).

The other two schemas, DSRL and Schematron, require only one invocation of a XSLT processor:
```
$ xsltproc -o turing-machine-config.dsrl --stringparam target config \
> $PYANG_XSLT_DIR/gen-dsrl.xsl turing-machine.dsdl 
$ xsltproc -o turing-machine-config.sch --stringparam target config \
> $PYANG_XSLT_DIR/gen-schematron.xsl turing-machine.dsdl
```

# Validation Procedure #

The procedure for validating instance XML documents or NETCONF messages against DSDL schemas is outlined in this picture:

![https://pyang.googlecode.com/svn/trunk/doc/tutorial/validation.png](https://raw.githubusercontent.com/mbj4668/pyang/master/doc/tutorial/validation.png)

To demonstrate the individual steps, we will validate the configuration document from [Creating an XML Instance](InstanceValidation#creating-an-xml-instance).

## RELAX NG ##

First of all, we have to validate the grammar and datatypes with a RELAX NG validator, such as **xmllint**:
```
$ xmllint --noout --relaxng turing-machine-config.rng turing-machine-config.xml
turing-machine-config.xml validates
```

Bear in mind that the type of the validated document must always match the target of the schema, in this case `config`.

## Document Schema Renaming Language (DSRL) ##

Before validating semantic constraints with Schematron, it is necessary to add default values to the instance document where possible. This is because YANG defines the context for evaluating **must** expressions and other semantic constraints so that all applicable default values are “in use” (see [Section 7.6.1](http://tools.ietf.org/html/rfc6020#section-7.6.1) in RFC 6020).

DSRL schema language has been added to the DSDL suite relatively recently, the standard was published in 2008. It is the only DSDL schema language that is allowed to change the XML information set of the validated document. One of the roles of DSRL is to define default contents for missing or empty elements, and this is exactly what we need.

As of yet, no generic implementation of DSRL is available, but pyang includes a XSLT stylesheet, [dsrl2xslt.xsl](https://github.com/mbj4668/pyang/blob/master/xslt/dsrl2xslt.xsl), which is a partial implementation – it converts the DSRL schema generated from a hybrid schema into another XSLT stylesheet, `add-defaults.xsl`, that performs the defauls-adding function.
```
$ xsltproc -o add-defaults.xsl $PYANG_XSLT_DIR/dsrl2xslt.xsl turing-machine-config.dsrl
```

The `add-defaults.xsl` stylesheet can now be applied to an XML instance document:
```
$ xsltproc -o tm-config-wd.xml add-defaults.xsl turing-machine-config.xml
```

The two XML files, `turing-machine-config.xml` and `tm-config-wd.xml`, differ only in those place where the original file has no `<head-move>` element in the output part of a transition rule. The processed file has the `head-move` element there with the default content of `right`:
```
<head-move><?dsrl?>right</head-move>
```

In the transition rules that have no output part at all, the `<output>` element is added as well:
```
<output><?dsrl?>
  <head-move>right</head-move>
</output>
```

Note that in both cases the added elements are tagged with the processing instruction `<?dsrl?>` so that Schematron (and other tools) can recognize the default contents that have been added by DSRL validation.

## Schematron ##

Rules in a Schematron schema specify various semantic constraints. The flexibility of Schematron follows from the expressive power of [XPath](http://www.w3.org/TR/1999/REC-xpath-19991116/) and [XSLT](http://www.w3.org/TR/1999/REC-xslt-19991116).

Another very useful feature is that error messages are defined as a part of Schematron rule specification. In our case, Schematron error messages are often mapped from the corresponding **error-message** statements in YANG modules. As a result, Schematron validation often gives quite specific and precise information about the context of a semantic validity problem.

Pyang distribution includes the open source ISO Schematron [implementation](https://code.google.com/p/schematron/) by Rick Jelliffe. It is an XSLT-based Schematron validator comprising three stylesheets in the `xslt` subdirectory: `iso_abstract_expand.xsl`, `iso_schematron_skeleton_for_xslt1.xsl` and `iso_svrl_for_xslt1.xsl`.

First, the Schematron schema is transformed into an XSLT stylesheet, `check-semantics.xsl` (in two steps, but this is an unimportant technical detail):
```
$ xsltproc $PYANG_XSLT_DIR/iso_abstract_expand.xsl turing-machine-config.sch | \
> xsltproc -o check-semantics.xsl $PYANG_XSLT_DIR/iso_svrl_for_xslt1.xsl -
```

The stylesheet can now be applied to the instance document with added defaults, `tm-config-wd.xml`:
```
xsltproc -o check-defaults.svrl check-semantics.xsl tm-config-wd.xml
```

The output file `check-defaults.svrl` introduces yet another XML format – Schematron Validation Report Language (SVRL). It describes the semantic validation results, in particular failed asserts or successful reports, together with context information for the corresponding Schematron rule.

Let's assume the instance configuration again has the same problem as in [InstanceValidation](InstanceValidation), namely that there are two `<delta>` entries with identical input parameters, `<state>` and `<symbol>`. The SVRL file then contains the following report.

```
  <svrl:successful-report 
    test="preceding-sibling::tm:delta[tm:input/tm:state=current()/tm:input/tm:state and
          tm:input/tm:symbol=current()/tm:input/tm:symbol]"
    location="/*[local-name()='config' and   
              namespace-uri()='urn:ietf:params:xml:ns:netconf:base:1.0']/
              *[local-name()='turing-machine' and     
              namespace-uri()='http://example.net/turing-machine']/
              *[local-name()='transition-function' and 
              namespace-uri()='http://example.net/turing-machine']/*
              [local-name()='delta' and 
              namespace-uri()='http://example.net/turing-machine'][8]">
    <svrl:text>
      Violated uniqueness for "tm:input/tm:state tm:input/tm:symbol"
    </svrl:text>
  </svrl:successful-report>
```

The `test` attribute shows the rule, and `location` contains an XPath context (albeit in a little complicated form) in which the rule was checked.

For the command-line interface of the **yang2dsdl** script we need a simple text output, so the following simplified report is extracted from the SVRL file using the XSLT stylesheet [svrl2text.xsl](https://github.com/mbj4668/pyang/blob/master/xslt/svrl2text.xsl):
```
$ xsltproc $PYANG_XSLT_DIR/svrl2text.xsl check-semantics.svrl
--- Validity error at "/nc:config/tm:turing-machine/tm:transition-function/tm:delta":
    Violated uniqueness for "tm:input/tm:state tm:input/tm:symbol"
```

# Implementing Validation on Devices #

The procedure described in the preceding sections looks somewhat daunting but, in fact, it can be effectively implemented even on small devices with relatively limited memory. As long as the data model supported by the device doesn't change, the only required software components are: a RELAX NG validator and an XSLT processor.

Two approaches are possible, or any combination thereof:
  1. Only the hybrid schema describing the current data model is stored on the device.
  1. The RELAX NG schema plus the two final XSLT stylesheets, `add-defaults.xsl` and `check-semantics.xsl` are generated in advance and stored on the device.

In the former case, the device just has to run more XSLT transformations.



<a href='Hidden comment: 
Local Variables:
mode: fundamental
mode: visual-line
End:
'></a>