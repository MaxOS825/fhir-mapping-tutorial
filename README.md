# FHIR tutorial mapping example
FHIR has a mapping language to convert between logical models an a nice [tutorial](http://build.fhir.org/mapping-tutorial.html) how use it, this projects tries to build the structure definitions and mappings for it.

ATTENTION: work in progress, to be run the fhir source from gforge has to checked out and run. 

The [validator](http://build.fhir.org/validation.html) has also the possiblity to run the transforms, this functionality however is in the current build (date 06-06-2017) not activated.

```
java -jar org.hl7.fhir.validator.jar ../maptutorial/step1/source/source1.xml -ig ../maptutorial/st
ep1/logical/structuredefinition-tleft.xml -ig ../maptutorial/step1/logical/structuredefinition-tright.xml -defn igpack.zip -transform -map
 ../maptutorial/step1/map/step1.map -output ./step1.xml
```


## java based testing setup

eclipse based configuration

* import projects fhir/build/implementations/java/org.hl7.fhir.r4
* import projects fhir/build/implementations/java/org.hl7.fhir.utilities
* check out this project into your eclipse workspace and adjust the paths in
src/ch/ahdis/fhir/r4/test/TutorialMapTests.java


## run the tutorial
for each step there is a directory below the maptutorial directory

logical: structurdefinitions for tleft, tright

map: maps

source: files used to transform

target: result of map transform (will be generated by the unittests)

To run the tutorial: right-click on `TutorialMapTests.java`, select `Run As` and pick `JUnit Test`.


## step1 - ok

TLeft --> TRight
```xml
<?xml version="1.0" encoding="UTF-8"?>
<TLeft xmlns="http://hl7.org/fhir/tutorial">
  <a>test</a>
</TLeft>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<TRight xmlns="http://hl7.org/fhir/tutorial">
  <a>test</a>
</TRight>
```

## step2 - ok

## step3 - fails

note: did not use the structure definition like in steps1 and steps2 but simplified that the value
is an attribute and not the content of the element.


```xml
<TLeft xmlns="http://hl7.org/fhir/tutorial">
	<a2 value="012345678901234567890012345678901234567890012345678901234567890" />
</TLeft>
```

```
rule_a20b: for source.a2 as a where a2.length() <= 20 make target.a2 = a
rule_a20c: for source.a2 as a check a2.length() <= 20 make target.a2 = a
```
change it to:
```
rule_a20b: for source.a2 as a where $this.length() <= 20 make target.a2 = a
rule_a20c: for source.a2 as a check $this.length() <= 20 make target.a2 = a
```
looks like the FhirPathEnginge cannot evaluate a2 (L)
Evaluate {'$this.length() < 20'} on [a2=string[012345678901234567890012345678901234567890012345678901234567890]]

tried to create a patch for FHIRPathEngine L878 (private List<Base> execute(ExecutionContext context, List<Base> focus, ExpressionNode exp, boolean atEntry) throws FHIRException { )

however the examples in build/implemtations/r2maps in the build have also the $this parameter

## step4 - fails

```
"rule_a21a" : for source.a21 as a make target.a21 = cast(a, "integer") // error if it's not an integer
"rule_a21b" : for source.a21 as a where a1.isInteger make target.a2 = cast(a, "integer") // ignore it
"rule_a21c" : for source.a21 as a where not at1.isInteger make target.a21 = 0 // just assign it 0
```

- Transform cast not supported yet. added integer functionality with patch.
- isInteger is not supported in fhirpath? only option to use matches with regex
- regex: how to escape \ in map defintion for (\+|-)?\d+ (does give an error message if you using \d or \\d)
- regex: function matches implementation does not work? changed it, but not sure if is correct

changed it to:

```
  rule_a21a: for source.a21 as a make target.a21 = cast(a, "integer")
  rule_a21b: for source.a21 as a where a21.matches("[0-9]+") make target.a21 = cast(a, "integer") 
  rule_a21c: for source.a21 as a where (a21.matches("[0-9]+")).not() make target.a21 = 0
```

## step5 - ok 
  
## step6 - ok

## step7 - ok, fails alternative apporach

first map ok, the alternative approach has not the correct syntax  ab_content{s_aa, t_aa) , adjusted:

```
map "http://hl7.org/fhir/StructureMap/tutorial7" = "tutorial"

uses "http://hl7.org/fhir/StructureDefinition/tutorial-left" as source
uses "http://hl7.org/fhir/StructureDefinition/tutorial-right" as target

group tutorial
  input source : TLeft as source
  input target : TRight as target

rule_aa : for source.aa as s_aa make target.aa as t_aa then ab_content(s_aa, t_aa)  // make aa exist

endgroup

group ab_content
  input src as source
  input tgt as target

  rule_ab : for src.ab as ab make tgt.ab = ab // copy ab inside aa
endgroup
```
 
## step 8 - ok
  
conceptmap can be embedded in map: 

```
map "http://hl7.org/fhir/StructureMap/tutorial8" = "tutorial"

conceptmap "tutorialmap" {

  prefix s = "http://hl7.org/fhir/tutorial8/codeleft"
  prefix t = "http://hl7.org/fhir/tutorial8/coderight"

  s:vonhier = t:"nach-da"
  s:test = t:test
}

uses "http://hl7.org/fhir/StructureDefinition/tutorial-left" as source
uses "http://hl7.org/fhir/StructureDefinition/tutorial-right" as target

group tutorial
  input source : TLeft as source
  input target : TRight as target


  rule_d : for source.d as d make target.d = translate(d, '#tutorialmap', 'code')

endgroup
```

## step 9 - fails

changed mapping to
```
map "http://hl7.org/fhir/StructureMap/tutorial9" = "tutorial"

uses "http://hl7.org/fhir/StructureDefinition/tutorial-left" as source
uses "http://hl7.org/fhir/StructureDefinition/tutorial-right" as target

group tutorial
  input source : TLeft as source
  input target : TRight as target

  rule_i1 : for source as s where $this.m < 2 make target.j = evaluate(s,"i")
  rule_i2 : for source as s where $this.m >= 2 make target.k = evaluate(s,"i")

endgroup
```







