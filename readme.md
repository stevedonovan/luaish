## A better REPL for Lua

luaish is based on [lua.lua](http://lua-users.org/wiki/LuaInterpreterInLua) which is a Lua interpreter front-end written in Lua. 

    /mnt/extra/luaish$ lua52 lua.lua -h
    usage: lua.lua [options] [script [args]].
    Available options are:
      -e stat  execute string 'stat'
      -l name  require library 'name'
      -i       enter interactive mode after executing 'script'
      -v       show version information
      --       stop handling options
      -        execute stdin and stop handling options

Starting from this good working point, I first modified `lua.lua` to be 5.2 compatible, and added 'readline-like' support using [lua-linenoise](https://github.com/hoelzro/lua-linenoise) which is a linenoise binding by Rob Hoelz.   Although only a few hundred lines of C, linenoise is more than capable for straightforward line editing and history.  And it has enough tab completion support for our purposes.

So, say if you have typed 'st' then <tab> will give you the only matching Lua global, which is 'string'. If you now enter '.' , <tab> will cycle through all the available `string` table functions.

This also works with objects (such as strings). After 's:r' <tab> will complete 's:rep' for us:

    > s = '#'
    > = s:rep(10)
    "##########"

There is also a few shortcuts defined, so 'fn' <tab> gives 'function', and 'rt' <tab> gives 'return'.

luaish makes the command history available in the usual way, and saves it in the `~/luai-history` file.   Anything you put in the `~/luairc.lua` file will be loaded at startup.

There is an _optional_ dependency on [Microlight](https://github.com/stevedonovan/Microlight), which is only used to provide some table-dumping abilities to the REPL:

    > = {1,2;one=1}
    {1,2,one=1}

## Shell Mode

It can be irritating to have to switch between the Lua interactive prompt and the shell, as separate programs.  However, Lua would make a bad shell, in the same way (arguably) that Bash makes a poor programming language.

Any line begining with '.' is assumed to be a shell command:

    > .tail luaish.lua
    local luarc =  home..'/.luairc.lua'
    local f = io.open(luarc,'r')
    if f then
            f:close()
            dofile(luarc)
    else
            print 'no ~/.luairc.lua found'
    end

    return lsh
    > .ls
    luaish.lua  lua.lua  readme.md
    
In this shell sub-mode, tab completion switches to _working with paths_. In the above case, I typed '.tail l' and tabbed twice to get '.tail luaish.lua'.
    
`cd` is available, but is a _pseudo-command_. It changes the current working directory for the whole session, and updates the title bar of the terminal window. It acts rather like the `pushd` command, so that the pseudo-command `back` goes back to the directory you came from.

    > .cd ../lua/Penlight
    ../lua/Penlight
    > .ls
    github  Penlight  stevedonovan.github.com
    > .back
    /mnt/extra/luaish
    > .cd ../lua
    > .lua hello.lua
    hello dolly!
    > .l hello.lua
    hello dolly!
    
'l' is another pseudo-command, which is equivalent to the Lua call `dofile 'hello.lua`; thereafter '.l' will load the last named file.

Note that this works as expected:

    > .export P=$(pwd)
    > .echo $P
    /mnt/extra/luaish

But, given that luaish is just creating a subshell for commands, how can this command modify the environment of luaish?   How this is done is discussed next.

## Shell and Lua Mode communication

If a shell command ends with a '| -<fun> <args>' then 'fun' is assumed to be a function in the global table 'luaish'.  The predefined '>' function sets the global with the given name to the output, as a Lua table:

    > .ls -1 | -> ls
    > = ls
    {"luaish.lua","lua.lua","readme.md"}

Another built-in function is 'lf', which presents numbered output lines, You can then refer to the line as '$n'

    > .ls -1 | -lf
     1 luaish.lua
     2 lua.lua
     3 readme.md
    > .head -n 2 $3
    ## A better REPL for Lua
    
Any Lua globals are also expanded:

    > P = 'hello'
    > .echo $P $(pwd)
    hello /mnt/extra/luaish

The 'ls -1 |-lf' pattern is common enough that an _alias_ is provided:

    > .dir *.lua
     1 luaish.lua
     2 lua.lua

which is defined so:

    luaish.add_alias('dir','ls -1 %s |-lf')
    
Nothing fancy goes on here; any arguments to the alias are passed to the command directly.

Now the implementation of 'export' can be explained. There is a built-in function which uses [luaposix]()'s `setenv` function:

    function luaish.lsetenv (f)
        local line = f:read()
        local var, value = line:match '^(%S+) "(.-)"$'
        posix.setenv(var,value)
    end

(Note that these functions are passed a file object for reading from the shell process)

Here is the long way of using 'lsetenv':

    >.export P=$(pwd) && echo P "$P" | -lsetenv
    
And that's exactly what the builtin-command 'export' outputs when you say:

    >.export P=$(pwd)
    
Lua string values can be passed to the shell as expanded globals, but there's also an equivalent '| -' mechanism for pumping data into a shell command:

    > t = {'one','two','three'}
    > .-print t | sort
    one
    three
    two

Again, 'print' is a function in the 'luaish' table; these functions work exactly like the others, except they write to their file object. Here is a simplified implementation:

    function luaish.print (f,name)
        for _,line in ipairs(_G[name]) do 
            f:write(line,'\n')
        end
    end
    
The purpose of `~/.luairc.lua` is to let people define their own Lua filters and aliases, as well as preloading useful libraries.
    
## More Possibilities

luaish is currently in the 'executable proposal' stage of development, for people to try out and play with the possibilities.  It could do with some refactoring, so that a person may use it only as a linenoise-equiped Lua prompt, or even use that old dog `readline` itself.  Rob Hoelz and myself will be looking at how to build a more generalized and extendable framework.

Currently, you may _either_ use the 'push input' or 'pop output' forms, but not together. Since `popen2` can be implemented using luaposix, this restriction can be lifted, and we _can_ have a general mechanism for pumping Lua data through a shell filter:

    > . -print idata | sort | -> sdata

