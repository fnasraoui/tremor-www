# Tremor Basics

This guide will walk you through the tremor basics. It'll be a starting point to learning how to get data into the system, work with this data and get it out again.

The example is entirely synthetic and designed to teach fundamentals, but you can use it as a starting point for more complex systems.

We will go through this guide in steps, each building on the previous one but each step being a complete unit in itself - this will allow you to try and test at each step of the way, pause, and continue another time.

In the end, we'll have a small tremor application in which you can input text. It will capitalize and punctuate it and exit on command.

Runnable examples configuration for each of the steps is available [here](__GIT__/../code/basics).

## Topics

This guide introduces the following new concepts

* flows
* connectors
* pipelines
* scripts
* select with where
* `use` for functions, pipelines and connectors

## Passthrough

The simple configuration in tremor is what we call `passthrough` as it takes data from one end, passes it through, and puts it out the other end.

We handle the input and output with a `connector, `the processing or passing with a `pipeline`. Then we wrap the whole configuration in a flow.

### Flow

So let's start from the outside in and define a flow, we'll call this flow `main`, and it'll do nothing to start.

```troy
# Our main flow
define flow main
flow
end;
```

### Connectors

With that, let's start filling it. We will use `STDIN` and `STDOUT` for this example to read and write data. For that we have a #[`stdio` connector](../reference/connectors/stdio) that you could use. But since we know that it's a human reading and writing the text, we will instead use the pre-configured  `console` connector. You can find the `console` connector in the `troy::connectors` module of the standard library.

:::note
The console connector is a configured instance of the `stdio` connector that uses the `separate` pre and postprocessor to make it line-based and the [`string` codec](../reference/codecs/string) to avoid any parsing of the input data.
:::

```troy
# Our main flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;
  
  # create an instance of the console connector
  create connector console from connectors::console;
end;
```

### Pipeline

Now we have the connector that lets us receive and send data. Next, we need a pipeline to pass the data through.

We could write a custom pipeline, but since `passthrough` pipelines are somewhat common, we already have it defined in the `troy::pipelines` module, and we will be using that.

:::note
The definition of `troy::pipelines::passthrough` is just this:
```troy
define pipeline passthrough
pipeline
    select event from in into out;
end
```
:::

```troy
# Our main flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;
  # import the `troy::pipelines` module
  use troy::pipelines;

  # create an instance of the console connector
  create connector console from connectors::console;

  # create an instance of the passthrough pipeline
  create connector passthrough from pipelines::passthrough;
end;
```


### Wiring

We now have all the components we need, a flow to host it all, a connector to read and write data, and a pipeline to process (or pass) the data. With that all in place, we need to wire those parts up. We can do that with `connect` in the flow.

:::note
   Connect allows connecting to different ports on connectors and pipelines. When omitted, ports default to `/out`  for the connection's source and to `/in` for the connection's target. We will explore this more in a later step. For now, we'll ignore ports.
:::

```troy
# Our main flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;
  # import the `troy::pipelines` module
  use troy::pipelines;

  # create an instance of the console connector
  create connector console from connectors::console;

  # create an instance of the passthrough pipeline
  create pipeline passthrough from pipelines::passthrough;

  # connect the console (STDIN) to our pipeline input
  connect /connector/console to /pipeline/passthrough;

  # then connect the pipeline output to the console (STDOUT)
  connect /pipeline/passthrough to /connector/console;
end;
```

### Deploying the flow

We have already used pre-defined pipelines and connectors and created instances of them. Flows are not much different. They have a definition and an instantiation phase. While we call the instantiating of pipelines and connectors `create`, for flows, we use `deploy` as it is the step where the theoretical configuration becomes actual running code.

```troy
# Our main flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;
  # import the `troy::pipelines` module
  use troy::pipelines;

  # create an instance of the console connector
  create connector console from connectors::console;

  # create an instance of the passthrough pipeline
  create pipeline passthrough from pipelines::passthrough;

  # connect the console (STDIN) to our pipeline input
  connect /connector/console to /pipeline/passthrough;

  # then connect the pipeline output to the console (STDOUT)
  connect /pipeline/passthrough to /connector/console;

end;
# Deploy the flow, so tremor starts it
deploy flow main;
```

That's it, you can fetch this file from [git](__GIT__/../code/basics/passthrough.troy) and run it via:


```bash
$ tremor run passthrough.troy
hello
hello
world
world

```

## Transformation

We now have a way to pass data in our system, moving it through it and looking at the result. Our result is relatively simple, the same as the input we have. Let us change that and make the whole thing a bit more interesting.

Our goal will be to make each entry a "sentence" by capitalizing the first letter and adding a period `.` or question mark to the end.

:::note
Tremor has handy utility modules for most data types that provide several functions to work with them, the [reference documentation](../language/troy) gives an overview of them.
:::

### Defining our pipeline

We've been using the `troy::pipelines::passthrough` pipeline in the last step. It, as the name suggests, passes it through. So the first thing we need to do is replace this with our own. For simplicities sake, we'll start by replacing it with our pipeline. We will name this `main` as we will extend it to be more than a passthrough.

:::note
   Pipelines, by default, use the ports `in` for input, `out` and `err` for outputs. As with `connect`, those definitions can be omitted as long as we use the standard. For details on defining your own ports, you can refer to the [reference documentation](../language/troy).
:::

```troy
# Our main flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;

  # define our main pipeline
  define pipeline main
  pipeline
    select event from in into out;
  end;

  # create an instance of the console connector
  create connector console from connectors::console;

  # create an instance of the main pipeline
  create pipeline main;

  # connect the console (STDIN) to our pipeline input
  connect /connector/console to /pipeline/main;

  # then connect the pipeline output to the console (STDOUT)
  connect /pipeline/main to /connector/console;

end;
# Deploy the flow, so tremor starts it
deploy flow main;
```

### Transforming in the select body

Now we have our pipeline in which we will capitalize the text that's passed through the pipeline, [`std::string::capitalize`](../language/stdlib/std/string#capitalizeinput) will do that for us, and we can use it right in the select statement we:

:::note
In select statements, you can do any transformation that's creating new data, but you can't do any mutating manipulations. Simplified, you can think that `let` is not allowed.
:::

```troy
# Our main flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;

  # define our own main pipeline
  define pipeline main
  pipeline
    # use the `std::string` module
    use std::string;
    # Capitalize the string
    select string::capitalize(event) from in into out;
  end;

  # create an instance of the console connector
  create connector console from connectors::console;

  # create an instance of the main pipeline
  create pipeline main;

  # connect the console (STDIN) to our pipeline input
  connect /connector/console to /pipeline/main;

  # then connect the pipeline output to the console (STDOUT)
  connect /pipeline/main to /connector/console;

end;
# Deploy the flow, so tremor starts it
deploy flow main;
```


### Transforming in Scripts

Some transformations are a bit more complicated, and instead of the select body, we want a more elaborate script that makes the logic more readable.

:::note
The significant advantage of scripts is to allow more complex mutation, chained logic, and access to `state`. In this example, we'll not touch on all of them.
:::

So our goal will be to check if our sentence has punctuation. Otherwise, decide if we add a `.` or a `?`.

Scripts are a node in the pipeline, and we use `select` statements to connect them, the same way that we use `connect` to connect nodes of a flow.

```tremor
# Our passthrough flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;

  # define our main pipeline
  define pipeline main
  pipeline
    # use the `std::string` module
    use std::string;

    # define our script
    define script punctuate
    script
      # Short circuit if we already end with a punctuation
      match event of
        case ~re|[.?!]$| => emit event
        case _ => null
      end;

      # Find the right punctuation by looking at the first wird of the last sentence
      # yes this is a poor heuristic!
      let punctuation = match event of 
        case ~ re|.*[.?!]?(Why\|How\|Where\|Who)[^.?!]*$| => "?"
        case _ => "."
      end;
      event + punctuation
    end;

    # Create our script
    create script punctuate;

    # Wire our capitalized text to the script
    select string::capitalize(event) from in into punctuate;
    # Wire our script to the output
    select event from punctuate into out;
  end;

  # create an instance of the console connector
  create connector console from connectors::console;

  # create an instance of the main pipeline
  create pipeline main;

  # connect the console (STDIN) to our pipeline input
  connect /connector/console to /pipeline/main;

  # then connect the pipeline output to the console (STDOUT)
  connect /pipeline/main to /connector/console;

end;
# Deploy the flow, so tremor starts it
deploy flow main;
```


### Running

Same as before we can test our code, you can fetch the finished file from [git](__GIT__/../code/basics/transform.troy).

```bash
$ tremor run transfor.troy
hello
Hello.
why
Why?
```

## Filter

Aside from transformations, filtering is an integral part of what tremor can do to an event. We'll use this feature to allow users to type `exit` and have the application stop.

:::note
Stopping tremor is usually not something you'll want to do on a life server as it might impact other users, but it's a nice feature for an example like this.
:::

### Adding the `exit` connector.

We use the [`exit` connector](../reference/connectors/exit). This connector will stop the tremor instance on every event it receives.

We'll use a different port on the pipeline, the `exit` port, and wire this up to the `exit` connector.

We, however, stop short here from actually filtering just yet.

:::note
We can omit `in` and `out` as ports as that's what tremor defaults to. For `exit`, we have to be more specific.
:::

```troy
# Our main flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;

  # define our main pipeline
  define pipeline main
  pipeline
    # use the `std::string` module
    use std::string;

    # define our script
    define script punctuate
    script
      # Short circuit if we already end with a punctuation
      match event of
        case ~re|[.?!]$| => emit event
        case _ => null
      end;

      # Find the right punctuation by looking at the first wird of the last sentence
      # yes this is a poor heuristic!
      let punctuation = match event of 
        case ~ re|.*[.?!]?(Why\|How\|Where\|Who)[^.?!]*$| => "?"
        case _ => "."
      end;
      event + punctuation
    end;

    # Create our script
    create script punctuate;

    # Wire our capitailized text to the script
    select string::capitalize(event) from in into punctuate;
    # Wire our script to the output
    select event from punctuate into out;
  end;
  # Define the exit connector
  define connector exit from exit;

  # create the exit connector;
  create connector exit;

  # create an instance of the console connector
  create connector console from connectors::console;

  # create an instance of the passthrough pipeline
  create pipeline main;

  # connect the console (STDIN) to our pipeline input
  connect /connector/console to /pipeline/main;

  # then connect the pipeline output to the console (STDOUT)
  connect /pipeline/main to /connector/console;

  # connect the `exit` port of our pipeline to the exit connector
  connect /pipeline/main/exit to /connector/exit;

end;
# Deploy the flow, so tremor starts it
deploy flow main;
```

### filtering out exit messages

The last thing left to do is filter out messages that read `exit` and forward them to the `exit` port instead of out.

We need to add the `exit` port to the `into` part of the `pipeline` definition, so it becomes available.

:::hint
For pipelines, we can omit `into` and `from`. If done, it is treated as `into out, err` and `from in`. Replacing one will **not add** to these ports but replace them, so if we want `out` and `exit` as ports, we have to write `into out, exit`.

`from` and `into` are independent. Overwriting one does not affect the other.
:::

Once that is done, we can add the filter logic. To do that, we create a new select statement with a `where event == "exit"` clause that will only forward events that read `exit` and add a `where event != "exit"` to our existing clause, forwarding the rest to the punctuate script.

```troy
# Our main flow
define flow main
flow
  # import the `troy::connectors` module
  use troy::connectors;

  # define our main pipeline
  define pipeline main
  # the exit port is not a default port, so we have to overwrite the built-in port selection
  into out, exit
  pipeline
    # use the `std::string` module
    use std::string;

    # define our script
    define script punctuate
    script
      # Short circuit if we already end with a punctuation
      match event of
        case ~re|[.?!]$| => emit event
        case _ => null
      end;

      # Find the right punctuation by looking at the first wird of the last sentence
      # yes this is a poor heuristic!
      let punctuation = match event of 
        case ~ re|.*[.?!]?(Why\|How\|Where\|Who)[^.?!]*$| => "?"
        case _ => "."
      end;
      event + punctuation
    end;

    # Create our script
    create script punctuate;

    # filter eany event that just is `"exit"` and send it to the exit port
    select {"graceful": false} from in where event == "exit" into exit;

    # Wire our capitailized text to the script
    select string::capitalize(event) from in where event != "exit" into punctuate;
    # Wire our script to the output
    select event from punctuate into out;
  end;
  # Define the exit connector
  define connector exit from exit;

  # create the exit connector;
  create connector exit;

  # create an instance of the console connector
  create connector console from connectors::console;

  # create an instance of the passthrough pipeline
  create pipeline main;

  # connect the console (STDIN) to our pipeline input
  connect /connector/console to /pipeline/main;

  # then connect the pipeline output to the console (STDOUT)
  connect /pipeline/main to /connector/console;

  # connect the `exit` port of our pipeline to the exit connector
  connect /pipeline/main/exit to /connector/exit;

end;
# Deploy the flow, so tremor starts it
deploy flow main;
```

### Running

That all set, we can run our script as before, just this time, when entering `exit` tremor will terminate.

```bash
$ tremor run transfor.troy
hello
Hello.
why
Why?
exit
```