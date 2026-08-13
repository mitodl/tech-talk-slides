---
title: "ECommerce and Payment Gateway"
slides:
  theme: parchment.css
revealjs:
  width: 1920
  height: 1080
  margin: 0.02
  center: false
  pdfSeparateFragments: false
---

## An Overview Of
# ECommerce and PaymentGateway

---

# The Basics

#### Ecommerce Feature Set

* Standard common ecommerce features: products, shopping cart, checkout, receipts, discounts, and refunds <!-- .element: class="fragment" data-fragment-index="1" -->
* Automated fulfillment: once you check out, you get a certificate track enrollment <!-- .element: class="fragment" data-fragment-index="2" -->
* Plus a few specialty features: <!-- .element: class="fragment" data-fragment-index="3" -->
    * Financial assistance - offering lower price points for certificates for eligible learners; built using the discount subsystem <!-- .element: class="fragment" data-fragment-index="4" -->
    * B2B gating - access to contracts and enrollments are gated through the existing ecommerce workflows, also built on top of discounts <!-- .element: class="fragment" data-fragment-index="5" -->

Notes:
- Goal is to allow learners to earn a certificate - at the end of the day, that's what we're selling and ecommerce is geared towards that
- B2B is a much larger system that uses ecommerce, so will likely have a separate talk about that

---

# Typical Purchasing Workflow

For a typical learner:

- They find a course or program they’re interested in that has a certificate available <!-- .element: class="fragment" data-fragment-index="1" -->
- They add to the cart <!-- .element: class="fragment" data-fragment-index="2" -->
- They go to their cart to verify what they’re about to get <!-- .element: class="fragment" data-fragment-index="3" -->
    - Maybe they add a discount code here, if they have one
    - We may also automatically apply a discount if there’s one to apply (FA or discount rules)
- They click the Place Order button and are sent off-site to complete payment <!-- .element: class="fragment" data-fragment-index="4" -->
- They return to the app and we fulfill their order (maybe asynchronously) <!-- .element: class="fragment" data-fragment-index="5" -->

Notes:

---

# After Purchase

The app fulfills the order by updating their enrollment.

- Maybe later, the learner wants to request a refund <!-- .element: class="fragment" data-fragment-index="1" -->
    - They contact Customer Service, and CS tells them to fill out a Google Form (which drops into a Google Sheet) <!-- .element: class="fragment" data-fragment-index="2" -->
    - Someone processes their form submission and maybe approves it <!-- .element: class="fragment" data-fragment-index="3" -->
    - The app checks the Google Sheet - any new approved submissions kick off a refund <!-- .element: class="fragment" data-fragment-index="4" -->
    - If successful, we downgrade the enrollment (if we can) <!-- .element: class="fragment" data-fragment-index="5" -->

Notes:
- Adding the item to the cart (usually) gives them an audit enrollment, so we just upgrade that enrollment
- We do have a return policy (but for now you still have to talk to Customer Service)
- A self-serve refund process is in the works

--- 

# In Between "Place Order" and Fulfillment

<div class="fragment" data-fragment-index="1">

We use a payment processor that supports hosted checkout
* Hosted checkout means the payment processor presents the actual "checkout" page, and gathers the learner's payment and contact information
* We never see the payment data (card numbers, etc.) - reduces our data security footprint by a lot and avoid PCI compliance hassles
* We've used CyberSource for a number of years but now onboarding Stripe

</div>

<div class="fragment" data-fragment-index="2">

We have to prepare things to send the learner to the hosted checkout
* This can be different things - CyberSource (and others) work with a form, Stripe (and others) make you set up state via API call and provide a link to redirect the user to
* Cart data generally has to be reformatted to match the destination system too

</div>

<div class="fragment" data-fragment-index="3">

MITx Online uses an ol-django app - `payment_gateway` - to manage this stuff
* Provides a standardized interface for cart data and stand up/tear down of the processor's libraries
* Wraps the steps to perform checkout so it's consistent between processors

</div>

---

# PaymentGateway Internals

PaymentGateway provides an interface - for both the app and for implementing processor support - and a simple data model to handle data collection for orders.

* PaymentGateway class is the abstracted interface <!-- .element: class="fragment" data-fragment-index="1" -->
  * Uses Abstract Base Classes to provide a single class interface to any available gateway
  * A set of `abstractmethod`s provides the processor-specific surface that the individual gateways implement
  * Another set of methods is used by the application
    * These are wrapped in a helper - `find_gateway_class` - that allows the app to specify what gateway to use
    * Works per-call, so can use any gateway at any time (within reason)
* Simple data model for the cart <!-- .element: class="fragment" data-fragment-index="2" -->
  * Each processor wants the order information in different formats
  * So does each app that may use this library
  * The PaymentGateway `Order` and `CartItem` dataclasses help with transitioning between formats

---

# Sample Checkout

Checking out looks like this:

```python
from mitol.payment_gateway import api

order = api.Order(
    username="a username",
    reference="mitxonline-prod-999",
    ...
)
order.items = [
    api.CartItem(code=item.code, name=item.name...) for item in cart.items
]
result = api.PaymentGateway.start_payment("CyberSource", order, ...)
```

<div class="fragment">

The `result` you get back tells you what action to present to the user - either an auto-submitting form, or a redirect to a URL.

In either case, the learner goes off to enter their billing and payment information, and eventually (hopefully) returns to the app.

</div>

Notes:
- Not shown but there'd be a cart or basket object of some sort already loaded before this code gets executed. This is app-dependent.
- In `start_payment`, the first argument is the gateway class to use. If we wanted to use Stripe or something else instead, we'd update that argument and make whatever changes we need to support the result that comes back.
- CyberSource requires the form method - we have to build a form and submit it to CyberSource. Stripe gives you a URL and you redirect the user.

---

# Post-Checkout Events

When the learner finishes payment, the app gets notified by the payment processor so it can fulfill the order. There are two ways this can happen.

* Via webhook: the payment processor hits a webhook URL in the app to send event notifications <!-- .element: class="fragment" data-fragment-index="1" -->
* With the learner: the payment processor sends the learner back to the app with data attached (via a POST request)  <!-- .element: class="fragment" data-fragment-index="2" -->

Both Stripe and CyberSource use webhooks. CyberSource also sends data back with the learner when they're done. The data we get back from the payment processor is signed and must be validated. <!-- .element: class="fragment" data-fragment-index="3" -->

<div class="fragment" data-fragment-index="4">

* PaymentGateway includes some methods to check this signature: `validate_processor_response` and `get_formatted_response`.
* It doesn't do much processing of the data (at this point, just enough to determine success/failure).

</div>

Notes:
- The return-with-data method is fragile - the user doesn't always make it back to the app for whatever reason, and it doesn't work if the transaction ends up in a state that doesn't immediately resolve to a pass/fail. 
- Stripe uses webhooks extensively - any data that's not explicitly requested via API call comes back to the app via a webhook.
- PaymentGateway doesn't handle webhooks directly since the logic is very app-dependent.

---

# Refunds

If we grant a learner a refund for their purchase, we start that process from within the app.

* PaymentGateway has an interface for this - `get_refund_request` and `start_refund` - that splits the task into two 
    * `get_refund_request` reshapes the transaction data we received into a more useful form
    * `start_refund` submits the request to the payment processor
* If the refund is successful, we (usually) downgrade the enrollment

<div class="fragment" data-fragment-index="3">

Code sample:

```python
# order from a few slides ago..

refund = api.PaymentGateway.get_refund_request("CyberSource", order.transaction)
result = api.PaymentGateway.start_refund("CyberSource", refund)

```

</div>

Notes:
- For CyberSource, this actually uses a totally separate API. PaymentGateway handles that internally.
- Stripe will return a result but if the refund isn't immediately successful, it will hit the webhook.
- This is part of the reason PG doesn't process the data from webhooks/etc. much - it can have data in it that may be necessary for other purposes.

---

# Benefits and Limitations

There's some benefits to this route, and some limitations.

* Benefits <!-- .element: class="fragment" data-fragment-index="1" -->
  * Makes it (relatively) easy to add support for new processors, run multiple processors, etc. <!-- .element: class="fragment" data-fragment-index="2" -->
    * MITx Online can now <small>(technically)</small> use Stripe for payment processing
    * It can do this alongside CyberSource
    * Most of the changes needed was adding a more robust webhook handler 
  * PaymentGateway abstracts away (some) weirdness, and makes it easier to do certain operations <!-- .element: class="fragment" data-fragment-index="3" -->
    * Example: Refunds in CyberSource use a totally different API from payments
    * Just one set of API calls in the same PaymentGateway interface for that
* Limitations <!-- .element: class="fragment" data-fragment-index="4" -->
  * PaymentGateway is a light wrapper - some further processing in-app needs to be aware of the payment processor <!-- .element: class="fragment" data-fragment-index="5" -->
    * Received webhook and transaction data is mostly left as-is
  * Only covers a pretty simple set of use cases - basic checkout and full refunds <!-- .element: class="fragment" data-fragment-index="6" -->

Notes:
- Light processing is an advantage sometimes (see refunds)
- We can pretty easily add whatever new stuff we need

---

## The End Of An Overview Of
# ECommerce and PaymentGateway

### Thanks for listening!

Links to resources:

- Stripe API Reference: https://docs.stripe.com/api
- App in ol-django: https://github.com/mitodl/ol-django/tree/main/src/payment_gateway
- MITx Online implementation PR: https://github.com/mitodl/mitxonline/pull/3751
