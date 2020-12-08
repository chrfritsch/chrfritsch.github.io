---
layout:     post
title:      Mock HTTP requests in Drupal functional tests
date:       2020-12-08
tags:       [tests, guzzle]
---

When you are writing functional tests in Drupal (BrowserTestBase or FunctionalJavascript), you sometimes execute some code that makes external requests.
Because of tests should be stable and predictable, they shouldn't depend on external services. 
So this article will describe how we can mock all external requests.

First let's make a little test module:

*test_mock_request.info.yml*
{% highlight yaml %}
name: Test mock request
description: 'Mock requests in tests.'
type: module
package: Testing
version: VERSION
{% endhighlight %}

The trick is to register our own guzzle middleware. You can learn more about middlewares [here](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html). To register our middleware we need a services.yml file.

*test_mock_request.services.yml:*
{% highlight yaml %}
services:
  test_mock_request.http_client.middleware:
    class: Drupal\test_mock_request\MockHttpClientMiddleware
    arguments: ['@request_stack', '@state']
    tags:
      - { name: http_client_middleware }

{% endhighlight %}

And our middleware.

*src/MockHttpClientMiddleware.php*

{% highlight php %}
<?php

namespace Drupal\test_mock_request;

use Drupal\Core\State\StateInterface;
use GuzzleHttp\Psr7\Response;
use function GuzzleHttp\Promise\promise_for;
use Psr\Http\Message\RequestInterface;
use Symfony\Component\HttpFoundation\RequestStack;

/**
 * Sets the mocked responses.
 */
class MockHttpClientMiddleware {

  /**
   * The request object.
   *
   * @var \Symfony\Component\HttpFoundation\Request
   */
  protected $request;

  /**
   * The state service.
   *
   * @var \Drupal\Core\State\StateInterface
   */
  protected $state;

  /**
   * MockHttpClientMiddleware constructor.
   *
   * @param \Symfony\Component\HttpFoundation\RequestStack $requestStack
   *   The current request stack.
   * @param \Drupal\Core\State\StateInterface $state
   *   The state service.
   */
  public function __construct(RequestStack $requestStack, StateInterface $state) {
    $this->request = $requestStack->getCurrentRequest();
    $this->state = $state;
  }

  /**
   * Add a mocked response.
   *
   * @param string $url
   *   URL of the request.
   * @param string $body
   *   The content body of the response.
   * @param array $headers
   *   The response headers.
   * @param int $status
   *   The response status code.
   */
  public static function addUrlResponse($url, $body, array $headers = [], $status = 200) {

    $items = \Drupal::state()->get(static::class, []);
    $items[$url] = ['body' => $body, 'headers' => $headers, 'status' => $status];

    \Drupal::state()->set(static::class, $items);
  }

  /**
   * {@inheritdoc}
   *
   * HTTP middleware that adds the next mocked response.
   */
  public function __invoke() {
    return function ($handler) {
      return function (RequestInterface $request, array $options) use ($handler) {
        $items = $this->state->get(static::class, []);
        $url = (string) $request->getUri();
        if (!empty($items[$url])) {
          $response = new Response($items[$url]['status'], $items[$url]['headers'], $items[$url]['body']);
          // @phpstan-ignore-next-line
          return promise_for($response);
        }
        elseif (strstr($this->request->getHttpHost(), $request->getUri()->getHost()) === FALSE) {
          throw new \Exception(sprintf("No response for %s defined. See MockHttpClientMiddleware::addUrlResponse().", $url));
        }

        return $handler($request, $options);
      };
    };
  }
}
{% endhighlight %}
 
This is pretty cool. The middleware runs before every request is executed. So at that time we can decide:
 * Is for the current request URL a mock response defined, return it.
 * Is this a local request, pass it by.
 * Otherwise throw an exception.

In our tests we just call *addUrlResponse()* to add the mocked responses.
{% highlight php %}
   MockHttpClientMiddleware::addUrlResponse('https://example.com/file.json', '{ "foo": "bar"}', ['Content-Type' => 'application/json']);
{% endhighlight %}

Activate this functionality by adding your test module to the *$modules* property of your test class.

We added this functionality very recently to our [Thunder tests](https://github.com/thunder/thunder-distribution/tree/6.1.x/tests/modules/thunder_test_mock_request) to make them more stable.