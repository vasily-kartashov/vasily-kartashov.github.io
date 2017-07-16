---
layout: post
title: PHP Command Resource
---

There's a striking similarity between

- web resources, i.e. things that transform an http request into an http response,
- cli commands that accept user input and trigger some business logic occasionally reporting back to the user, and
- alexa resources, that encode parsed voice commands and report back a JSON formatted bowl of spagghetti

In fact there's probably 90% overlap. So here's a short note about how to use this overlap to your advantage.

First thing to note, is that in a process of transforming input into output we'll be going through multiple stages, and these stages are largely disjoint. HTTP request will need to conform to your web service spec, Alexa it's own spec, and CLI to whatever flavour of command line utilities you prefer. Here's a short diagram:

                                                    +------------+
    HTTP Request  -> Authorization -> Validation -> |            | -> HTTP Formatting  -> Output
    Alexa Request -> Authorization -> Dispatch   -> |            | -> Alexa Formatting -> Output
    CLI Request -> Validation                    -> |            | -> Output
                                                    +------------+

So the questionof reuse boils down to this one box in the middle. The interface of the box should be the common denominator and output should be understandable by all possible clients of this box.

The easiest interface one can think of is the `Symfony\Component\Console\Command\Command` as in

```
protected function execute(
    InputInterface $input,
    OutputInterface $output,
    ResponseInterface $webResponse = null,
    AlexaResponse $alexaResponse = null) {

    $controllerId = $input->getOption('controller-id');
    ...
}
```

We can extend it a bit with the following method operating on `psr/http-message` interfaces. For example as follows:

```
protected function executeWebRequest(RequestInterface $request, ResponseInterface $response)
{
    $input = new ArrayInput([
        '--controller-id' => $request->getParameter('controllerId')
    ], $this->getDefinition());

    $this->execute($intput, $output, $response);
}
```

And one more method for Alexa:

```
protected function executeAlexaRequest(AlexaRequest $request, AlexaResponse $response)
{
    if ($request instanceof IntentRequest) {
        if ($request->intentName == 'execute') {
            $intput = new ArrayInput([
                '--controller-id' => $request->getSlot('controllerId')
            ])
        }
        ...
    }

    $this->execute($input, $output, null, $response);
}
```

With this we now have one class that can be reused from web, Amazon Alexa and CLI. It's actually not bad, that PHP lets us to "implement" the `execute` method while addng parameters default to `null`.
