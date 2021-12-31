---
permalink: /writing-plugins/
title: Writing plugins
---

<div id="leftmenu" class="col-sm-4 col-md-3 hidden-xs">
<ul class="nav nav-list side-navigation" data-spy="affix" data-offset-top="{{ site.affix_offset }}">
    <li><a href="#misc">Don'ts and do's</a></li>
    <li><a href="#example">An example plugin</a></li>
    <li><a href="#render">Renderer plugins</a></li>
    <li><a href="#util">Utils</a></li>
</ul>
</div>
<div class="col-sm-8 col-md-9" markdown="1">

# Writing plugins

<section id="misc" markdown="1">

## Don'ts and do's

Kint and its plugins are in the unique situation of taking a variable by reference and being told not to do anything with it. The absolutely worst thing you can ever do is change an input variable.

If you do this you may fubar a user's data. That's a no-no whether you're writing a plugin for inclusion in Kint or for your own system.

**Never alter input data in a plugin. I will find you.**

### Accessing the parser

All plugins have the parser they're assigned to added to them through the `setParser()` method. If you have new data you want parsed into a value you do so via the parser:

<pre class="prettyprint linenums">
$more_data = get_data_from_somewhere($var);
$base_value = Kint\Zval\Value::blank('Magic somewhere data');
$base_value->depth = $o->depth + 1;
$new_object = $this->parser->parse($more_data, $base_value);
</pre>

### Arrays

Recursion detection in arrays is performed by adding a random-generated unique key to the array as a marker when parsing. If the parser sees this marker when parsing an array it knows it's been here before and is recursing.

As a result, when you try to read directly from an input variable that also happens to be an array, you'll end up with extra data. The way to fix this is to call `$this->parser->getCleanArray()` on the variable, which will shallow copy the array and remove the recursion marker for you.

Note that the first rule on this page still applies to the array's contents.

### Halting the parse

If you want to prevent any other plugins or the parser from messing with your carefully crafted data, you'll want to call `$this->parser->haltParse()`. This will stop any more plugins from running and prevent the base parse too if your plugin is running first.

You can use this with the `Parser::TRIGGER_BEGIN` trigger to prevent parsing entirely based on custom conditions.

**Warning:** The parser will do *absolutely nothing* if you halt a parse at `TRIGGER_BEGIN`. You are responsible for setting up basic information like object hashes and class names, which would usually be handled by the parser before plugins even run. Please take a look at the parser source code to see what needs to be filled in.

</section>
<section id="example" markdown="1">

## An example plugin

Let's imagine we're working on a system that's a big black box. Everything has an ID, and we can get all the data we need by calling a function for that ID.

Wouldn't it be great if we could automatically show the data associated with an ID whenever we come across one?

### A barebones plugin

<pre class="prettyprint linenums">
<?php

use Kint\Zval\Value;
use Kint\Zval\Representation\Representation;
use Kint\Parser\Parser;
use Kint\Parser\Plugin;

class MyPlugin extends Plugin
{
    public function getTypes()
    {
        return array('integer', 'string');
    }

    public function getTriggers()
    {
        return Parser::TRIGGER_SUCCESS;
    }

    public function parse(&amp;$var, Value &amp;$o, $trigger)
    {
        echo 'My parser found: ';
        var_dump($var);
    }
}

Kint::$plugins[] = new MyPlugin();

d(1234);
</pre>

Here we can see the 3 required methods of a Kint_Parser_Plugin.

* `getTypes()` returns the types of data this plugin can operate on. These are types as returned by `gettype()`. Since we're taking IDs they will probably be either strings or integers, so we return an array with both types.
* `getTriggers()` returns a bitmask of the events that will trigger this plugin. These are all constants of the parser class.
    * `TRIGGER_BEGIN` runs before any parsing is done
    * `TRIGGER_SUCCESS` runs after parsing successfully finishes
    * `TRIGGER_DEPTH_LIMIT` and `TRIGGER_RECURSION` run after parsing is halted
    * `TRIGGER_COMPLETE` is `TRIGGER_SUCCESS | TRIGGER_RECURSION | TRIGGER_DEPTH_LIMIT`

    We're going to use `TRIGGER_SUCCESS` for our plugin.
* `parse()` is the workhorse of the plugin. This is what's called when the plugin is expected to alter data. We'll just put a `var_dump` in here to make sure everything's working.

Lastly, we add the plugin to the `Kint::$plugins` array and try it out.

> ![]({{ site.baseurl }}/images/plugin-before.png)

Yay!

### Implementing our plugin's functionality

<pre class="prettyprint linenums:15">
public function parse(&amp;$var, Value &amp;$o, $trigger)
{
    if (!ctype_digit((string) $var)) {
        return;
    }

    global $big_black_box;
    $data = $big_black_box->get_data_from_id($var);
    if (empty($data)) {
        return;
    }

    $base_value = Value::blank('Black box data');
    $base_value->depth = $o->depth;

    if ($o->access_path) {
        $base_value->access_path = '$GLOBALS[\'big_black_box\']->get_data_from_id('.$o->access_path.')';
    }

    $r = new Representation('Black box data');
    $r->contents = $this->parser->parse($data, $base_value);

    $o->addRepresentation($r);
}
</pre>

* Check that what we have is actually an ID. If it's a random string we don't need to waste time trying to get data from it, so we'll just return.
* Get the data we want to add to the dump. If we couldn't find any we'll just return.
* Make our "Base value" - this needs to contain information the parser can't get about the variable like its name, access path, depth, whether it's public or private, etc.
* If we have an access path to the variable we're parsing now, we can continue the access path into the data by wrapping the current one in the code we need to get the data.

    This means if the ID is found at `$array['key']->prop` then `$data['childeren']` will have an accurate access path like:

    `$GLOBALS['big_black_box']->get_data_from_id($array['key']->prop)['childeren']`
* Make a new representation and put the parsed data inside it
* Add the representation to the value

> ![]({{ site.baseurl }}/images/plugin-after.png)

Yay!

---

You can look at the source code for the plugins shipped with Kint by default for more detailed examples on various behaviors.

</section>
<section id="render" markdown="1">

## Renderer plugins

Renderers don't have a unified plugin system, it's implemented by the individual renderers at will.

The one common factor is that parser plugins will attach strings to the `hints` array on values and representations which the renderer can use to decide what to do without having to re-parse the values.

In the case of the rich renderer there are 2 separate plugin pools, where the key is the hint and the value is the plugin class:

* `Kint\Renderer\RichRenderer::$value_plugins`: How to render a value itself. This alters both the way the bar for a value appears, and how the children are rendered. For example, it's a value renderer that adds the color swatch to the bar for a color string.
* `Kint\Renderer\RichRenderer::$tab_plugins`: How to render an individual tab. For example, how to render the docstring when you open a method.

You can also write your own renderer from scratch. For an example see the [kint-js project](https://github.com/kint-php/kint-js).

</section>
<section id="util" markdown="1">

## Utilities

These methods can come in handy.

`Kint\Parser::getCallerClass()` | Returns the class that called Kint or `null`.
`Kint\Parser::getDepthLimit()` | Returns the depth limit on the parser.
`Kint\Parser::haltParse()` | Forces the parser to ignore any further plugins and return the value.
`Kint\Parser::childHasPath()` | Tells you whether the caller has access to a property or variable based on its parent and the caller class.
`Kint\Parser::getCleanArray()` | Returns a copy of an array without the recursion marker in it. Do NOT pass an array that has had it's marker removed back into the parser, it will result in an extra recursion.

---

`Kint\Utils::getHumanReadableBytes()` | Returns an array with a unit and value of a human readable representation of the amount of bytes.
`Kint\Utils::isSequential()` | Returns true if the array is sequential.
`Kint\Utils::isAssoc()` | Returns true if the array has any string keys.
`Kint\Utils::isTrace()` | Returns true if the array appears to be a valid backtrace.
`Kint\Utils::truncateString()` | Truncates a string to a specified length, and appends an ellipsis if needed.
`Kint\Utils::getTypeString()` | Returns a string of the variable type across versions of PHP.

</section>

</div>