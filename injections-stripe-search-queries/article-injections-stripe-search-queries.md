![A light watercolor painting of injecting into a search box](images/DALL%C2%B7E%202023-05-22%2014.41.14%20-%20Continue%20the%20background%20.png)

# Stripe Search Query Injections and How to Prevent Them

If you directly use user input in a [Stripe search query](https://stripe.com/docs/search), it is vulnerable to injections. Attackers can exploit this to gain read access to all records of your Stripe resource. The principle is basically the same as in SQL injections. In this article, I propose a fix.

## Example

``` javascript
// NodeJS
const stripe = require("stripe")(process.env.STRIPE_SECRET_KEY)

const userInput = "124' OR created>0 OR status:'active" // Injection

let subscriptions = await stripe.subscriptions.search({
    query: `metadata['myField']: '${userInput}'`
})
console.log(subscriptions) // all subscriptions ever
```

## Background

I was developing a backend that integrates [Stripe](https://stripe.com) to manage customers and to process subscriptions. In one use case, it should search for user input in the metadata of Stripe subscriptions.
When I looked up the documentation about how to use the search functionality I came across the [Search Query Language](https://stripe.com/docs/search#search-query-language). It looked like users could easily inject their own additional query clauses. After a simple check: Yes, e.g. for the user input `"124' OR status:'active"` all active subscriptions are returned as well. 

I searched for `escape` and `injection` in Stripe documentation and asked Google and ChatGPT - no results.
The Stripe support informed me that there is currently no functionality for this type of validation in the Stripe NodeJS library or API. This is something developers would have to build on their own. 
As a developer, I would expect to find this statement in the documentation.

Is this a problem?
A quick search on GitHub for the usage of Stripe search query showed: 
4 out of 6 projects I looked at do not implement any input escape or input validation functionality. Therefore, all these projects are potentially vulnerable.

## Solution

I wrote a small NodeJS library [stripe-escape-input](https://www.npmjs.com/package/stripe-escape-input) that does the job.

```shell
npm i stripe-escape-input
```

``` javascript
const escapeInput = require("stripe-escape-input")
const stripe = require("stripe")(process.env.STRIPE_SECRET_KEY)

const userInput = "124' OR created>0 OR status:'active" // Injection

let subscriptions = await stripe.subscriptions.search({
    query: `metadata['myField']: '${escapeInput(userInput)}'`
})
console.log(subscriptions) // 0 subscriptions
```

To prevent injections, we need to escape the user input before using it in a Stripe search query. What exactly has to be considered tells us the [Stripe query language documentation](https://stripe.com/docs/search#search-query-language). Let's hope it is complete.
Based on that, it comes down to string replacement of single quotes `'`, double quotes `"`, and backslashes `\`, so an attacker can not end the value part and add additional query clauses.

## Final Thoughts

Is this a good solution? No. As an external developer, I can only implement what is documented by Stripe. There might exist undocumented behavior that could lead to new vulnerabilities. The best approach would be for Stripe to provide this functionality since they know all the implementation details of their search queries. Until then, this is the best we can achieve.

## References
- https://stripe.com/docs/search
- https://stripe.com/docs/api/subscriptions/search
- https://github.com/soerenmetje/stripe-escape-input
- https://www.npmjs.com/package/stripe-escape-input
