BPM Library: Arg
================

Argument processing in Bash. It's simplistic, but pretty useful. These functions will help you parse arguments that are passed into your script. This document shows real-world scenarios and solutions.


Options versus Arguments
------------------------

An option is a command-line parameter that looks like `--option-name`, `-v` or `--thing=value`. They all have a hyphen at the beginning. If there is a value, it must be immediately after an equal sign. Using `--option value` is *not supported*.

Arguments can include options, which can be confusing. To help out, `arg::getArgument` and `arg::getArguments` supply only arguments that would not be parsed as options.

It gets a bit tricky because things after a double hyphen (`--`) are all treated as arguments, not options. This is explained better with an example. Suppose we are processing this command:

    scriptName --opt -o arg1 --opt2=value arg2 -- --arg3=value -a

Options: `--opt` and `-o` are given the value `true`. The last option, `--opt2=value` is parsed as `--opt2` and is given the value `value`.

Arguments: `arg1` and `arg2` are arguments that appear before the double hyphen and. They are hidden among the options but don't have hyphens and are clearly arguments. `--` is ignored and signals the end of options. Because everything after `--` is an argument, then `--arg3=value` and `-a` are not treated as options and are arguments.


Retrieving Single Options
-------------------------

Consider the times when you want a simple flag that would enable features, such as a verbose mode.

    # Find if `--verbose` was passed.
    arg::getOption verbose --verbose "$@"

    [[ -n "$verboseFlag" ]] && echo "Verbose mode enabled"

The value of `$verboseFlag` is set to `true`. That's the default value for all options that do not have a value explicitly set.

Checking if a flag was passed in seems easy. It's not any harder to also look for particular values.

    # The --user and --role options are mandatory.
    arg::requireOptions --user --role -- "$@" || exit 1

    # Get the user and role.
    arg::getOption user --user "$@"
    arg::getOption role --role "$@"

    # The --verbose option is optional.
    arg::getOption verboseFlag --verbose "$@"

    echo "USER:  $user"
    echo "ROLE:  $role"

    [[ -n "$verboseFlag" ]] && echo "Verbose flag enabled"


Retrieving Single Arguments
---------------------------

Normally one would use `$1` to get the first argument. When you start processing options, the options could come before or after arguments, so `$1` could be anything. To make sure you get the first non-option argument you would use `arg::getArgument`.

    # Get the first non-option argument and save it as $source.
    # Get the second non-option argument and save it as $destination.
    arg::getArgument $source 0 "$@"
    arg::getArgument $destination 1 "$@"

Please be careful! The indexing on this command starts at zero. So, in the example above, you may normally say `$1` for `$source` and instead `arg::getArgument` uses index `0`.

If there are not enough arguments in `$@` then the variable is set to an empty string.

You can see how useful this is when we start mixing arguments and options. Let's make a helper function:

    diagnose() {
        arg::getArgument first 0 "$@"
        arg::getArgument second 1 "$@"
        echo "${first}...${second}"
    }

    # In comments I will show you the output of the following commands

    diagnose
    # ...

    diagnose a b
    # a...b

    diagnose a b --option1 --option2
    # a...b

    diagnose --key=value -abc "complex example" --left --right "still works"
    # complex example...still works

That last example really illustrates the goal of these functions.


Passing Arguments Through
-------------------------

Imagine a function that will consume one argument and pass the rest to another function. Perhaps we get a username as the first argument and a home directory as a second.

    setupUserDir() {
        local args oldIfs

        # Get a list of all non-option arguments
        arg::getArguments args "$@"

        # Remove the first item from the list. The IFS notation is to
        # ensure this works even in Bash 3, regardless of IFS settings.
        oldIfs=$IFS
        IFS=
        args=("${args[@]:1}")
        IFS=$oldIfs

        # Pass the remaining arguments to another function
        setupDir "${ARGS[@]}"
    }


Passing Options Through
-----------------------

Let's say you have a function called `setupService` and it wants to support all of the options that `makeTargetFile` supports.

    setupService() {
        local options

        wickGetOptions options "$@"

        # $options is now set to a list of all options that were passed
        # in to this function. This next bit of trickery will pass all options
        # at the end of the command. It also will not create an empty value
        # when there were no options.
        makeTargetFile source-file.txt /dest/folder/ ${options[@]+"${options[@]}"}
    }

To make it more complex, we alter `setupService` to accept a `--reload` parameter but we don't want to pass `--reload` to `makeTargetFile`. This also uses the `array::filter` function from the array library.

    bpm::include array

    setupServiceOptionsFilter() {
        case "$1" in
            --reload|reload=*)
                # Return failure to remove the item from the array. Match
                # against both --reload and --reload=* because the option
                # could be specified in either way by the user.
                return 1
                ;;
        esac

        # Keep this element in the array.
        return 0
    }

    setupService() {
        local options

        arg::getOptions options "$@"

        # $options could have `--reload`
        # This is how we remove items from the array

        array::filter options setupServiceOptionsFilter ${options[@]+"${options[@]}"}

        # $options now will not have `--reload`
        makeTargetFile source-file.txt /dest/folder/ ${options[@]+"${options[@]}"}
    }


Installation
============

Add to your `bpm.ini` file the following dependency.

    [dependencies]
    arg=*

Run `bpm install` to add the library. Finally, use it in your scripts.

    #!/usr/bin/env bash
    . bpm
    bpm::include arg

    arg::getOption value --my-option "$@"


API
===


[//]: # (AUTOGENERATED FROM libarg - START)

`arg::getArgument()`
--------------------

Retrieves a single argument from the list of arguments.  It's similar to `arg::getOption`, except it only returns non-options that were passed to a function.

* $1    - Name of the variable that should get the result.
* $2    - Index of the non-option argument, starting with 0.
* $3-$@ - Command line arguments to parse.  Typically you use `"$0"` here.

Examples

    # Get the first non-option and place it into $name
    # If the argument does not exist, sets $name to ""
    arg::getArgument name 0 "$@"

Returns errors if any other functions generate them.


`arg::getArguments()`
---------------------

Grab all non-option arguments and return them as an array.

* $1    - Name of variable that should receive the result.
* $2-$@ - Command line arguments to parse.  Typically this is `"$@"`.

Examples

    # Any non-option arguments are placed into $arguments.
    # If there were none, $arguments is set to an empty list.
    arg::getArguments arguments "$@"

Returns nothing.


`arg::getOption()`
------------------

Retrieve a named option from the list of arguments.

* $1    - Name of variable where the value will be stored.
* $2    - Option's name.  May have leading hyphens (eg. `--option-name` and `option-name` are both valid).
* $3-$@ - Arguments to parse.  Typically this is `"$@"`.

This splits up single-hyphen options. Thus, if `-abc` goes in, it is treated as `-a` and `-b` and `-c` individually.

Examples

    # If --verbose=XYZ is passed, $verboseOption will be set to "XYZ".
    # If --verbose= is passed (empty value), $verboseOption is set to "".
    # If --verbose is passed without a value, $verboseOption is set to "true".
    # If --verbose is not passed at all, $verboseOption is set to "".
    arg::getOption verboseOption verbose "$@"

    # Split single-hyphen options:
    # Assuming that -abc is passed in ...
    arg::getOption letterC c "$@"
    arg::getOption letterD d "$@"
    echo "$letterC" # Echos "true"
    echo "$letterD" # Echos nothing

Returns nothing.


`arg::getOptions()`
-------------------

Get all options from a list of arguments and return a list without any processing.

* $1    - Name of variable where the array will be stored.
* $2-$@ - Arguments that are to be parsed.  Typically this is `"$@"`.

This does not split up single-hyphen options. That's because the purpose of this function is to simply extract all options from a list of arguments that were passed to a function.

Examples:

    arg::getOptions opts "$@"
    # $opts is now a list of all options that were passed to the script.
    # $opts is unprocessed, so "-abc" would still be kept as "-abc" instead
    # of converting to "-a", "-b", and "-c".

Returns nothing.


`arg::requireArguments()`
-------------------------

Confirm that there's a specific number of arguments and that those arguments are not empty. Options are not counted.

* $1    - Number of elements that are required.
* $2-$@ - Command line arguments to parse.  Typically this is `"$@"`.

Examples

    # Ensure the $@ has 4 non-empty arguments.
    arg::requireArguments 4 "$@"

    # This would fail the test and return 1 because the second argument is
    # empty.
    array=(first "" third)
    arg::requireArguments 3 "${array[@]}"

    # This one requires exactly two arguments, which is correct. The options
    # are ignored.
    array=(--option1=value --option2 argument1 --option2=again argument2)
    arg::requireArguments 2 "${array[@]}"

Returns true (0) if arguments are present and have non-empty values, 1 if not. If incorrect arguments are passed, this returns 2.


`arg::requireOptions()`
-----------------------

Guarantee that some options are passed to a script.  This is typically used within a role or a formula's `run` or `depends` script.  The error reporter can be overridden so the library function can be used in external scripts as well.  If any options are missing, this function returns an error status code and calls `arg::requireOptionsFailure` once for each missing argument.

* $1-$x - Required option names. Leading hyphens may be omitted.
* --    - A double hyphen separates the required options from the argument list.
* $y-$@ - The list of arguments.  Typically this is `"$@"`.

Examples

    # Test to make sure this function or file received both
    # "access-key" and "secret-key"
    arg::requireOptions --access-key --secret-key -- "$@"

    # Same as above - you may include or exclude the leading hyphens before
    # the required options. You still need to use "--" to separate the two
    # lists.
    arg::requireOptions access-key secret-key -- "$@"

    # Use your own reporter and make sure that both --mom and --dad are set.
    # This would get called once for each option that is missing.
    missingOption() {
        echo "Hey, you need to specify --$1 as an argument"
    }
    ARG_REQUIRE_OPTIONS_FAILURE=mmissingOption \
        arg::requireOptions --mom --dad -- "$@"

Returns zero if options exist, one if they do not.


`arg::requireOptionsFailure()`
------------------------------

Default failure reporter for `arg::requireOptions`. Override this function with one of your own or set `$ARG_REQUIRE_OPTIONS_FAILURE` to call another function.

* $1                           - Option that was not specified.
* $ARG_REQUIRE_OPTIONS_FAILURE - Alternate function or command to call with the same arguments as was passed to this function.

Returns nothing.


`arg::safeName()`
-----------------

Change a string so it is a valid variable name in Bash.

* $1 - Name of the variable that should receive the altered string.
* $2 - The string we are making safe.

You can't use variables like `$ABC-DEF` because the hyphen is invalid for a variable's name. This function will turn the hyphen and other invalid characters into an underscore. This is a one-way transform because it maps all invalid characters to an underscore and there's no way to determine what the original character was.

Some extensive testing was done for `tomdoc.sh` and Bash variable names can have a leading character of `[A-Za-z_]`, while subsequent characters can be `[A-Za-z0-9=_]`.

Examples

    arg::safeName fixed "ABC-DEF"
    # $fixed is "ABC_DEF"

Returns nothing.

[//]: # (AUTOGENERATED FROM libarg - END)


License
=======

This project is placed under an [MIT License](LICENSE.md).
