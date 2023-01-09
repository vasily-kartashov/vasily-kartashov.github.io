---
layout: post
title: PHP Command Resource
tags: php symfony
---

There is an obvious similarity between web resources, command-line interface (CLI) commands, and Alexa resources. Web resources transform HTTP requests into HTTP responses, CLI commands accept user input and trigger business logic while occasionally reporting back to the user, and Alexa resources encode parsed voice commands and report back a JSON-formatted response. In fact, there is probably a 90% overlap between these three types of resources. Here is a short note on how to use this overlap to your advantage:

First, it is important to note that in the process of transforming input into output, we will go through multiple stages, and these stages are largely disjoint. An HTTP request must conform to a specific web service specification, an Alexa request must conform to its own specification, and a CLI request must conform to the flavor of command line utilities you prefer.


                                                    +---------+
    HTTP Request  -> Authorization -> Validation -> |         | -> HTTP Formatting  -> Output
    Alexa Request -> Authorization -> Dispatch   -> |         | -> Alexa Formatting -> Output
    CLI Request -> Validation                    -> |         | -> Output
                                                    +---------+

To reuse code across these different types of resources, the interface should be the common denominator, and the output should be understandable by all possible clients of this interface. One possible interface is the Symfony\Component\Console\Command\Command, which can be extended to operate on the psr/http-message interfaces as well. For example:

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

To extend the `Symfony\Component\Console\Command\Command` interface to work with `psr/http-message` interfaces, we can add the following method:

```
protected function executeWebRequest(RequestInterface $request, ResponseInterface $response)
{
    $input = new ArrayInput([
        '--controller-id' => $request->getParameter('controllerId')
    ], $this->getDefinition());

    $this->execute($intput, $output, $response);
}
```

We can also add a method for handling Alexa requests:

```
protected function executeAlexaRequest(AlexaRequest $request, AlexaResponse $response)
{
    if ($request instanceof IntentRequest) {
        if ($request->intentName == 'execute') {
            $intput = new ArrayInput([
                '--controller-id' => $request->getSlot('controllerId')
            ], $this->getDefinition());
        }
        ...
    }

    $this->execute($input, $output, null, $response);
}
```

With these methods, we can reuse the same class from web, Amazon Alexa, and CLI. It is convenient that PHP allows us to "implement" the `execute` method while adding parameters that default to `null`.
