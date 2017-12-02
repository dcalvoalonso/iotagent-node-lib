# Measurement Transformation Expression Language

## Index

* [Overview](#overview)
* [Measurement transformation](#transformation)
  * [Expression definition](#definition)
  * [Variable values](#values)
  * [Expression execution](#execution)
* [Language description](#description)
  * [Types](#types)
  * [Values](#values)
  * [Allowed operations](#operations)
* [Examples of expressions](#examples)
* [NGSIv2 support](#ngsiv2)

## <a name="overview"/> Overview
The IoTAgent Library provides an expression language for measurement transformation, that can be used to adapt the
information coming from the South Bound APIs to the information reported to the Context Broker. Expressions in this
language can be configured for provisioned attributes as explained in the Device Provisioning API section in the
main README.md.

## <a name="transformation"/> Measurement transformation

### <a name="definition"/> Expression definition

Expressions can be defined for Active attributes, either in the Device provisioning or in the Configuration provisioning.
The following example shows a device provisioning payload with defined expressions:
```
{
   "devices":[
      {
         "device_id":"45",
         "protocol":"GENERIC_PROTO",
         "entity_name":"WasteContainer:WC45",
         "entity_type":"WasteContainer",
         "attributes":[
            {
               "name":"location",
               "type":"geo:point",
               "expression": "${@latitude}, ${@longitude}"
            },
            {
               "name":"fillingLevel",
               "type":"Number",
               "expression": "${@level / 100}"
            },
            {
               "object_id":"tt",
               "name":"temperature",
               "type":"Number"
            }
         ]
      }
   ]
}

```

The value of the `expression` attribute is a string that can contain any number of expression patterns. Each expression
pattern is marked with the following secuence: `${<expression>}` where `<expression>` is a valid construction of the
Expression Language (see definition [below](#description)). In order for the complete expression to be evaluated, all the
expression patterns must be evaluatable (there must be a value in the measurement for all the variables of all the
expression patterns).

The exact same syntax works for Configuration and Device provisioning.

### <a name="values"/> Variable values

Attribute expressions can contain values taken from the value of other attributes. Those values have, by default, the
String type. For most arithmetic operations ('*', '/', etc...) if a variable is involved, its value will be cast to
Number, regardless of the original type. For, example, if a variable @humidity has the value `'50'` (a String value),
the following expression:
```
${@humidity * 10}
```
will give `500` as the result (i.e.: the value `'50'` is cast to number, to get `50`, that is then multiplied by 10). If
this cast fails (because the value of the variable is not a number, e.g.: `'Fifty'`), the overall result will be `NaN`.

### <a name="#execution"/> Expression execution

Whenever a new measurement arrives to the IoTAgent for a device with declared expressions, all of the expressions for
the device will be checked for execution: for all the defined active attributes containing expressions, the IoTAgent
will check which ones contain expressions whose variables are present in the received measurement. For all of those
whose variables are covered, their expressions will be executed with the received values, and their values updated in
the Context Broker.

E.g.: if a device with the following provisioning information is provisioned in the IoTAgent:
```
{
   "name":"location",
   "type":"geo:point",
   "expression": "${latitude}, ${longitude}"
},
{
   "name":"fillingLevel",
   "type":"Number",
   "expression": "${@fillingLevel / 100}",
   "cast": "Number"
},
```
and a measurement with the following values arrive to the IoTAgent:
```
latitude: 1.9
level: 85.3
```
The only expression rule that will be executed will be that of the 'fillingLevel' attribute. It will produce the value
'0.853' that will be sent to the Context Broker.

## <a name="description"/> Language description

### <a name="types"/> Types

Expressions can have two return types: String or Number. This return type must be configured for each attribute that
is going to be converted. Default value type is String.

Whenever a expression is executed without error, its result will be cast to the configured type. If the conversion fails
(e.g.: if the expression is null or a String and is casted to Number), the measurement update will fail, and an error
will be reported to the device.

### <a name="values"/> Values

#### Variables

All the information reported in the measurement received by the IoT Agent is available for the expression to use. For
every attribute coming from the South Bound, a variable with the syntax '@<object_id>' will be created for its use in
the expression language.

#### Constants

The expression language allows for two kinds of constants:
- Numbers (integer or real)
- Strings (marked with double quotes)

Current allowed characters are:
- All the alphanumerical characters
- Whitespaces

### <a name="operations"/> Allowed operations

The following operations are currently available, divided by attribute type

#### Number operations

- multiplication ('*')
- division ('/')
- addition ('+')
- substraction ('-' binary)
- negation ('-' unary)
- power ('^')

#### String operations

- concatenation ('#'): returns the concatenation of the two values separated by `#`.
- substring location (`indexOf(<variable>, <substring>)`): returns the index where the first occurrence of the substring
`<substring>` can be found in the string value of `<variable>`.
- substring (`substr(<variable>, <start> <end>)`): returns a substring of the string variable passed as a parameter,
starting at posisiton `start` and ending at position `<end>`.
- whitespace removal (`trim(<string>)`): removes all the spaces surrounding the string passed as a parameter.

#### Other available operators

Parenthesis can be used to define precedence in the operations. Whitespaces between tokens are generally ignored.

## <a name="examples"/> Examples of expressions

The following table shows expressions and their expected outcomes for a measure with two attributes: "@value" with value
6 and "@name" with value "DevId629".

| Expression                  | Expected outcome      |
|:--------------------------- |:--------------------- |
| '5 * @value'                | 30                    |
| '(6 + @value) * 3'          | 36                    |
| '@value / 12 + 1'           | 1.5                   |
| '(5 + 2) * (@value + 7)'    | 91                    |
| '@value * 5.2'              | 31.2                  |
| '"Pruebas " + "De Strings"' | 'Pruebas De Strings'  |
| '@name value is @value'     | 'DevId629 value is 6' |

## <a name="ngsiv2"/> NGSIv2 support

As it is explained in previous sections, expressions can have two return types: String or Number, being the former one the default. Whenever an expression is executed without error, its result will be cast to the configured type. 

On one hand, in NGSIv1 since all attributes' values are of type String, in the expression parser the expression type is set always to String and the transformation of the information coming from the SouthBound is done using replace instruction. Therefore, values sent to the CB will always be Strings. This can be seen in previous examples.

On the other hand, NGSIv2 fully supports all the types described in the JSON
specification (string, number, boolean, object, array and null). Therefore, the result of an expression must be cast to the appropriate type (the type used to define the attribute) in order to avoid inconsistencies between the type field for an attribute and the type of the value that is being sent.

Currently, the expression parser does not support JSON Arrays and JSON document. A new issue has been created to address this aspect https://github.com/telefonicaid/iotagent-node-lib/issues/568. For the rest of types the workflow will be the following:

1. Variables will be cast to Number or String depending on the expression type.
2. The expression will be applied
3. The output type will be cast again to the original attribute type.
  * If attribute type is "Integer" then the value is casted to integer (JSON number)
  * If attribute type is "Float" then the value is casted to float (JSON number)
  * If attribute type is "Boolean" then the value is cast to boolean (JSON boolean). In order to do this conversion, only `true` or `1` are cast to true.
  * If attribute type is "None" then the value is cast to null (JSON null)

E.g.: if a device with the following provisioning information is provisioned in the IoTAgent:
```
{
   "name":"status",
   "type":"Boolean",
   "expression": '${@status *  20}'
}
```
and a measurement with the following values arrive to the IoTAgent:
```
status: true
```

1. The expression `*` is a multiplication, so the expression type makes `status` to be casted to Number. The cast of `true` to number is 1 (everything with a "value" is true in JavasScript).
2. Expression is evaluated, resulting in 20
3. 20 is cast to `20` since Expression Plugin always use String as Expression type.
4. The attribute type is `Boolean` so the result is casted to Boolean before sending it to CB. The cast of `20` to boolean is false (only `true` or `1` are cast to true).

More examples of this workflow are presented below for the different types of attributes supported in NGSIv2 and the two possible types of expressions: Integer (arithmetic operations) or Strings.

* pressure: 52 -> ${@pressure * 20} -> ${ 52 * 20 } -> $ { 1040 } -> $ { "1040"} -> 1040
* pressure: 52 -> ${trim(@pressure)} -> ${trim("52")} -> $ { "52" } -> $ { "52"} -> 52
* consumption: 0.44 -> ${@consumption * 20} -> ${ 0.44 * 20 } -> $ { 8.8 } -> $ { "8.8"} -> 8.8
* consumption: 0.44 -> ${trim(@consumption)} -> ${trim("0.44")} -> $ { "0.44" } -> $ { "0.44"} -> 0.44
* active: null -> ${@active * 20} -> ${ 0 * 20 } -> $ { 0 } -> $ { "0"} -> null
* active: null -> ${trim(@active)} -> ${trim("null")} -> $ { "null" } -> $ { "null"} -> null
* update: true -> ${@update * 20} -> ${ 1 * 20 } -> $ { 20 } -> $ { "20"} -> False
* update: false -> ${@update * 20} -> ${ 0 * 20 } -> $ { 0 } -> $ { "0"} -> False
* update: true -> ${trim(@updated)} -> ${trim("true")} -> $ { "true" } -> $ { "true"} -> True
