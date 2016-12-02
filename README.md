# What is it?
Pure is a *specification* for a configuration file format. Its goal is to
suck less than other configuration file formats.

Most people will find it entirely natural to read and edit. See examples below.

Any good configuration formats needs to please three kinds of people:

* Programmers
* Operations and support staff
* End users

Yes, it's true: sometimes you (often through support staff) need to ask an end user
to change a setting in a config file. By "end user" I mean anything from a fairly technical customer
installing your server software, to a non-technical dude trying to help debug an issue on
a desktop client you made.

And you really want them to succeed doing that.

# XML, JSON and YAML are pure too. Pure crap.
Anecdotal evidence exists.

The worst formats to ask people to edit (programmers included) is *by far* YAML and JSON.

They are guaranteed to accidentally screw things up. Like misplacing or removing a "," or "}" in JSON. It happens
all the time, and it's not always easy to spot these mistakes for people trying to help you using Notepad. Over the phone.

Considering asking a customer to install Sublime + jslint says a lot about the steaming pile of shit called JSON.

Moreover, few non-technical people seem to understand all the weird sigils present in a YAML file. It's butt ugly too.

XML is hard to edit for unsuspecting end users. Requests like "edit the quoted value after the equals sign
inside the log tag, you know, just before the greater-than sign" tends to fail. And did I mention they all
double click XML files to edit them, after which they'll ask why IE 9 launches? Okay, that's not a problem with the format though, just
really annoying.

# Old formats were easier to edit, but...
People have few problems editing "good old" INI files, and Java-style property files are easy enough too. Problem is, they
are not very good at deeply nested structures. For instance, you can get hierarchies by naming conventions in property files,
but you end up with a *lot* of repeating prefixes.

# A format made from first principles
To please all camps, then, you need a format that:

* Is easy to understand for everyone. No mysterious sigils. No superfluous characters to guide parsers. Values and their names is about it.
* Support comments, to help realize the first point.
* Allows for flat config files
* Allows for hierarchies of properties
* Allows for graphs of properties (that is, referencing other parts of the config file for reuse)
* Understands Unicode values
* Allows for strict validation as an option (structure and type checking)

# Comment on comments
Douglas Crockford of JSON fame claims that comments in JSON files were bad,
so he removed them. You know, because they end up being used as directives.
Guess what? Bogus property naming conventions ended up being used as directives instead.

By the way, Douglas nuked comments with a straight face, convincing thousands of
people it was a good idea.

Comments in config files are essential. They are obviously useful and helpful, with
no real downsides.

Everybody at the same time: **"Not allowing comments in a config file is a crooked crock of crap"**

# Filename
Call it *server.settings*, *db.conf*, *app.properties*, *data.pure* or whatever you want.

What matters is that the content of the config file doesn't suck. And that there are comments.

# Examples, finally!
The claim was a flexible, yet natural structure. Let's see if it is true.

To start off, a pure file can be as simple as flat list of configuration properties:

```
port = 8443
bind = 0.0.0.0
```

It's as simple as *property = value*

Let's collect those related properties into a group called *server*

```
server.port = 8443
server.bind = 0.0.0.0
```

If we get a lot of these, we have to repeat the *server* prefix. Bad. Pure supports
nested grouping, which should be your default choice:

```
server
    port = 8443
    bind = 0.0.0.0
```

**However**. You can combine nesting and dot notation for groups. This is more useful than
you might think. For instance, you may want to place all log levels at the end of
the file, instead of spreading them all over the place:

```
# Server configuration
server
    port = 8443
    bind = 0.0.0.0

# Db configuration
database
    url = something-cool-here
    user = sys
    password = something
    timeout = 30s

    # Another nesting level (there's no limit)
    data
        path = ../data
        indexed = true

# A separate section with log levels
server.log.level = debug
database.log.level = info
```

A side note: ```user = sys``` and ```user = "sys"``` is the same thing, quotes
are optional. Joe Plumber hate quotes.

# Graph structure
You can *reference* any property, at any level, in the config file.

This is accomplished by using => instead of =

```
# Both the server and database config will reuse most of this:
shared
    log
        filename = server.log
        rolling = true
        keep-count = 10
        max-size = 50MB

# Use the same log config as the shared one, but override the max-size.
# Let's also add a data property only relevant for the server.
server
    log => shared.log
        max-size = 10MB
        date-format = "yyyy-mm-dd"

# We're happy with the log defaults, but change the file name
database
    log => shared.log
        filename = db.log
```

Indeed, this means you can serialize object graphs *with cycles* to a highly
readable format.

You can also reference a specific value.

```
vars.filename = thefile.txt

# Later on, we need to reference vars.filename
server.data => vars.filename
```

Yup, ```config.get("server.data")``` will return "thefile.txt"

# Accessing environment variables
The parser *must* replace environment variables. It's easy to implement, and very useful.

```
# The $VARNAME syntax is used on all platforms.
app.logfile = $HOME/.logs/myapp.log
```

# Format definitions
Optionally, a Pure parser may support format definitions. This is a separate file
defining the structure of a config file.

You pass this to the parser along with the config file. If the config file doesn't adhere
to the format definitions, parsing will fail.

Property definitions with default values are optional; all others are required.

Any attempt to add properties not defined in the *def* file will cause
parsing to fail (catching spelling mistakes). However, a group can contain
undefined properties if the group is marked *allow-undefined* (currently
the only annotation)

**server-config.def**
```
server
    log
        date-format: string("yyyy-mm-dd")
        keep-count: int(10)
        filename: path
        max-size: quantity(50MB)
        rolling: bool

    user-properties [allow-undefined]
```

See how the definition format mirrors the property value format? Instead of
*prop = value* we have *prop: type*

Of course, we can reference other definitions using => to avoid
duplication:

```
shared
    log
        filename: path(server.log)
        rolling: bool(true)
        max-size: quantity(50MB)
        keep-count: int(10)

server
    log => shared.log
    bind: string
    port: int(8443)
```

# Data types

The parser *should* designate a data type to every property, whether a
definition file exists or not. This makes sense only for statically typed
languages, of course.

*To allow for minimal API's, every typed value must have a string representation,
which is simply its literal string, as it appears in the file.*

The type inference rules are straightforward. Note that all values can be
null (value not present)

Why data types like "quantity" and "path" ? Because these are so frequent in
configuration files that it makes sense to provide special support for them. By
special support, we mean that the API *may* provide typed getter/setters, although
that's not a requirement.

```
Data type           Inference rule
----------------------------------------------------------------------------
string              If the value is "quoted" or 'single quoted', it's
                    always a string.

                    Moreover, any value not inferred to be of any other
                    type is a string, such as username = admin

int                 64-bit signed integer literal, such as *9500*

double              64-bit IEEE double literal, such as *5.2*

bool                Literals *true* and *false*

path                A string leading with . or containing / or \

                    The parser is required to translate path separators to
                    the proper system separator.

quantity            An integer followed by a unit, such as *250ms,
                    50MB, 60s, 200cm*

                    SI units should be used, but anything is allowed. There
                    must be an API to get both the value and the unit.
```

# API
The API in an OOP language is simply a set of get/set methods to access properties. Other languages? Whatever is idiomatic for that language.

Use "dot.notation" to reference groups. This works even if indentation style grouping is used.

Java example:

```
// Get the port
int port = config.getInt("server.port");

// Or, interpret as string, no matter what the value type is
String port = config.get("server.port");

// Check if the path exists
if (config.getPath("database.dir").exists())
  ...

// Get a quantity
int logSize = config.getQuantityValue("log.size"); // 500
String unit = config.getQuantityUnit("log.size");  // "MB"

// Same as above, using a pair type provided by the API (or whatever
// the language runtime offers)
Pair logSize = config.getQuanity("log.size");

// Set the port
config.put("server.port", 8443);
```

# What's next

* A formal specification, although the informal description above should be
enough to write a parser.
* Yes, actual parsers must be written. Pull requests accepted for any language!
A non-validating one should doable as a weekend project for your favorite language.
* Consider supporting inline type definitions, such as *username: string = admin* 
