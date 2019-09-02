# Module 12: Capture and forward correlation ID

## Capture and forward correlation IDs

Since we're using the `@dazn/lambda-powertools-pattern-basic` and `@dazn/lambda-powertools-logger`, we are already capturing incoming correlation IDs and including them in the logs.

However, we need to make sure the `get-index` function forwards the correlation IDs along, while still using the X-Ray instrumented `https` module.

The `dazn-lambda-powertools` project also has Kinesis and SNS clients that will automatically forward captured correlation IDs. We can use them as stand-in replacements for the Kinesis and SNS clients from the AWS SDK.

<details>
<summary><b>Forward correlation IDs from get-index</b></summary><p>

You can access the auto-captured correlation IDs using the `@dazn/lambda-powertools-correlation-ids` package. From here, we can include them as HTTP headers.

1. At the project root, run `npm install --save @dazn/lambda-powertools-correlation-ids`.

2. Open `functions/get-index.js` and require the `@dazn/lambda-powertools-correlation-ids` module (at the top of the file).

```javascript
const CorrelationIds = require('@dazn/lambda-powertools-correlation-ids')
```

3. Staying in `functions/get-index.js`, replace the `getRestaurants` function with the following

```javascript
const getRestaurants = () => {
  const { hostname, pathname } = URL.parse(restaurantsApiRoot)

  return new Promise((resolve, reject) => {
    const options = {
      hostname: hostname,
      port: 443,
      path: pathname,
      method: 'GET',
      headers: Object.assign({}, CorrelationIds.get())
    }

    const req = https.request(options, res => {
      res.on('data', buffer => {
        const body = buffer.toString('utf8')
        resolve(JSON.parse(body))
      })
    })

    req.on('error', err => reject(err))

    req.end()
  })
}
```

These changes are enough to ensure correlation IDs are included in the HTTP headers in the request to the `GET /restaurants` endpoint.

4. Deploy the project.

5. Once the deployment is done, load the page. And then open the X-Ray console to make sure that the X-Ray tracing is still working.

6. Open the CloudWatch console to check the logs for both `get-index` and `get-restaurants`. You should see that the same correlation ID is included in both logs.

</p></details>

<details>
<summary><b>Forward correlation IDs through Kinesis events</b></summary><p>

As an exercise for you to carry out on your own after the workshop. See if you can get correlation IDs flowing through the Kinesis events as well.

Familiarize yourself with the various powertools in the dazn-lambda-powertools [repo](https://github.com/getndazn/dazn-lambda-powertools). You will need:

* the [Kinesis client](https://github.com/getndazn/dazn-lambda-powertools/tree/master/packages/lambda-powertools-kinesis-client) as a stand-in replacement for the AWS SDK's Kinesis client. This client would auto-forward any captured correlation IDs along.

* for batched event sources (Kinesis, Firehose and SQS), the [correlation-IDs middleware](https://github.com/getndazn/dazn-lambda-powertools/tree/master/packages/lambda-powertools-middleware-correlation-ids), which is applied through the `wrap` function we used to wrap our handlers, requires you to change your processor code slightly. Read the [relevant section](https://github.com/getndazn/dazn-lambda-powertools/tree/master/packages/lambda-powertools-middleware-correlation-ids#kinesis) of the README and change the handler code accordingly.

</p></details>