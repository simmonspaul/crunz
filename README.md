# Crunz

Install a cron job once and for all, manage the rest from the code.

Crunz is a framework-agnostic package to schedule periodic tasks (cron jobs) in PHP using a fluent API.

Crunz is capable of executing any kind of executable command as well as PHP closures.

[![Version](http://img.shields.io/packagist/v/lavary/crunz.svg?style=flat-square)](https://packagist.org/packages/lavary/crunz)
[![Packagist](https://img.shields.io/packagist/dt/lavary/crunz.svg?style=flat-square)](https://packagist.org/packages/lavary/crunz/stats)
[![Packagist](https://img.shields.io/packagist/dm/lavary/crunz.svg?style=flat-square)](https://packagist.org/packages/lavary/crunz/stats)

|Version|Supported PHP versions|Linux build|Windows build|
|---|---|---|---|
|stable (v2.0.2)|![7.2+](https://img.shields.io/badge/php-%5E7.2-blue.svg?style=flat-square)|[![Build Status](https://img.shields.io/travis/lavary/crunz/v2.0.2.svg?style=flat-square)](https://travis-ci.org/lavary/crunz)|*Tag build not supported*
|dev v2 (master/v2.x-dev)|![7.2+](https://img.shields.io/badge/php-%5E7.2-blue.svg?style=flat-square)|[![Build Status](https://img.shields.io/travis/lavary/crunz/master.svg?style=flat-square)](https://travis-ci.org/lavary/crunz)|[![AppVeyor branch](https://img.shields.io/appveyor/ci/lavary/crunz/master.svg?style=flat-square)](https://ci.appveyor.com/project/lavary/crunz)
|dev v1.12.x (v1.12.x-dev)|![5.6+](https://img.shields.io/badge/php-%5E5.6%20%7C%7C%20%5E7.0-blue.svg?style=flat-square)|[![Build Status](https://img.shields.io/travis/lavary/crunz/1.12.x.svg?style=flat-square)](https://travis-ci.org/lavary/crunz)|[![AppVeyor branch](https://img.shields.io/appveyor/ci/lavary/crunz/1.12.x.svg?style=flat-square)](https://ci.appveyor.com/project/lavary/crunz)

## Roadmap
|Version|Release date|Active support until|Bug support until 
|---|---|---|---|
|v1 (v1.12.x)|April 2016|April 2019|April 2020
|v2 (master/v2.x-dev)|April 2019|April 2021|April 2022

## Installation

To install it:

```bash
composer require lavary/crunz
```
If the installation is successful, a command-line utility named **crunz** is symlinked to the `vendor/bin` directory of your project. 

## How It Works?

The idea is very simple: instead of installing cron jobs in a crontab file, we define them in one or several PHP files, by using the Crunz interface. 

Here's a basic example:

```php
<?php
// tasks/backupTasks.php

use Crunz\Schedule;

$schedule = new Schedule();
$task = $schedule->run('cp project project-bk');       
$task->daily();

return $schedule;
```

To run the tasks, you only need to install an ordinary cron job (a crontab entry) which runs **every minute**, and delegates the responsibility to Crunz' event runner:

```bash
* * * * * cd /project && vendor/bin/crunz schedule:run
```

The command `schedule:run` is responsible for collecting all the PHP task files and run the tasks which are due.

## Task Files

Task files resemble crontab files. Just like crontab files they can contain one or more tasks.

Normally we create our task files in the `tasks/` directory within the project's root directory. 

> By default, Crunz assumes all the task files reside in the `tasks/` directory within the project's root directory.

There are two ways to specify the source directory: 1) Configuration file  2) As a parameter to the event runner command.
 
We can explicitly set the source path by passing it to the event runner as a parameter:

```bash
* * * * * cd /project && vendor/bin/crunz schedule:run /path/to/tasks/directory
```

### Creating a Simple Task

In the terminal, change the directory to your project's root directory and run the following commands:

```bash
mkdir tasks && cd tasks
nano GeneralTasks.php
```

Then, add a task as below:

```php
<?php
// tasks/FirstTasks.php

use Crunz\Schedule;

$schedule = new Schedule();

$task = $schedule->run('cp project project-bk'); 
$task
    ->daily()
    ->description('Create a backup of the project directory.');

// ...

// IMPORTANT: You must return the schedule object
return $schedule; 
```

There are some conventions for creating a task file, which you need to follow. First of all, the filename should end with `Tasks.php` unless we change this via the configuration settings.  

In addition to that, we **must** return the instance of `Schedule` class at the end of each file, otherwise, all the tasks inside the file will be skipped by the event runner.

Since Crunz scans the tasks directory recursively, we can either put all the tasks in one file or across different files (or directories) based on their usage. This behavior helps us have a well organized tasks directory.


## The Command

We can run **any** command or script by using `run()`. This method accepts two arguments:  **the command to be executed**, and **the command options** (as an associative array) if there's any.

### Normal Command or Script

```php
<?php

use Crunz\Schedule;

$schedule = new Schedule();
$task = $schedule->run('/usr/bin/php backup.php', ['--destination' => 'path/to/destination']);
$task
    ->everyMinute()
    ->description('Copying the project directory');

return $schedule;
```

In the above example, `--destination` is an option supported by `backup.php` script.

### Closures

We can also write to a closure instead of a command:

```php
<?php

use Crunz\Schedule;

$schedule = new Schedule();

$x = 12;
$task = $schedule->run(function() use ($x) { 
   // Do some cool stuff in here 
});

$task
    ->everyMinute()
    ->description('Copying the project directory');

return $schedule;
```

## Frequency of Execution

There are a variety of ways to specify **when** and **how often** a task should run. We can combine these methods together to get our desired frequencies. 

### Units of Time

There are a group of methods which specify a unit of time (bigger than minute) as frequency. They usually end with `ly` suffix, as in `hourly()`, `daily()`, `weekly`, `monthly()`, `quarterly()`, and  `yearly` .

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php backup.php');    
$task->daily();
// ...
```

The above task will run **daily at midnight**.

Here's another one, which runs on the **first day of each month**.

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php email.php');
$task->monthly();
// ...
```

> All the events scheduled with this set of methods happen at the **beginning** of that time unit. For example `weekly()` will run the event on Sundays, and `monthly()` will run it on the first day of each month.

### Dynamic Methods

Dynamic methods give us a wide variety of frequency options on the fly. We just need to follow this pattern:

```php
every[NumberInCamelCaseWords]Minute|Hour|Day|Months?
```

As we can see, the method should start with the word `every`, followed by a number in camel-case words, ending with one of these units of time: **minute, hour, day, and month**. 

The `s` at the end is optional and it's just used for grammar's sake.

With that said, the following methods are valid:

* `everyFiveMinutes()`
* `everyMinute()`
* `everyTwelveHours()`
* `everyMonth`
* `everySixMonths()`
* `everyFifteenDays()`
* `everyFiveHundredThirtySevenMinutes()`
* `everyThreeThousandAndFiveHundredFiftyNineMinutes()`
* ...

This is how it is used in a task file:

```php
<?php
// ...

$task = $schedule->run('/usr/bin/php email.php');
$task->everyTenDays();

$task = $schedule->run('/usr/bin/php some_other_stuff.php');
$task->everyThirteenMinutes();
// ...

return $schedule;
```

### Running Events at Certain Times

To schedule one-off tasks, you may use `on()` method like this:

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php email.php'); 
$task->on('13:30 2016-03-01');
// ...
```

The above the task will run on the first of march 2016 at 01:30 pm. 

> `On()` accepts any date format parsed by PHP's [strtotime](http://php.net/manual/en/function.strtotime.php) function.

To specify the time we use `at()` method:

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php email.php'); 
$task
    ->daily()
    ->at('13:30');
// ...
```

We can use `dailyAt()` to get the same result:

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php email.php');       
$task->dailyAt('13:30');
// ...
```

If we only pass a time to the `on()` method, it will have the same effect as using `at()`

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php email.php');   
$task
    ->mondays()
    ->on('13:30');
         
// is the sames as
$task = $schedule->run('/usr/bin/php email.php');       
$task
    ->mondays()
    ->at('13:30');
// ...
```

### Weekdays

Crunz also provides a set of methods which specify a certain day in the week. These methods have been designed to be used as a constraint and should not be used alone. The reason is that weekday methods just modify the `Day of Week` field of a cron job expression.

Consider the following example:

```php
<?php
// Cron equivalent:  * * * * 1
$task = $schedule->run('/usr/bin/php email.php');
$task->mondays();
```

At first glance, the task seems to run **every Monday**, but since it only modifies the "day of week" field of the cron job expression, the task  runs **every minute on Mondays**.

This is the correct way of using weekday methods:

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyThreeHours()
    ->mondays();
// ...
```

In the above task, we use `mondays()` as a constraint to run the task **every three hours on Mondays**.

### Setting Individual Fields

Crunz's methods are not limited to the ready-made methods explained. We can also set individual fields to compose our custom frequencies. These methods either accept an array of values, or list arguments separated by commas:

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php email.php');
$task       
    ->minute(['1-30', 45, 55])
    ->hour('1-5', 7, 8)
    ->dayOfMonth(12, 15)
    ->month(1);
```

Or:

```php
<?php
// ...
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->minute('30')
    ->hour('13')
    ->month([1,2])
    ->dayofWeek('Mon', 'Fri', 'Sat');

// ...
```

### The Classic Way

We can also do the scheduling the old way, just like we do in a crontab file:

```php
<?php

$task = $schedule->run('/usr/bin/php email.php');
$task->cron('30 12 * 5-6,9 Mon,Fri');
```

Based on our use cases, we can choose and combine the proper set of methods, which are easier to use.

### Force run

While in development it may be useful to force run all tasks regardless of their actual run time,
which can be achieved by adding `--force` to `schedule:run`:

```bash
vendor/bin/crunz schedule:run --force
```

## Changing Directories

You can use the `in()` method to change directory before running a command:

```php
<?php

// ...

$task = $schedule->run('./deploy.sh');
$task
    ->in('/home')
    ->weekly()
    ->sundays()
    ->at('12:30')
    ->appendOutputTo('/var/log/backup.log');

// ...

return $schedule;
```

## Task Life Time

In a crontab entry, we can not easily specify a task's lifetime (the period of time when the task is active). However, it's been made easy in Crunz:

```php
<?php
//
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyFiveMinutes()
    ->from('12:30 2016-03-04')
    ->to('04:55 2016-03-10');
 //       
```
Or alternatively we can use the `between()` method to accomplish the same result:

```php
<?php
//
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyFiveMinutes()
    ->between('12:30 2016-03-04', '04:55 2016-03-10');

 //       
```

If we don't specify the date portion, the task will be active **every** day but only within the specified duration:

```php
<?php
//
$task = $schedule->run('/usr/bin/php email.php');
$task
     ->everyFiveMinutes()
     ->between('12:30', '04:55');

 //       
```

The above task runs **every five minutes** between **12:30 pm** and **4:55 pm** every day.

## Running Conditions

Another thing that we cannot do very easily in a traditional crontab file is to make conditions for running the tasks. This has been made easy by `when()` and `skip()` methods.

Consider the following code:

```php
<?php
//
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyFiveMinutes()
    ->between('12:30 2016-03-04', '04:55 2016-03-10')
    ->when(function() {
        if ((bool) (time() % 2)) {
            return true;
        }
        
        return false;
    });
```

Method `when()` accepts a callback,  which must return `TRUE` for the task to run. This is really useful when we need to check our resources before performing a resource-hungry task.

We can also skip a task under certain conditions, by using `skip()` method. If the passed callback returns `TRUE`, the task will be skipped.

```php
<?php
//
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyFiveMinutes()
    ->between('12:30 2016-03-04', '04:55 2016-03-10')
    ->skip(function() {
        if ((bool) (time() % 2)) {
            return true;
        }
        
        return false;  
    });

 //       
```

We can use these methods **several** times for a single task. They are evaluated sequentially.


## Configuration

There are a few configuration options provided by Crunz in YAML format. To modify the configuration settings, it is highly recommended to have your own copy of the configuration file, instead of modifying the original one. 

To create a copy of the configuration file, first we need to publish the configuration file:

```bash
/project/vendor/bin/crunz publish:config
The configuration file was generated successfully
```

As a result, a copy of the configuration file will be created within our project's root directory.

 The configuration file looks like this:

```yaml
# Crunz Configuration Settings

# This option defines where the task files and
# directories reside.
# The path is relative to the project's root directory,
# where the Crunz is installed (Trailing slashes will be ignored).
source: tasks

# The suffix is meant to target the task files inside the ":source" directory.
# Please note if you change this value, you need
# to make sure all the existing tasks files are renamed accordingly.
suffix: Tasks.php

# Timezone is used to calculate task run time
# This option is very important and not setting it is deprecated
# and will result in exception in 2.0 version.
timezone: ~

# This option define which timezone should be used for log files
# If false, system default timezone will be used
# If true, the timezone in config file that is used to calculate task run time will be used
timezone_log: false

# By default the errors are not logged by Crunz
# You may set the value to true for logging the errors
log_errors: false

# This is the absolute path to the errors' log file
# You need to make sure you have the required permission to write to this file though.
errors_log_file:

# By default the output is not logged as they are redirected to the
# null output.
# Set this to true if you want to keep the outputs
log_output: false

# This is the absolute path to the global output log file
# The events which have dedicated log files (defined with them), won't be
# logged to this file though.
output_log_file:

# By default line breaks in logs aren't allowed.
# Set the value to true to allow them.
log_allow_line_breaks: false

# This option determines whether the output should be emailed or not.
email_output: false

# This option determines whether the error messages should be emailed or not.
email_errors: false

# Global Swift Mailer settings
#
mailer:
    # Possible values: smtp, mail, and sendmail
    transport: smtp
    recipients:
    sender_name:
    sender_email:


# SMTP settings
#
smtp:
    host:
    port:
    username:
    password:
    encryption:
```

As you can see there are a few options like `source` which is used to specify the source tasks directory. The other options are used for error/output logging/emailing purposes.

Each time we run Crunz commands, it will look into the project's root directory to see if there's any user-modified configuration file. If the configuration file doesn't exists, it will use the one shipped with the package.


## Parallelism and the Locking Mechanism

Crunz runs the scheduled events in parallel (in separate processes), so all the events which have the same frequency of execution, will run at the same time asynchronously. To achieve this, Crunz utilizes the [symfony/Process](http://symfony.com/doc/current/components/process.html) library for running the tasks in sub-processes.

If the execution of a task lasts until the next interval or even beyond that, we say that the same instances of a task are overlapping. In some cases, this is a not a problem. But there are times, when these tasks are modifying database data or files, which might cause unexpected behaviors, or even data loss.

To prevent critical tasks from overlapping each other, Crunz provides a locking mechanism through `preventOverlapping()` method, which, ensures no task runs if the previous instance is already running. 

```php
<?php
//
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyFiveMinutes()
    ->preventOverlapping();
 //       
```

By default, crunz uses file based locking (if no parameters are passed to `preventOverlapping`). For alternative lock mechanisms, crunz uses the [symfony/lock](https://symfony.com/doc/current/components/lock.html) component that provides lock mechanisms with various stores. To use this component, you can pass a store to the `preventOverlapping()` method. In the following example, the file based `FlockStore` is used to provide an alternative lock file path.

```php
<?php

use Symfony\Component\Lock\Store\FlockStore;

$store = new FlockStore(__DIR__ . '/locks');
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyFiveMinutes()
    ->preventOverlapping($store);

```

## Keeping the Output

Cron jobs usually have outputs, which is normally emailed to the owner of the crontab file, or the user(s) set by the `MAILTO` environment variable inside the crontab file.

We can also redirect the standard output to a physical file using `>` or `>>` operators:

```bash
* * * * * /command/to/run >> /var/log/crons/cron.log
```

This kind of output logging has been automated in Crunz. To automatically send each event's output to a log file, we can set `log_output` and `output_log_file` options in the configuration file accordingly:

```yaml
# Configuration settings

## ...
log_output:      true
output_log_file: /var/log/crunz.log
## ...
```

This will send the events' output (if executed successfully) to `/var/log/crunz.log` file. However, we need to make sure we are permitted to write to the respective file.

If we need to log the outputs on an event-basis, we can use `appendOutputTo()` or `sendOutputTo()` methods like this:

```php
<?php
//
$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyFiveMinutes()
    ->appendOutputTo('/var/log/crunz/emails.log');

 //       
```

Method `appendOutputTo()` **appends** the output to the specified file. To override the log file with new data after each run, we use `saveOutputTo()` method.

It is also possible to send the errors as emails to a group of recipients by setting `email_output` and `mailer` settings in the configuration file.

## Error Handling

Crunz makes error handling easy by logging and also allowing you add a set of callbacks in case of an error.

## Error Callbacks

You can set as many callbacks as needed to run in case of an error:

```php
<?php

use Crunz\Schedule;

$schedule = new Schedule();

$task = $schedule->run('command/to/run');
$task->everyFiveMinutes();

$schedule
->onError(function() {
   // Send mail
})
->onError(function() {
   // Do something else
});

return $schedule;
```

If there's an error the two defined callbacks will be executed.

## Error Logging

To log the possible errors during each run, we can set `log_error` and `error_log_file` settings in the configuration file as below:

```yaml
# Configuration settings

# ...
log_errors:      true
errors_log_file: /var/log/error.log
# ...
```

As a result, if the execution of an event is unsuccessful for some reasons, the error message is appended to the specified error log file. Each entry provides useful information including the time it happened, the event description,  the executed command which caused the error, and the error message itself.

It is also possible to send the errors as emails to a group of recipients by setting `email_error` and `mailer` settings in the configuration file.

## Pre-Process and Post-Process Hooks

There are times when we want to do some kind of operations before and after an event. This is possible by attaching pre-process and post-process callbacks to the respective event.

To do this, we use `before()` and `after()` on both `Event` and `Schedule` objects, meaning we can have pre and post hooks on an event-basis as well as schedule basis. The hooks bind to schedule will run before all events, and after all the events are finished.

```php
<?php

use Crunz\Schedule;

$schedule = new Schedule();

$task = $schedule->run('/usr/bin/php email.php');
$task
    ->everyFiveMinutes()
    ->before(function() { 
        // Do something before the task runs
    })
    ->before(function() { 
        // Do something else
    })
    ->after(function() {
        // After the task is run
    });
 
$schedule
    ->before(function () {
       // Do something before all events
    })
    ->after(function () {
       // Do something after all events are finished
    })
    ->before(function () {
       // Do something before all events
    });
```

> We might need to use these methods as many times we need by chaining them.

Post-execution callbacks are only called if the execution of the event has been successful.

## Other Useful Commands

We've already used a few of `crunz` commands like `schedule:run` and `publish:config`. 

To see all the valid options and arguments of `crunz`, we can run the following command:

```bash
vendor/bin/crunz --help
```

### Listing Tasks

One of these commands is `crunz schedule:list`, which lists the defined tasks (in collected `*.Tasks.php` files) in a tabular format.

```php
vendor/bin/crunz schedule:list

+---+---------------+-------------+--------------------+
| # | Task          | Expression  | Command to Run     |
+---+---------------+-------------+--------------------+
| 1 | Sample Task   | * * * * 1 * | command/to/execute |
+---+---------------+-------------+--------------------+
```

### Generating Tasks

There is also a useful command named `make:task`, which generates a task file skeleton with all the defaults, so we won't have to write them from scratch. We can modify the output file later based on our requirements. 

For example, to create a task, which runs `/var/www/script.php` every hour on Mondays, we run the following command:

```text
vendor/bin/crunz make:task exampleOne --run scripts.php --in /var/www --frequency everyHour --constraint mondays
Where do you want to save the file? (Press enter for the current directory)
```

When we run this command, Crunz will ask about the location we want to save the file. By default, it is our source tasks directory.

As a result, the event is defined in a file named `exampleOneTasks.php` within the specified tasks directory.

To see if the event has been created successfully, we list the events:

```text
crunz schedule:list

+---+------------------+-------------+----------------+
| # | Task             | Expression  | Command to Run |
+---+------------------+-------------+----------------+
| 1 | Task description | 0 * * * 1 * | scripts.php    |
+---+------------------+-------------+----------------+
```

To see all the options of `make:task` command with all the defaults, we run this:

```bash
vendor/bin/crunz make:task --help
```

## Development ENV flags

The following environment flags should be used only while in development.
Typical end-users do not need to, and should not, change them.

### `CRUNZ_CONTAINER_DEBUG`

Flag used to enable/disable container debug mode, useful only for development.
Enabled by default in `docker-compose`.

### `CRUNZ_DEPRECATION_HANDLER`

Flag used to enable/disable Crunz deprecation handler, useful only for integration tests.
Disabled by default for tests.

## Contributing

### Which branch should I choose?

In most cases you should target branch `1.12.x`, as this is active development branch.
Branch `master` is for future release, but all bugs/features should go to `1.11.x` anyway.

## If You Need Help

Please submit all issues and questions using GitHub issues and I will try to help you.


## Credits

* [PabloKowalczyk](https://github.com/PabloKowalczyk)
* [Reza Lavarian](https://github.com/lavary)
* [All Contributors](https://github.com/lavary/crunz/graphs/contributors)

## License
Crunz is free software distributed under the terms of the MIT license.
