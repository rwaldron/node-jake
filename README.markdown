### Jake -- JavaScript build tool for Node.js

### Installing

Prerequisites: Jake requires Node.js. (<http://nodejs.org/>)

Get Jake:

    git clone git://github.com/mde/node-jake.git

Build Jake:

    cd node-jake && make && sudo make install

### Installing with [npm](http://npmjs.org/)

    npm install jake

Or, get the code, and `npm link` in the code root.

### Basic usage

    jake [options] target (commands/options ...)

### Description

    Jake is a simple JavaScript build program with capabilities similar to the regular make or rake command.

    Jake has the following features:
        * Jakefiles are in standard JavaScript syntax
        * Tasks with prerequisites
        * Namespaces for tasks
        * Async task execution

### Options

    -V
    --version                   Display the program version.

    -h
    --help                      Display help information.

    -f *FILE*
    --jakefile *FILE*           Use FILE as the Jakefile.

    -C *DIRECTORY*
    --directory *DIRECTORY*     Change to DIRECTORY before running tasks.

    -T
    --tasks                     Display the tasks, with descriptions, then exit.

### Jakefile syntax


Use `task` or `file` to define tasks. Call it with three arguments (and one more optional argument):

    task(name, dependencies, handler, [async]);

Where `name` is the string name of the task (or file), `dependencies` is an array of the dependencies, and `handler` is a function to run for the task.

The `async` argument is optional, and when set to `true` (`async === true`) indicates the task executes asynchronously. Asynchronous tasks need to call `complete()` to signal they have completed.

Tasks created with `task` are always executed when asked for (or depended on). Tasks created with `file` are only executed if no file with the given name exists or if any of the files it depends on are more recent than the file named by the task. Also, if any dependency is a regular task, the file task will always be executed.


Use `desc` to add a string description of the task.

Here's an example:

    desc('This is the default task.');
    task('default', [], function (params) {
      console.log('This is the default task.');
      console.log(arguments);
    });

And here's an example of an asynchronous task:

    desc('This is an asynchronous task.');
    task('asynchronous', [], function () {
      setTimeout(complete, 1000);
    }, true);

Use `namespace` to create a namespace of tasks to perform. Call it with two arguments:

    namespace(name, namespaceTasks);

Where is `name` is the name of the namespace, and `namespaceTasks` is a function with calls inside it to `task` or `desc` definining all the tasks for that namespace.

Here's an example:

    desc('This is the default task.');
    task('default', [], function () {
      console.log('This is the default task.');
      console.log(arguments);
    });

    namespace('foo', function () {
      desc('This the foo:bar task');
      task('bar', [], function () {
        console.log('doing foo:bar task');
        console.log(arguments);
      });

      desc('This the foo:baz task');
      task('baz', ['default', 'foo:bar'], function () {
        console.log('doing foo:baz task');
        console.log(arguments);
      });

    });

In this example, the foo:baz task depends on both the default and the foo:bar task.

### Passing parameters to jake

Parameters can be passed to Jake two ways: plain arguments, and environment variables.

To pass positional arguments to the Jake tasks, enclose them in square braces, separated by commas, after the name of the task on the command-line. For example, with the following Jakefile:

    desc('This is an awesome task.');
    task('awesome', [], function () {
      console.log(Array.prototype.slice.call(arguments));
    });

You could run `jake` like this:

    jake awesome[foo,bar,baz]

And you'd get the following output:

    [ 'foo', 'bar', 'baz' ]

Note that you *cannot* uses spaces between the commas separating the parameters.

Any paramters passed after the Jake task that contain a colon (:) or equals sign (=) will be added to process.env.

With the following Jakefile:

    desc('This is an awesome task.');
    task('awesome', [], function () {
      console.log(Array.prototype.slice.call(arguments));
      console.log(process.env.qux + ' ... ' + process.env.frang);
    });

You could run `jake` like this:

    jake awesome[foo,bar,baz] qux:zoobie frang=asdf

And you'd get the following output:

    [ 'foo', 'bar', 'baz' ]
    'zoobie ... asdf'

Running `jake` with no arguments runs the default task.

### Running tasks from within other tasks

Jake supports the ability to run a task from within another task via the `invoke` and `execute` methods.

The `invoke` method will run the desired task, along with its dependencies:

    desc('Calls the foo:bar task and its dependencies.');
    task('invokeFooBar', [], function () {
      // Calls foo:bar and its deps
      jake.Task['foo:bar'].invoke();
    });

It will only run the task once, even if you call `invoke` repeatedly.

    desc('Calls the foo:bar task and its dependencies.');
    task('invokeFooBar', [], function () {
      // Calls foo:bar and its deps
      jake.Task['foo:bar'].invoke();
      // Does nothing
      jake.Task['foo:bar'].invoke();
    });

The `execute` method will run the desired task without its dependencies:

    desc('Calls the foo:bar task without its dependencies.');
    task('executeFooBar', [], function () {
      // Calls foo:bar without its deps
      jake.Task['foo:baz'].execute();
    });

Calling `execute` repeatedly will run the desired task repeatedly.

    desc('Calls the foo:bar task without its dependencies.');
    task('executeFooBar', [], function () {
      // Calls foo:bar without its deps
      jake.Task['foo:baz'].execute();
      // Can keep running this over and over
      jake.Task['foo:baz'].execute();
      jake.Task['foo:baz'].execute();
    });

If you want to run the task and its dependencies more than once, you can use `invoke` with the `reenable` method.

    desc('Calls the foo:bar task and its dependencies.');
    task('invokeFooBar', [], function () {
      // Calls foo:bar and its deps
      jake.Task['foo:bar'].invoke();
      // Does nothing
      jake.Task['foo:bar'].invoke();
      // Only re-runs foo:bar, but not its dependencies
      jake.Task['foo:bar'].reenable();
      jake.Task['foo:bar'].invoke();
    });

The `reenable` method takes a single Boolean arg, a 'deep' flag, which reenables the task's dependencies if set to true.

    desc('Calls the foo:bar task and its dependencies.');
    task('invokeFooBar', [], function () {
      // Calls foo:bar and its deps
      jake.Task['foo:bar'].invoke();
      // Does nothing
      jake.Task['foo:bar'].invoke();
      // Only re-runs foo:bar, but not its dependencies
      jake.Task['foo:bar'].reenable(true);
      jake.Task['foo:bar'].invoke();
    });

It's easy to pass params on to a sub-task run via `invoke` or `execute`:

    desc('Passes params on to other tasks.');
    task('passParams', [], function () {
      var t = jake.Task['foo:bar'];
      // Calls foo:bar, passing along current args
      t.invoke.apply(t, arguments);
    });

### Aborting a task

You can abort a task by calling the `fail` function, and Jake will abort the currently running task. You can pass a customized error message to `fail`:

    desc('This task fails.');
    task('failTask', [], function () {
      fail('Yikes. Something back happened.');
    });

Uncaught errors will also abort the currently running task.

### Showing the list of tasks

Passing `jake` the -T or --tasks flag will display the full list of tasks avaliable in a Jakefile, along with their descriptions:

    $ jake -T
    jake default       # This is the default task.
    jake asdf          # This is the asdf task. It depends on both qwer and the default
    jake concat.txt    # File task, concating two files together
    jake failure       # Failing task.
    jake lookup        # Jake task lookup by name.
    jake foo:bar       # This the foo:bar task
    jake foo:fonebone  # This the foo:fonebone task

Setting a value for -T/--tasks will filter the list by that value:

    $ jake -T foo
    jake foo:bar       # This the foo:bar task
    jake foo:fonebone  # This the foo:fonebone task

The list displayed will be all tasks whose namespace/name contain the filter-string.

### CoffeeScript Jakefiles

Jake can also handle Jakefiles in CoffeeScript. Be sure to make it Jakefile.coffee so Jake knows it's in CoffeeScript.

Here's an example:

    sys = require('sys')

    desc 'This is the default task.'
    task 'default', [], (params) ->
      console.log 'Ths is the default task.'
      console.log(sys.inspect(arguments))
      invoke 'new', []

    task 'new', [], ->
      console.log 'ello from new'
      invoke 'foo:next', ['param']

    namespace 'foo', ->
      task 'next', [], (param) ->
        console.log 'ello from next with param: ' + param

### Related projects

James Coglan's "Jake": <http://github.com/jcoglan/jake>

Confusingly, this is a Ruby tool for building JavaScript packages from source code.

280 North's Jake: <http://github.com/280north/jake>

This is also a JavaScript port of Rake, which runs on the Narwhal platform.

### Author

Matthew Eernisse, mde@fleegix.org

### Contributors

Mark Wubben / EqualMedia <mark.wubben@equalmedia.com>
Patrick Walton <pcwalton@mimiga.net>
Andrzej Sliwa <andrzej.sliwa@i-tool.eu>
Nikolay V. Nemshilov aka St <nemshilov@gmail.com>
Sascha Teske <sascha.teske@gmail.com>

