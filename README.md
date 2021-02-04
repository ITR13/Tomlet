# Tomlet
## A WIP TOML library for .NET

Tomlet is a WIP library for the [TOML](https://toml.io/) configuration file format. It's targeting [TOML v1.0.0](https://toml.io/en/v1.0.0).

Currently supported features are as follows:

- [x] Primitive key-value pair reading
- [x] Table Reading
- [x] Inline Table Reading
- [x] Array Reading
- [x] Table-Array Reading
- [ ] Primitive key-value pair writing
- [ ] Table Writing
- [ ] Inline Table Writing
- [ ] Array Writing
- [ ] Table-Array Writing
- [x] Full Unit Tests for everything supported here.

## Usage

### Parse a TOML File

`TomlParser.ParseFile` is a utility method to parse an on-disk toml file.

```c#
TomlDocument document = TomlParser.ParseFile(@"C:\MyFile.toml");
```

### Parse Arbitrary TOML input
Useful for parsing e.g. the request of a web request.
```c#
TomlParser parser = new TomlParser();
TomlDocument document = parser.Parse(myTomlString);
```

### Retrieve data from a TomlDocument

```c#
TomlDocument document; // See above for how to obtain one.
int someInt = document.GetInteger("myInt");
string someString = document.GetString("myString");

// TomlArray implements IEnumerable<TomlValue>, so you can iterate over it or use LINQ.
foreach(var value in document.GetArray("myArray")) {
    Console.WriteLine(value.StringValue);
}

//It also has an index operator, so you can do this
Console.WriteLine(document.GetArray("myArray")[0]);

List<string> strings = document.GetArray("stringArray").Select(v => (TomlString) v).ToList();

//Retrieving sub-tables. TomlDocument is just a special TomlTable, so you can 
//call GetSubTable on the resulting TomlTable too.
string subTableString = document.GetSubTable("myTable").GetString("aString");

//Date-Time objects. There's no API for these (yet)
var dateValue = document.GetValue("myDateTime");
if(dateValue is TomlOffsetDateTime tomlODT) {
    DateTimeOffset date = tomlODT.Value;
    Console.WriteLine(date); //27/05/1979 00:32:00 -07:00
} else if(dateValue is TomlLocalDateTime tomlLDT) {
    DateTime date = tomlLDT.Value;
    Console.WriteLine(date.ToUniversalTime()); //27/05/1979 07:32:00
} else if(dateValue is TomlLocalDate tomlLD) {
    DateTime date = tomlLD.Value;
    Console.WriteLine(date.ToUniversalTime()); //27/05/1979 00:00:00
} else if(dateValue is TomlLocalTime tomlLT) {
    TimeSpan time = tomlLT.Value;
    Console.WriteLine(time); //07:32:00.9999990
} 
```

### Handling Errors

Both `ParseFile` and `Parse` can throw multiple exceptions (currently around 40), but they are all an instance of `TomlException`.

Most of these are specific to various errors as specified in the specification, but there is also `TomlInternalException` which is thrown if there is an exception in Tomlet itself. 
If it is thrown, it will have the unhandled exception as a cause, and the exception message will state the line number which caused the exception. I'd appreciate it
if you could report these to me.

The full list of exceptions follows, in alphabetical order:

| Exception Class | Probable Cause | Resolution    |
| :-------------- | :------------: | ------------: |
| `InvalidTomlDateTimeException` | A generic date/time error is present in the file. The string which Tomlet tried to parse as a date is provided in the message. | Fix the string to be a valid RFC 3339 date/time. |
| `InvalidTomlEscapeException`   | A string escape such as `\n` was found in a double-quoted string, but is reserved according to the TOML specification. | Remove the escape sequence, escape the backslash, or use \uXXXX to specify the unicode character. |
| `InvalidTomlInlineTableException` | A key-value pair was found in an inline table which either doesn't have a key, or is missing an equals sign (=). The exception cause will have more information | Fix the syntax error in the inline table. |
| `InvalidTomlKeyException` | Never thrown by the parser. Only thrown during calls to `TomlTable.GetXXXX` when the key provided contains both single and double quotes | Pass a valid TOML key into the `GetXXXX` function |
| `InvalidTomlNumberException` | A number (float or integer) literal was found in the TOML document, but isn't valid. The exception message contains the line number. | Quote the value in the file and interpret it as a string, or remove the value |
| `MissingIntermediateInTomlTableArraySpecException`| A Table-Array was found in the file which is a child element of an as-yet undefined value. The exception message details this value. | Fix the file to be in a valid order according to the TOML specification|
| `NewLineInTomlInlineTableException` | A new-line character (`\n`) was found between the key-value pairs of an inline table. The exception message provides the line number. | Remove the new-line character |
| `NoTomlKeyException` | A key-value pair was found in the TOML document which doesn't have a key before the equals sign. The exception message provides the line number. | If you really want an empty-string as a key, quote it (`""`) |
| `TimeOffsetOnTomlDateOrTimeException` | A time offset (such as `-07:00`, or `Z`) was found on a Local Date or Local Time object in the TOML document. This is not valid. | Remove the time offset, or provide a full date-time string. |
| `TomlArraySyntaxException` | The parser was expecting a comma (`,`) or closing bracket (`]`) but found something else, while parsing an inline array. The exception message provides the line number and offending character. | Fix the array syntax.|
| `TomlContainsDottedKeyNonTableException`| A call was made to `TomlTable.ContainsKey` or `TomlTable.GetXXXX` with a dotted key, which implied that a key should be a table, when it in fact wasn't. The exception message provides the line number and offending key. | Fix the call to `ContainsKey` or `GetXXXX`|
| `TomlDateTimeMissingSeparatorException` | RFC 3339 date/time literals must contain a `T`, `t`, or space between the date and time strings. This literal did not have such a separator. The exception message provides the line number. | Insert an appropriate separator between the date and time |
| `TomlDateTimeUnnecessarySeparatorException`| A date/time separator (`T`, `t`, or a space) was found in a Local Date or Local Time string, where it has nothing to separate. This is not allowed. | Remove the separator.|
| `TomlDottedKeyException` | An attempt was made to programmatically insert a value into a `TomlTable` using a dotted key, which implied that an intermediary key was a table, when it is in fact not. The exception message provides the intermediary key. | Fix the code which inserted the value into the TomlTable |
| `TomlDottedKeyParserException` | Similar to the above, except the dotted key was in the TOML document being parsed. The exception message provides the line number and intermediary key. | Check the TOML document to ensure you are using the correct key.|
| `TomlEOFException` | The end of a file was reached while attempting to parse a key/value pair. | Restore the truncated data from the end of the file |
| `TomlInlineTableSeparatorException` | The parser was expecting a comma (`,`) or closing brace (`}`) after a key-value pair, while parsing an inline table, but found something else. The exception message provides the line number and offending character. | Fix the syntax error in the inline table.|
| `TomlInternalException` | Detailed in the paragraph preceding this table | Report the issue on the GitHub repository.|
| `TomlInvalidValueException` | While parsing a key-value pair, the parser read the first character of a value, which does not appear to indicate the start of any valid value type. The exception message provides the line number and offending character. | Correct the value. Is it supposed to be a (quoted) string literal?|
| `TomlKeyRedefinitionException` | The Parser found that a value is present twice within the TOML document. | Remove the duplicate assignment. |
| `TomlMissingEqualsException` | The Parser found a value before it found an equals sign, after a key. The exception message provides the line number. | Insert the equals sign, or, if the key is supposed to contain whitespace, quote the key.| 
| `TomlMissingNewlineException` | The Parser found the beginning of another key-value pair on the same line as one it has previously parsed. All key-value pairs must be on their own line. The exception message provides the line number and first character of the second key-value pair. | Insert a newline character between the key-value pairs.|
| `TomlNonTableArrayUsedAsTableArrayException`| The Parser found a table-array declaration (`[[TableArrayName]]`) which is re-using the name of a previous array. The exception message provides the line number and array name. | Remove the conflicting array declaration, or rename the table-array. |
| `TomlNoSuchValueException` | Thrown when a call to `TomlTable.GetXXXX` is called with a key which wasn't present in the TOML document | Check the call to `GetXXXX`, or check if the key is present first using `ContainsKey` |
| `TomlStringException` | The parser found a string which starts with two of the same quote (single or double) and then has a non-quote, non-whitespace, non-comment character. The exception message provides the line number. | Correct the string literal.|
| `TomlTableArrayAlreadyExistsAsNonArrayException`| A Table-Array declaration was found which overwrites a non-array value. The exception message provides the line number and array name. | Remove the conflicting value, or rename the table-array |
| `TomlTableArrayIntermediateNonTableException`| A Table-Array declaration was found which references an intermediate value which is an array, but the values contained within that array aren't tables. The exception message provides the line number and offending array name. | Fix the table-array name to refer to the correct intermediate array.|
| `TomlTableLockedException` | The Parser found an attempt to insert or modify the value of a key contained within a table which was declared inline. Inline tables are immutable. The exception message provides the line number and key being updated or inserted.| Remove the assignment, or declare the table non-inline.|
| `TomlTableRedefinitionException` | The Parser found an attempt to re-declare an already declared table name. The exception message provides the line number and conflicting table name. | Remove or rename the duplicated definition.|
| `TomlTypeMismatchException` | Thrown when a call is made to `GetXXXX` but the type of `XXXX` does not match the type of the literal read from the TOML document. | Check the expected type in the TOML document|
| `TripleQuoteInTomlMultilineLiteralException` | A triple single quote (`'''`) was found within a TOML multiline literal (triple-single-quoted string). The exception message provides the line number. | Remove the triple-quote or use a double-quoted string with appropriate escape sequences. You cannot escape within a triple-quoted string.|
| `TripleQuoteInTomlMultilineSimpleStringException`| A triple double quote (`"""`) was found within a TOML multiline simple string (triple-double-quoted string). The exception message provides the line number. | Escape one of the double quotes.|
| `UnterminatedTomlArrayException` | The Parser finished parsing an array and didn't find a closing bracket (`]`). The exception message provides the line number. | Ensure the array is correctly terminated.|
| `UnterminatedTomlInlineObjectException` | The Parser finished parsing an inline object and didn't find a closing brace (`}`). The exception message provides the line number. | Ensure the inline object is correctly terminated.|
| `UnterminatedTomlKeyException` | The Parser found an unterminated quoted key in the TOML document. The exception message provides the line number. | Terminate the quoted key.|
| `UnterminatedTomlStringException` | The Parser finished parsing a string literal and didn't find the appropriate closing quote(s). The exception message provides the line number. | Ensure the string literal is terminated correctly.|
| `UnterminatedTomlTableArrayException` | The Parser finished parsing a Table-Array declaration and didn't find the closing double-bracket (`]]`). The exception message provides the line number. | Ensure the Table-Array declaration is appropriately terminated.|
| `UnterminatedTomlTableNameException` | The Parser finished parsing a Table declaration and didn't find the closing bracket (`]`). The exception message provides the line number. | Ensure the table declaration is terminated appropriately.|