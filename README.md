# TaskMachine
### Modular micro-service task pipelining & orchestration with validated state machine integrity.

Define micro-service tasks and arrange them into state machine managed flows with a simple and expressive API. You can create build tool chains, processing pipelines, utility services, or even serve web pages...

#### State machines for the masses!

# Examples
## Simple pipeline
We define two simple inline tasks which are independent. The machine executes the two tasks in order and then finishes.
```php
$tm = new TaskMachine;

$tm->task('hello', function () {
  echo 'Hello World';
});

$tm->task('goodbye', function () {
  echo 'Goodbye World';
});

// Define a machine
$tm->machine('greetings')
  // specify an initial task and transition
  ->first('hello')->then('goodbye')
   // specify a final task
  ->finally('goodbye')
  ->build();

// Run the machine.
$tm->run('greetings');
```

## Pipeline with DI
Now we introduce some more tasks with DI. Tasks are isolated by definition and optionally have expected inputs and outputs.
```php
// Bootstrap your own Auryn injector and throw it in
$tm = new TaskMachine($myInjector);

$tm->task(
  'translate',
  function (InputInterface $input, MyTranslationInterface $translator) {
    // Auryn injects service fully constructed. Run your things.
    $translation = $translator->translate($input->get('text'));
    return ['text' => $translation];
  }
);

// Input from previous task is injectable and immutable
$tm->task('echo', function(InputInterface $input) {
  echo $input->get('text');
});

$tm->task('goodbye', function () {
  return ['closing' => 'Goodbye World'];
});

 // Define a machine
$tm->machine('translator')
  ->first('translate')->then('echo')
  ->task('echo')->then('goodbye')
  ->finally('goodbye')
  ->build();

// Run with input and then echo the output from the last task
$output = $tm->run('translator', ['text' => 'Hello World']);
echo $output->get('closing');
```

>## Any faults in the configuration of your machine will result in a build error! Tasks must be linked together correctly and have valid unambiguous transitions.

## Conditional branching
Machines can branch to different tasks based on conditions written in Symfony Expression Language.
```php
$tm = new TaskMachine($myInjector);

// A task which outputs a random true or false result
$tm->task('process', function () {
  $result = (bool)random_int(0,1);
  return ['success' => $result];
});

// Task with your instantiated object which implements TaskHandlerInterface
$tm->task('finish', new Finisher($myService));

// Task with your handler which implements TaskHandlerInterface
// Your dependencies are injected
$tm->task('fail', MyCustomServiceInterface::class);

// Define a machine with multiple final tasks
$tm->machine('switcher')
  ->first('process')
    // Specify switch conditions and subsequent tasks
    ->when([
       'output.success' => 'finish',
       '!output.success' => 'fail'
    ])
  ->finally('finish')
  ->finally('fail')
  ->build();
  
// Run it.
$tm->run('switcher');
```
