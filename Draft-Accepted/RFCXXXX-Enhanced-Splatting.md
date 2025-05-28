---
RFC: RFCXXXX
Author: Jordan Borean
Status: Draft
SupercededBy: N/A
Version: 1.0
Area: Language/Engine
Comments Due: When hell freezes over
Plan to implement: Yes
---

# PowerShell Splatting Enhancements

Splatting is a feature that allows users to build up parameters and values through a dictionary or array object to then bind directly as parameter or positional arguments respectively when invoking a command. The following example shows an equivalent way of specifying the `-Path` and `-Force` parameters with `Get-ChildItem` directly and with splatting as a hashtable.

```powershell
Get-ChildItem -Path C:\Windows -Force

$mySplat = @{
    Path = 'C:\Windows'
    Force = $true
}
Get-ChildItem @mySplat
```

The splat syntax is limited today to just a simple variable name to splat. This RFC is a proposal to extend support for splatting to include:

+ inline hashtable and array splatting
+ splatting a member value
+ splatting the result of a single expression `(...)`

What is not covered, but is still considered, in this RFC is the ability to:

+ splat .NET method arguments
+ splat sub expressions `$(...)` results
+ inline filtering of hashtable values

## Motivation

    As a PowerShell scripter,
    I can use enhanced splatting,
    so that I can streamline my code and not have to define a variable just to use splatting.

## User Experience

Some use cases behind splatting would be to:

+ split up long lines without relying on backticks

```powershell
# +140 chars long
Start-Process -FilePath C:\some\really\long\path.ps1 -ArgumentList 'some argument that is long' -WorkingDirectory C:\Windows -PassThru -Wait

# vs splatting can reduce the horizontal length
$mySplat = @{
    FilePath = 'C:\some\really\long\path.ps1'
    ArgumentList = 'some argument that is long'
    WorkingDirectory = 'C:\Windows'
    PassThru = $true
    Wait = $true
}
Start-Process @mySplat
```

+ passing through parameters in proxy functions

```powershell
Function Get-ProxyValue {
    [CmdletBinding()]
    param (
        ...
    )

    # Manually passing them through
    Get-Value -Param1 $Param1 -Param2 $Param2 -SomeSwitch:$SomeSwitch ...

    # Vs using a splat to do it automatically
    Get-Value @PSBoundParameters
}
```

+ making it easier to document, or add/remove parameters when they are per line vs embedding a string

```powershell
# If I wanted to remove -Param2 I would need to move the cursor to that position and delete the selection
Get-Value -Param1 $Param2 -Param2 $Param2 -SomeSwitch

# With a splat I can just remove the line which most editors have a shortcut or can Shift+End -> Delete
# I can also easily add a comment if need be
$mySplat = @{
    Param1 = $Param1
    # Some extra info providing more context to Param2
    Param2 = $Param2
    SomeSwitch = $true
}
Get-Value @mySplat
```

+ adding/remove values conditionally

```powershell
# Adds -File $somePath if the path exists
$mySplat = @{}
if (Test-Path $somePath) {
    $mySplat.File = $somePath
}
Test-Function @mySplat
```

The two main problems with splatting today is that it only works with a simple variable name and it is not possible to conditionally splat values without having to redefine a new `IDictionary`. For example:

```powershell
$myObjProperty = $myObj.Property
Test-Function @myObjProperty

$splatValue Get-SplatValue
Test-Function @splatValue

# This looks like it might work but it will pass the value positionally
# and not splat them
Test-Function @(Get-SplatValue)

Function Get-ProxyValue {
    [CmdletBinding()]
    param (
        [Parameter()]
        [string]
        $ProxyParameter,

        [Parameter()]
        [string]
        $InnerValue
    )

    # Makes a top level copy of the parameters and removes
    # ProxyParameter if present
    $innerSplat = [hashtable]$PSBoundParameters
    $innerSplat.Remove('ProxyParameter')

    if ($ProxyParameter) {
        # Do something in the proxy
    }

    Get-Value @innerSplat
}
```

For all these examples, we require a variable that is an `IDictionary` or `IList` to splat either through named parameters or positionally respectively. By expanding splatting functionality we could avoid such scenarios by providing ways to do more complex splatting without the intermediate variable.

+ we need to name the variable something/add clutter to the code
+ we actually need a variable to be stored in the session state, takes up a slot in the function's tuple store
+ tab completion/prediction is more complicated as it needs to scan additional lines to try and find the command it is used for

While not part of this RFC a future addition might include support for splatting named arguments to .NET method invocations. As an example, here is a C# function with a single mandatory argument and two optional arguments.

```csharp
public void ExampleMethod(
    string label
    int count = 5,
    string comment = "default comment");
```

If we wanted to pre-defined the arguments in a hashtable or inline we could do something like

```powershell
# How would we splat a hashtable as a var
$arguments = @{
    comment = "Custom comment"
}
$class.ExampleMethod("My label", <# $arguments #>)

# Also how would we splat a hashtable literal
$class.ExampleMethod("My label", <# @{comment = "Custom comment"} #>)
```

Please note there is an [existing PR](https://github.com/PowerShell/PowerShell/pull/21487) to add support for optional argument literals like the below but this is **NOT** considered splatting and a different problem to this proposal:

```powershell
$class.ExampleMethod("My label", comment: "Custom comment")
```

The end result of this RFC should take the .NET argument splatting into consideration to ensure this could be applied in a consistent fashion there.

## Options

These are the 5 options considered in this RFC. Each example is how this would look like when specified as an argument.

|Option|Variable|Variable Member|Hashtable Literal|Array Literal|Expression|
|-|-|-|-|-|-|
|1. `@@<expr>`|`@var`|`@var.Member`|`@@{Key='Value'}`|`@@('Positional])`|`@&{...}`|
|2. `..<expr>`|`..$var`|`..$var.Member`|`..@{Key='Value'}`|`..@('Positional)`|`..(...)`|
|3. `@[<expr>]`|`@[$var]`|`@[$var.Member]`|`@[@{Key='Value'}]`|`@[@('Positional')]`|`@[...]`|
|4. `-@` param|`-@ $var`|`-@ $var.Member`|`-@ @{Key='Value'}`|`-@ @('Positional')`|`-@ (...)`|
|5. `-splat` operator|`(-splat $var)`|`(-splat $var.Member)`|`(-splat @{Key='Value'})`|`(-splat @('Positional'))`|`(-splat (...))`|

_Note: Regardless of the option chosen, `@var` will always continue to work for backwards compatibility._

This is how I (yes it's subjective to me) would rank each option from 1-5 with the following categories:

+ Intuitiveness - How intuitive is the option
+ Usability - How much typing is needed
+ Consistency - How consistent is the splat options between all the scenarios
+ Impact - The risk (5 being lowest, 1 being highest) of this option breaking existing scripts

|Option|Intuitiveness|Usability|Consistency|Impact|Total|
|-|-|-|-|-|-|
|1. `@@<expr>`|5|4|1|3|13|
|2. `..<expr>`|1|5|3|1|10|
|3. `@[<expr>]`|3|2|2|5|12|
|4. `-@` param|4|3|5|2|14|
|5. `-splat` operator|2|1|4|4|11|

My recommendation would be:

+ Use `-@` param (or `-PSSplat`) if consistency is desired and we can accept the risk of breaking existing code
  + Also recommend supporting multiple `-@` entries to splat multiple items
+ Use `@@<expr>` if language consistency is not desired and we want to continue the feel of `@splat`
  + Also risk the breaking change to support `@$(...)` for an expression rather than `@&{...}`
+ Use `@[<expr>]` if consistency and we really don't want to risk any breakages
  + Ideally supporting `@[Key='Value']` without requiring `@{}` if we could pull it off in parsing

Each of the option examples have the following defined:

```powershell
Function Get-SplatValue {
    @{
        Path = 'C:\path'
    }
}

$ht = @{
    Path = 'C:\path'
}
$array = @('C:\path')

$obj = [PSCustomObject]@{
    Property = $ht
}
$obj | Add-Member -Name Method -MemberType ScriptMethod -Value { $this.Property }

$item = @{
    Key = $ht
}
```

## Option 1 - @@<expr>

This is a continuation on the existing `@var` splatting syntax. It proposes to:

+ expand the single `@` syntax to support members like `@obj.Property`, `@it['Key']`, or `@obj.Method()`
+ use `@@{}` and `@@()` for inline hashtable or array literals
+ use `@&{}` for inline expressions

```powershell
# Simple var
Test-Function @ht
Test-Function @array

# Member value
Test-Function @obj.Property
Test-Function @obj.Method()
Test-Function @item['Key']
Test-Function @item["Key"]
Test-Function @item[$keyInVar]

# Inline hashtable
Test-Function @@{
    Path  = 'C:\path'
    Force = $true
}

# Inline array
Test-Function @@(
    'Positional 1'
    'Positional 2'
)

# Inline expression
Test-Function @&{Get-SplatValue}
```

The original though for an inline expression was to support `@$(...)` but unfortunately this is currently valid syntax. It is treated like the following so using this syntax would mean a breaking change.

```powershell
$var1 = $$
$var2 = $(...)
Test-Function @var1 $var2
```

The `@&{}` syntax was chosen as it is similar to the call operator for a scriptblock. There is an argument for making a breaking change and treating `@$(...)` as a splatting expression value as:

+ `$$` is a string
+ splatting a string means splatting the chars positionally which is not useful
+ `$$` only makes sense when used in a REPL and not a script with the latter being where this splatting example would be more useful

In the future, support for splatting a hashtable in a method invocation would just follow the same rules as parameter splatting:

```powershell
$obj.Method(@var)
$obj.Method(@@{argName = $foo})
```

The advantages of this option are:

+ member values follow the existing `@` splatting syntax and work as expected
+ inline hashtable and array literals follow the convention of just prepending another `@`
+ when asking for feedback, most people expect the inline ht literals to work like this
+ the new syntaxes are currently illegal so no chance of breaking existing scripts

The disadvantages are:

+ there are 3 different ways to splat thing, `@var`, `@@{}/@@()`, and `@&{...}`
+ the inline expression example seems to be out of place but supporting this scenario now or in the future is ideal

I think this option would be the ideal choice if the inline expression syntax could follow the rule of "prepend `@` before the literal value", e.g. `@$(...)`. If a breaking change here would be accepting this would be the number one choice but if a breaking change is not acceptable then the inconsistency in the splatting rules for the different scenarios makes this less ideal for me.

## Option 2 - Spread Operator ..<expr>

The chosen method for enhanced splatting would be to implement something similar to the new [C# spread element](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/collection-expressions#spread-element). The syntax for this to specify the spread literal `..` then the expression of the result to "spread/splat".

```powershell
# Simple var
Test-Function ..$ht
Test-Function ..$array

# Member value
Test-Function ..$obj.Property
Test-Function ..$obj.Method()
Test-Function ..$item['Key']
Test-Function ..$item["Key"]
Test-Function ..$item[$keyInVar]

# Inline hashtable
Test-Function ..@{
    Path  = 'C:\path'
    Force = $true
}

# Inline array
Test-Function ..@(
    'Positional 1'
    'Positional 2'
)

# Inline expression
Test-Function ..(Get-SplatValue)
```

It could optional also be used to union multiple hashtables together like a C# collection expressions.

```powershell
$ht1 = @{ Foo = 'value' }
$ht2 = @{ Bar = 'value' }

@{
    Other = 'value'
    ..$ht1
    ..$ht2
}
```

In the future, support for splatting a hashtable in a method invocation would just follow the same rules as parameter splatting:

```powershell
$obj.Method(..$var)
$obj.Method(..@{argName = $foo})
```

The advantages of this option are:

+ the splatting syntax for all options are consistent
+ aligns with C# syntax so has some precedence in a language
+ doesn't require any parsing changes but will most likely require some changes to the compiler

The disadvantages are:

+ we now have two ways to splat simple variables `..$var` and `@var`
+ users may be unfamiliar with the `..<expr>` syntax and expect some form of `@<expr>`
+ the syntax is not illegal and could cause conflicts with existing code

The last point is important and worth going into more detail.

```powershell
Function Test-Function {
    ConvertTo-Clixml $args
}

$var = @{Foo = 'bar'}
Test-Function ..$var
# <LST>
#     <S>..System.Collections.Hashtable</S>
# </LST>

Test-Function ..@{Foo = 'bar'}
# <LST>
#     <S>..@</S>
#     <SBK>Foo = 'bar'</SBK>
# </LST>

Test-Function ..@('Position 1', 'Position 2')
# <LST>
#     <S>..@</S>
#     <Obj RefId="1">
#     <TNRef RefId="0" />
#     <LST>
#         <S>Position 1</S>
#         <S>Position 2</S>
#     </LST>
#     </Obj>
# </LST>

$var = @{ Prop = @{ 'Foo' = 'bar' } }
Test-Function ..$var.Prop
# <LST>
#     <S>..System.Collections.Hashtable.Prop</S>
# </LST>

Function Get-SplatValue {
    @{ Foo = 'bar' }
}
Test-Function ..(Get-SplatValue)
# <LST>
#     <S>..</S>
#     <Obj RefId="1">
#     <TN RefId="1">
#         <T>System.Collections.Hashtable</T>
#         <T>System.Object</T>
#     </TN>
#     <DCT>
#         <En>
#         <S N="Key">Foo</S>
#         <S N="Value">bar</S>
#         </En>
#     </DCT>
#     </Obj>
# </LST>
```

The only time I could see someone using a positional parameter value that starts with `..` would be for a relative path. The parser could treat this new syntax as the spread ast if the third character is `$`, `@`, or `(` meaning `../dir/file.txt` would still be treated as a single string value.

Ultimately I don't think this should be chosen, while C# does use this feature it is somewhat unknown in PowerShell and the fact that it could be used today makes it more likely to break existing code. While that breakage could be mitigated by checking the next char after `..` is `$`, `@`, or `(` I cannot be sure that people do not do this anyway. The main pro this has going for it is a consistent syntax between all the options.

## Option 3 - @[<expr>]

This proposal aims to add in a new block sigil `@[<expr>]` to represent that the expression value should be splatted. It aims to provide a consistent way of splatting a value while trying to somewhat align with the `@{}`, `@()` convention in PowerShell.

```powershell
# Simple var
Test-Function @[$ht]
Test-Function @[$array]

# Member value
Test-Function @[$obj.Property]
Test-Function @[$obj.Method()]
Test-Function @[$item['Key']]
Test-Function @[$item["Key"]]
Test-Function @[$item[$keyInVar]]

# Inline hashtable
Test-Function @[@{
    Path  = 'C:\path'
    Force = $true
}]

# Inline array
Test-Function @[@(
    'Positional 1'
    'Positional 2'
)]
# Could also be an array expression if there is more than 1 value
Test-Function @["Positional 1", "Positional 2"]

# Inline expression
Test-Function @[Get-SplatValue]
```

A slight alternative would be treat `@[]` by itself as a hashtable literal and require `@[(...)]` for an expression so that the more common example is easier to write and view:

```powershell
Test-Function @[
    Path  = 'C:\path'
    Force = $true
]

Test-Function @[(Get-SplatValue)]
```

But that does not seem too clean and might raise some tricky parsing logic for simple vars or var members without it being inside `(...)`. If there is a way to allow the `@[<expr>]` syntax but also allow an inline hashtable to be defined without the `@{}` around it then this option is a strong contender.

For optional future support for method invocation it would just be treating like a parameter:

```powershell
$obj.Method(@[$var])
$obj.Method(@[@{argName = $foo}])
# Or possibly
# $obj.Method(@[argName = $foo])
```

The advantages of this option are:

+ the splatting syntax for all options are consistent
+ the `@[]` is not valid syntax today so it will not break any existing scripts
+ the expression syntax is quite concise compared to other options

The disadvantages are:

+ we now have two ways to splat simple variables `..$var` and `@var`
+ a new thing users have to learn
+ an inline hashtable is probably the most common scenario here but still requires the `@[@{}]` wrapper around the value which could be considered ugly

I like this option a lot as it ticks boxes like it being consistent for all scenarios, won't break existing scripts, and is simple enough but the fact that an inline hashtable literal most likely needs to still have the `@{}` inside the `@[]` turns me off this choice. The other hiccup is that people just expect `@@{}` to be the splatting choice and wouldn't naturally think `@[]` could be used for splatting.

If it actually isn't hard to support a hashtable literal without `@{}` alongside any other expression I would probably recommend this option. The method could potentially be called a parameter block or something other than splatting to keep `@var` and `@[...]` from being confused even if the overall feature is about the same.

## Option 4 - -@ Parameter

This option moves away from using a special sigil to using a parameter that identifies the value to splat. The proposal is to add the "parameter" `-@` to any function where the value specified will be splatted by the binder.

```powershell
# Simple var
Test-Function -@ $ht
Test-Function -@ $array

# Member value
Test-Function -@ $obj.Property
Test-Function -@ $obj.Method()
Test-Function -@ $item['Key']
Test-Function -@ $item["Key"]
Test-Function -@ $item[$keyInVar]

# Inline hashtable
Test-Function -@ @{
    Path  = 'C:\path'
    Force = $true
}

# Inline array
Test-Function -@ @(
    'Positional 1'
    'Positional 2'
)

# Inline expression
Test-Function -@ (Get-SplatValue)
```

The parameter `-@` is chosen over other names like `-Splat`, `-PSSplat`, `-SplatObj`, etc as:

+ it continues the theme of `@` being used for splatting
+ it is easy to type `-@`
+ `-?` is already used as a special help method so there is some precedence for this format
+ it is less likely to have already been used as an existing parameter as it cannot be used without splatting right now

```powershell
Function Test-Function {
    [CmdletBinding()]
    param ([Parameter()]${@})

    ${@}
}

# Won't work, '-@' is bound positionally to ${@} and abc is left over
Test-Function -@ 'abc'

# Works and bound to `-@` positionally
Test-Function 'abc'

# A splat is the only way to actually bind to `-@` as a parameter
$splat = @{
    '@' = 'abc'
}
Test-Function @splat
```

There are three other scenarios that need to be handled if this is the chosen format:

+ what happens if you try and splat `?` in the splat, e.g. `Test-Function -@ @{'@' = ...}`
  + my recommendation is to make this illegal and have it error
+ currently restricted to only be used once
  + either live with this restriction or support `-@` being specified multiple times
  + `Test-Function -@ $var -@ @{ Foo = 'Bar' } -@ (Get-SplatValue)`
+ should this apply to advanced functions only
  + non advanced functions might rely on `-@` being a positional argument
  + same as native binaries
  + restricting this syntax to advanced functions reduces the risk of this impacting existing scripts but lessens the versatility of this feature
  + my gut feeling is that there is very little impact to warrant restricting it and could be initially gated behind an experimental feature

A final side note is that this implementation should not be exposed through a commands metadata and act like `-?` does. If it was exposed like other common parameters, tools like `platyPS` will try and document the new parameter like what happened with `-ProgressAction`.

The advantages of this option are:

+ the splatting syntax for all options are consistent
+ no parsing or Ast changes would be required
+ most options are quite concise and work just like normal arguments
+ there is already a precedence with `-ProgressAction` for introducing new parameters

The disadvantages are:

+ runs the risk of impacting existing functions if they define `${@}` as a parameter
+ there's an open question as to how non-advanced functions/cmdlets would use this feature
+ unsure if this would be tricky to implement in the parameter binder
+ it is not clear if it could be extended to support splatting method arguments `$obj.Method(-? @{...})`

I think this is a solid suggestion if the ability to use `-@` multiple times is supported. It follows the standard of `@` being used to signal splatting a value, it has a consistent experience with all the various scenarios, and won't require any Ast or parsing changes to accommodate making the implementation simpler. The biggest questions is whether the risk for impacting existing scripts is worth it and how this should apply to non advanced functions/cmdlets.

## Optional 5 - -splat operator

The final option is to add a unary operator called `-splat` that could be used to mark an object as being splattable or not. The implementation could be as simple as adding a NoteProperty to the object so that when the binder goes to process that value it can check that note property to decide whether it's a positional vs splatted value.

```powershell
# Simple var
Test-Function (-splat $ht)
Test-Function (-splat $array)

# Member value
Test-Function (-splat $obj.Property)
Test-Function (-splat $obj.Method())
Test-Function (-splat $item['Key'])
Test-Function (-splat $item["Key"])
Test-Function (-splat $item[$keyInVar])

# Inline hashtable
Test-Function (-splat @{
    Path  = 'C:\path'
    Force = $true
})

# Inline array
Test-Function (-splat @(
    'Positional 1'
    'Positional 2'
))

# Inline expression
Test-Function (-splat (Get-SplatValue))
```

The advantages of this option are:

+ the splatting syntax for all options are consistent
+ no parsing or Ast changes would be required
+ no risk of breaking existing code as people cannot define their own parameters
+ it is theoretically possible to mark an object as splattable outside of the function line at runtime
+ it could be extended to support the ability to mark values to remove from the splat or optional to splat `(-splat $ht, @('Param1', 'Param2'))

The disadvantages are:

+ it is quite verbose, not very nice from a scripting perspective

This is an interesting option that is quite flexible and doesn't risk breaking any existing scripts but IMO falls short from a usability perspective.
