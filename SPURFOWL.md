# SPURFOWL

Sandboxed Private User Reporting Functions Operating Within Limits

See `SPURFLOW_slides.pdf` in this repository for the slides used in 2020-12-08
web-adv meeting.

## Authors

* Mikko Juola (NextRoll)
* Joey Robert (NextRoll)

## Stakeholder feedback / Opposition

| Stakeholder | Sentiment | Discussion |
|--|--|--|
| NextRoll | In favor | This document

##  Introduction

Advertising reporting today works by storing users' event data outside the
browser and then using the third party cookies to figure out which events
belong to the same person. One of the most prominent use cases of this is
measuring the performance of ad campaigns.

In light of third party cookie being deprecated and eventually blocked in major
browser vendors, alternative, more privacy-focused approaches need to be
considered.

This proposal suggests two mechanisms to be added into the browser: a "trail
store" and "private sandboxed functions". These mechanisms make it possible to
implement complicated reporting in a privacy-preserving way. 

## Scope

This proposal is about technologies that would be implemented into the browser,
as opposed to server-side infrastructure. The purpose of this proposal is to
offer a path for doing reporting and measurement in a privacy-preserving way.
We assume the existence of some form of MPC-based aggregation on server-side, such
as explained in [Conversion Measurement API Multi-Browser Aggregation Service
Explainer](https://github.com/WICG/conversion-measurement-api/blob/master/SERVICE.md).

## Goals

Key goals:

* Personal data stays in the user agent. This is not shared to any third parties as-is.
* Allow interested parties (DSPs, SSPs, publishers, third-party reporting
  frameworks, etc.) to compute non-trivial reporting.
* User agent does not need to have high level of trust in entities that run
  the server-side infrastructure (we assume secure MPC-type aggregation to get this)

## Motivation

On a technical level, a significant portion of useful reporting is relying on
the ability to track users through third-party cookies. SPURFOWL is an attempt
to present a fairly general mechanism to continue to do reporting that still
relies on tracking users but any sensitive information about that tracking
stays in the browser.

Anything to do with tracking impressions is useful, e.g.:

  * Attribution models; in particular multi-touch attribution models
  * Funnel: how many users saw impressions, how many of those users came to the
    website, how that compares with organic traffic etc.
  * Which ad campaigns drive more traffic, how do users targeted with different
    ad campaigns behave when they arrive to the website.

Some additional motivation can be found in other proposals such as:

  * [SPARROW reporting](https://github.com/WICG/sparrow/blob/master/Reporting_in_SPARROW.md#how-it-relates-to-some-common-use-cases-that-require-reporting-capabilities)
  * [PELICAN](https://github.com/neustar/pelican)

We hope that SPURFOWL can start a discussion on what browser-side features are
needed to support privacy-preserving measurement.

## SPURFOWL summary

Here is what SPURFOWL proposes:

  * A "trail store" that stores events happening inside the browser.

  * Private, sandboxed JavaScript functions that compute reports.

These components would be operated through a new Web API.

### The trail store

We propose that browser shall hold a list of events, i.e. the "trail store".
Here's an illustrating example what might be stored:

```
+------------+----------------+------------+------------+-----------+
| type       |   timestamp    |  domain    | user agent |   adv     |
+------------+----------------+------------+------------+-----------+
| impression   15:00:00 25/11    news.com     chrome      shoes.com |
| impression   15:00:02 25/11    news.com     chrome      shoes.com |
| impression   16:30:15 25/11    blah.com     chrome      shoes.com |
| click        16:30:20 25/11    blah.com     chrome      shoes.com |
| pageview     16:30:22 25/11    shoes.com    chrome      shoes.com |
| pageview     16:30:22 25/11    shoes.com    chrome      shoes.com |
| pageview     16:30:22 25/11    shoes.com    chrome      shoes.com |
| conversion   16:30:22 25/11    shoes.com    chrome      shoes.com |
+-------------------------------------------------------------------+
```

An interested party can populate the trail store by using a JavaScript Web API
or through special `<a>` tags.

### Private sandboxed functions

If we allowed just any JavaScript code to read from the trail store, it could
just send the trails directly to reporting origin and defeat all privacy.

Instead of allowing that, we propose:

  * The trail store is write-only with one exception:
  * The trail store becomes readable inside _private sandboxed functions_

Example scenario: reporting agency can present a `reporting.js` code to the user agent.
This code will be executed in a special sandboxed environment. The sandbox will
prevent the code from doing any persistent changes to the browser, and will
prevent it from making any network calls. The sandbox will also limit CPU
time, memory use and size of output.

It is in this sandboxed environment that the trail store becomes readable.

The sandboxed reporting code computes all the metrics locally in the browser,
computing all the metrics that the reporting agency cares about. The results of
that computation is sent out encrypted to aggregation infrastructure (assumed to be secure MPC-type aggregation infrastructure).

### Trust model

The browser should only trust scripts coming from the same domain as the site
it is currently visiting, i.e. same-origin policy.

For example: let's say browser is visiting `shoes.com`. Any attempt to populate
trail store by parties other than `shoes.com` will be rejected. In theory, this
means `shoes.com` could just implement all reporting on its own.

In practice, `shoes.com` would likely want contract a third party to do
reporting for them. In this case, `shoes.com` can tell the browser
"`measurement.example.com` is allowed to access and do reporting for my data".

### Justification

Combining the trail store and private functions allows interested parties to
compute interesting and useful metrics without having to limit themselves to
some weak computational model. The trail store can contain highly sensitive
data, but with a secure aggregation infrastructure, the user agent can be
confident that sensitive information cannot leak out to third parties.

In other words, this proposal balances the desire for user privacy while at the
same time presenting a generic framework for doing reporting in the browser.

For example, this allows reporting agencies to implement whatever conversion
models they want. Or they can implement other kinds of measuring: funnel
reporting, multi-channel attribution, shopping cart metrics etc. SPURFOWL is a
mechanism for doing reporting in a private way and it is up to reporting
frameworks to decide what exactly to do with it.

## SPURFOWL details

The rest of this document is an imagination how the trail store and private
functions might actually look and feel like.

### Trail store

Trail store is a list of events, that are stored in the browser.

The events are namespaced by reporting origin.

From JavaScript, the following web API pushes an event to the trail store:

```
window.trailStore.push(
    <reporting origin>,
    <JavaScript object>)
```

Example API use:

```
window.trailStore.push(
    'dsp.example.com',
    { 'advertiser':       'we-like-shoes.com',
      'type':             'conversion',
      'path':             '/checkout',
      'conversion-value': 123.45,
      'country':          'US' })
```

The user agent should set a limit how many events can be stored per reporting
agency and how many events are stored overall. When pushing a new event to
trail store which has reached the limit, drop the oldest event.

Following same-origin policy, pushes to trail store should be rejected unless
the origin (i.e. the web page the browser is currently visiting) has allowed
it.

We would be pushing events to trail store when:

*  Browsing the advertiser's site.
*  When rendering an impression.
*  When user clicks an impression.

A typical scenario might be that the DSP, on behalf of an advertiser, populates
the trail store while a user is browsing the advertiser's site. When rendering
ads, we can have the SSP run JavaScript or use special tags on an `<a>` element
to populate the trail store.

### Click tags

Inspired by the [Conversion Measurement
API](https://github.com/WICG/conversion-measurement-api), we propose that
clicks can be stored inside the trail store through special tags on `<a>`
element.

```
<a
  reportingorigin="dsp.example.com"
  conversiondestination="advertiser.example.com"
  data="we-like-shoes.com:ad1"
  ...
/>
```

Clicking on the `<a>` tag can have the effect of:

```javascript
window.trailStore.push(
  'dsp.example.com',
  { 'type': 'click',
    'conversiondestination': 'advertiser.example.com',
    'data': 'we-like-shoes.com:ad1' })`
```

For impressions (i.e. ads that were not clicked), we can populate the
trail store normally through the JavaScript API.

### Private sandboxed functions

Private sandboxed function are JavaScript functions but they run in a special environment. In this environment:

*  No network calls to outside world are allowed

*  No making any persistent changes to browser state

*  Limits on CPU time, memory, and code size.

*  No access to Web APIs other than the trail store.

### Private functions for reporting

Reporting agencies can store their reporting code at some location:

```
https://dsp.example.com/.well-known/reporting.js
```

This file should contain one function called `trailStoreMap()` that will do
whatever reporting computations the reporting agency is interested in.

### Code details

`reporting.js` is a JavaScript module containing one function:

`trailStoreMap(<trail-store-for-reporting-origin>)`: Map has full access
to the `trailStore` list for that Reporting Origin. This code is
executed in the **user agent** in sandboxed execution mode.

Here is a simple example of `trailStoreMap` that counts page views and conversion events. (Real-world code would likely be much more complicated).

This code produces two reports: `page_views` and `conversions`.

```javascript
function trailStoreMap(trail_store) {
    var trail_length = trail_store.length();
    var page_views = 0;
    var conversions = 0;
    
    for (var i = 0; i < trail_length; ++i) {
        // This returns the same objects as previously pushed with
        // window.trailStore.push() or through <a> tags.
        var event = trail_store.get(i);
        if (event['type'] === 'page_view') {
            page_views++;
        }
        if (event['type'] === 'conversion') {
            conversions++;
        }
    }

    return {'page_views': {'value': page_views},
            'conversions': {'value': conversions}}
}
```

### Scheduling of reports

In order to send reports out to the reporting agency, the private functions
need to be executed and their results sent out.

The browser can tell where to get reporting and and where to send reports by
observing calls to `window.trailStore.push(...)`. The `push()` function has
reporting origin as its first argument:

Example:

```javascript
// Put some events to trail store.
window.trailStore.push(
    'dsp.example.com',         // <-- reporting origin
    { ... })
```

After some time, say, 24-48 hours later (to prevent timing attacks) the user
agent fetches `https://dsp.example.com/.well-known/reporting.js`, and invokes
`trailStoreMap()`, which then in turn will do whatever reporting computation
the reporting agency wants to do.

If no new events are pushed to trail store in some amount of time, user agent
stop any further calls to `https://dsp.example.com/.well-known/reporting.js`.

### Reports

Reports can be simple JavaScript objects:

```javascript
function trailStoreMap(store) {
    ...
    return {<aggregation key 1>: {'value': <value>},
            <aggregation key 2>: {'value": <value>},
            ...}
}
```

For example:

```javascript
function trailStoreMap(store) {
    ...
    return {'page_views': {'value': page_views},
            'conversions': {'value': conversions}}
}
```

This way we can return multiple reports at a time.

In practice, we would likely add date, advertiser, and version to the keys, e.g.:

```javascript
    return {'page_views-2020-10-15-advertiser.example.com-v0.1': {'value': page_views},
            'conversions-2020-10-15-advertiser.example.com-v0.1': {'value': conversions}}
```

This is modeled after [Conversion Measurement API service
proposal](https://github.com/WICG/conversion-measurement-api/blob/master/SERVICE.md)

## Security and privacy questions

### Avoiding poisoning attacks

Scenario: someone hacks their browser and modifies it so that it produces garbage.

There are some mitigations to this.

[Prio](https://crypto.stanford.edu/prio/paper.pdf) describes a mechanism where
a user agent can present a zero-knowledge proof that a value was computed
through a known arithmetic circuit. It is possible to have server-side
infrastructure verify that the user agent is sending values in a certain range
using their proposal.

In practical terms this means, for example, that we can encode constraints "page views is between 0-100" and the browser cannot lie about this no matter how much hacking is done. It will limit the extend of "lying" a single browser can do.

Example hypothetical API:

```javascript
function trailStoreMap(trail_store) {
    ...
    return { 'page-views': {'value': 23,
                            'range-proof': [0, 100] } }
}
```

### What prevents `reporting.js` from encoding a trail into lots of reports and sending them?

We believe this type of attack would be difficult to pull off.

Reports are processed independently from each other. To elaborate: let's say
`trailStoreMap()` produced a report like this:

```javascript
function trailStoreMap(store) {
    return { 'page_views': { 'value': 100 },
             'unique_id':  { 'value': 0x123982 } }
}
```

The two reports `page_views` and `unique_id` should be sent separately and at
different times. A proxy server can be implemented that blinds IP addresses.
K-anonymity can reject reports that are unique.

In other words, this attack can be mitigated in the server-side aggregation
infrastructure.

## Relationships with other proposals

### Conversion Measurement API

This proposal was written assuming an aggregation framework very similar to
[Conversion Measurement API **Multi-Browser Aggregation Service
Explainer**](https://github.com/WICG/conversion-measurement-api/blob/master/SERVICE.md).

You can do conversion measuring with our proposal but results will need to go
through differential privacy and K-anonymity to preserve privacy.

### TURTLEDOVE & TERN

References:
[TURTLEDOVE](https://github.com/WICG/turtledove/blob/master/README.md)
[TERN](https://github.com/WICG/turtledove/blob/master/TERN.md)

We can make interest groups available in the trail store.

```javascript
const myGroup = {'owner': 'www.wereallylikeshoes.com',
                 'name': 'athletic-shoes',
                 'readers': ['first-ad-network.com',
                             'second-ad-network.com']
                };
window.navigator.joinAdInterestGroup(myGroup, 30 * kSecsPerDay);
```

We can make this have a side-effect that is equivalent to calling:

```javascript
window.trailStore.push('dsp.example.com', { 'type': 'join-interest-group', ... })
```

Or we can simply, when we call `window.navigator.joinAdInteresetGroup(...)`,
also call `window.trailStore.push(...)`.

### MURRE

[MURRE](https://github.com/AdRoll/privacy/blob/main/MURRE.md#the-construction-of-trails)
requires that the browser collects a trail of events. In MURRE, this trail of
events is used to build machine learning models.

The idea of trail store in SPURFOWL could be used for MURRE's use case, perhaps
we can have a partially shared mechanism for both reporting and machine learning.

## Example: Linear attribution conversion model

Each impression is given equal credit. Clicks are ignored in this example.

```javascript
function trailStoreMap(trail_store) {
    var seen_by_advertiser_by_ad = {};
    var credit_by_advertiser_by_ad = {};
    var totals_by_advertiser = {};
    var conversions_by_advertiser = {};

    // compute weights for all ads
    for (var i = 0; i < trail_store.length(); ++i) {
        var event = trail_store.get(i);
        if (event.type === 'impression') {
            var adv = event.conversion_destination;
            var ad = event.data;
            seen_by_advertiser_by_ad[adv] = seen_by_advertiser_by_ad[adv] || {};
            seen_by_advertiser_by_ad[adv][ad] = seen_by_advertiser_by_ad[adv][ad] || 0;
            seen_by_advertiser_by_ad[adv][ad] += 1;
            totals_by_advertiser[adv] = totals_by_advertiser[adv] || 0;
            totals_by_advertiser[adv] += 1;
        }
        if (event.type === 'conversion') {
            var adv = event.conversion_destination;
            credit_by_advertiser_by_ad[adv] = seen_by_advertiser_by_ad[adv];
            conversions_by_advertiser[adv] = conversions_by_advertiser[adv] || 0;
            conversions_by_advertiser[adv] += 1;
        }
    }

    // normalize weights to sum up to number of conversions
    for (var adv in credit_by_advertiser_by_ad) {
        var credit = credit_by_advertiser_by_ad[adv];
        for (var ad in credit) {
            credit_by_advertiser_by_ad[adv][ad] /= totals_by_advertiser[adv];
            credit_by_advertiser_by_ad[adv][ad] *= conversions_by_advertiser[adv];
        }
    }

    // prepare report
    var report = {};
    for (var adv in credit_by_advertiser_by_ad) {
        var credit = credit_by_advertiser_by_ad[adv];
        for (var ad in credit) {
            report['linear-model-conversions-' + adv + '-' + ad] = credit[ad];
        }
    }

    return report;
}
```

