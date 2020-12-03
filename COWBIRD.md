# COWBIRD: Coordinated Optimization Without Big Resource Demands

## Introduction

This document proposes an approach to simultaneously achieving two
conflicting goals: preserving privacy for browser users, and enabling
machine learning optimization within the ad-tech ecosystem.

In particular, we propose the use of a [federated learning](https://en.wikipedia.org/wiki/Federated_learning)
paradigm in which the browser is responsible for evaluating tiny ad-tech
models stored on-device. Later, the browser evaluates the gradients of
these models and sends them to an aggregation service. After these
gradients are aggregated, they are sent to the corresponding ad-tech
company, where they can be used to privately improve model accuracy.

In this proposal, the browser acts as a federated learning platform,
allowing ad-tech companies to optimize toward customizable objectives
with customizable models and features.

### Background

The ad-tech ecosystem pays publishers for the content they create while
enabling companies to advertise their products. A key component of this
value exchange is the ability of buyers to accurately value ad
opportunities. Without this ability, there is an asymmetry of
information about the value of ad opportunities that could have
[severe adverse effects](https://en.wikipedia.org/wiki/The_Market_for_Lemons)
on the efficacy of this market.

Presently, buyers learn the value of ad opportunities by training
machine learning models on centralized datasets collected using
third-party cookies. However, third-party cookies pose serious privacy
risks, and there is much interest in deprecating them. Consequently,
we would like to design a privacy preserving mechanism by which buyers
in the ad-tech ecosystem may perform machine learning optimizations.

### The Motivating Observation

The primary observation inspiring this proposal comes from [MURRE](https://github.com/AdRoll/privacy/blob/main/MURRE.md#selecting-epsilon):
the sparse logistic regression models like those used at NextRoll can be
reduced in size by several orders of magnitude without sacrificing all
that much in terms of performance. We even see tiny bidding models with
as few as `2^15` weights performing reasonably well.

Moreover, [1] shows that we can represent model weights using two
bytes or less without degrading performance. (Other compact encoding
schemes such as [minifloats](https://en.wikipedia.org/wiki/Minifloat)
could also work.) This means we can have a decent bidding model
despite using only approximately `2^16` bytes or 64KB. With models
this small it becomes possible to run many of them on-device.


## The Proposal

### Scope

This proposal is concerned specifically with enabling machine-learning
optimizations within the ad-tech ecosystem. Were this proposal to be
adopted, other mechanisms would still be required to address other open
questions about the functioning of the ad-tech ecosystem in a world
without third-party cookies.

### Prerequisites

This proposal assumes that the browser is storing an event history in a
way resembling the [SPURFOWL](https://github.com/AdRoll/privacy/blob/main/SPURFOWL.md)
proposal. These events include impression, pixel, click, and conversion
events along with some associated metadata. This private cache of
personal data is to remain in the browser, and as described below,
will be leveraged to provide feature information to the local bidding
models.

### Step 1: On an Advertiser's Site

In this proposal, when a person browses an advertiser's site, the DSP
asks the browser to download some data and code that will allow the
browser to evaluate bidding logic on behalf of the DSP. In particular,
the following data associated with that DSP is to be periodically
pulled by the browser:
* Interest groups like those described in TURTLEDOVE/TERN
* A limited amount of versioned model data
* A function to compute model features for an ad-opportunity given:
    * Contextual information related to the ad opportunity
    * The browser event history
    * The interest group data
* A function to evaluate the model given model features

For example, we might be interested in performing sparse binary
logistic regression. In this case, the model data would be an array
of floats. The model feature function could extract information such
as the number of impressions the user has seen from the advertiser
associated with the interest group from the browser event history.
(See [this section](https://github.com/AdRoll/privacy/blob/main/MURRE.md#feature-extraction)
of MURRE for a more detailed explanation of feature extraction).
Evaluating the model given features, in this example, would be
summing weights in the array of floats and passing the result through
the sigmoid function.

The DSP also asks the browser to periodically pull code for computing
the model gradient. The model gradient is what allows the DSPs to
optimize their bidding based on the outcomes associated with purchased
impressions. In particular, the following data is written into the
browser:
* A function to determine observation labels given the browser event history
* A function to determine the model gradient given:
    * The label of the observation
    * The model features associated with the observation
    * The model data

As an example, let's say a DSP is interested in optimizing for long
clicks, which are clicks that successfully load the landing page URL.
Whether an impression resulted in a long click can be determined from
the browser event history by checking for clicked impressions followed
shortly by a pixel event from the advertiser associated with the
impression. Under this proposal, a DSP could label impressions by
whether or not they long clicked in order to train a long click
prediction rate model. As another example a DSP could label clicks by
whether or not they resulted in a conversion, allowing a DSP to train
a post-click conversion model.

Thus, a strength of this proposal is that it allows DSPs to make their
own modeling choices including how to label their training data, and
what sort of model to use. The key restrictions are that these models
must be small so as to fit comfortably on-device, and that these
models need to be trainable using gradient descent.

### Step 2: On a Publisher's Site

On the publisher's site,  model inference is straightforward. It is the
browser's responsibility to evaluate each model for each of its
associated interest groups. By design, the browser has everything it
needs to compute bids for any impression opportunity. In particular,
the browser can do something like the following:
```
all_bids = []
foreach model in models_downloaded:
    foreach interest_group associated with model:
        features  = compute_features(interest_group, contextual_info, browser_history)
        bid_price = compute_bid(features, model)
        all_bids.append(bid_price)
```

At this point, the browser could run the auction or the bids could be
communicated to whatever party is responsible for the auction itself.

It is important for the gradient computation, described next, that we
store the features associated with the winning bid. A natural place
for this is in the browser event history.

### Step 3: Label and Gradient Generation

We must wait some period of time before we can know the correct label
for an observation. If we are interested in whether an impression long
clicked then this could be on the order of 20 minutes. If we are
are interested in whether a click became a conversion this delay could
be days or weeks. Consequently, when and how labels and gradients are
evaluated is somewhat subtle.

One approach to this timing issue is that label and gradient generation
be attempted periodically. In this implementation the label extraction
functions could have a return value signifying that an observation's
label is not yet ready. It would also be important to keep track of
which observations had already reported their gradients so as not to
double count anything.

We propose the browser compute something like the following for each
DSP with data in the browser:
```
batch_gradient = (0, ..., 0)
foreach observation in browser_history:
    if observation.dsp = current_dsp and observation is not labeled:
        label = compute_label(observation, browser_history)
        if label is not SUBSAMPLE_EVENT or RECOMPUTE_LATER:
            features = observation.saved_model_feature_state
            gradient = compute_gradient(label, features, model)
            batch_gradient += gradient
```

At this point, the batch gradient can be sent to the gradient
aggregation service. The submission should be keyed on both the DSP
and a model version. Keying on model version is necessary because
gradients are only meaningful in the context of the weights they
were computed against. It is worth noting that associating a model
version with gradients submissions also unlocks A/B testing.

It would also make sense for each DSP to send information about the
state of model training to the aggregate reporting API at this stage
of the process. In particular, information such as the training loss
over the observations in the browser should be sent for aggregation
so that the value of optimization objective function can be known
for each model version.

### Step 4: Gradient Aggregation

The gradient aggregation service simply sums gradients associated with
a each model version and DSP. The gradient aggregation service may also
distort model gradients before summing for privacy purposes. This is
is elaborated on below. After a sufficient number of gradient updates
have been collected for a particular model, the results may be pulled
by the DSP.

### Step 5: Model Training and Updates

To train their model, the DSP takes a step in the direction of the
gradient computed by the gradient aggregation service. Weight decay,
as described in [2] can also be applied for regularization, if
desired. Information about the performance of the model can be pulled
from the aggregate reporting API to monitor things like overfitting.

At this point, the DSP can release a new and improved version of their
model. Browsers, periodically pulling the latest model version, would
eventually receive this model update. At which point, the process
described here can repeat itself indefinitely.

It is worth noting here that the `compute_bid` function need not
leverage the model being trained. For example, it could be reasonable
to flat-bid during the first few training iterations of a new model.

## Resource Considerations

Resource limitations are a concern for the feasibility of this proposal.
We should consider each of the following:

* The amount of on-device memory consumed by bidding models
* The amount of network communication required to communicate models and gradients
* The amount of CPU required to compute predictions in the browser

For the back-of-the-envelope style calculations that follow, we will
assume that the browser has a limit of `2^9 ~ 500` ad-tech models per
device, and that each model is 128 KB. This isn't to say that this many
models will be needed on each device, but rather to give a rough sense
of the scale of resources that may be consumed on-device under this
proposal. It is likely far fewer models would be required for a
typical device.

### On-Device Memory

If we had 500 models each of 128KB in size we would need 64MB of memory
in which to store them. Gradients are likely to be sparse, but they
could require as much as another 64MB of memory in the worst case.
Memory for gradients would only be needed until gradients are sent to
the aggregation service.

### Network Communication

First, each model needs to be downloaded on a regular cadence, perhaps
at the start of each browsing session. This would require as much as
64MB of data to be downloaded per session, per device. In the case of
model downloads, compression is unlikely to have a large impact
on communication requirements because these tiny models are unlikely
to have unused weights, and each weight is likely to be distinct.

Second, model gradients need to be communicated to the aggregation
servers on a regular cadence. The end of each browsing session might
be a reasonable time to do this. Most likely these will be sparse
gradients. After accumulating a browsing session worth of these sparse
gradients on the device we might have approximately `2^9` non-zero
gradient components per model. Let's also assume that each of the 500
models had a labeled observation, and therefore a gradient. With 2
bytes per gradient component this might require that roughly 512KB
(`2 * 2^9 * 2^9 = 2^19`) of gradient data be uploaded per session, per
device. The upload format of this data could be a sparse vector, or
it could be a compressed dense vector. In the case of a sparse
gradient, compression should work well. In the worst possible case,
where every model required a dense gradient, this proposal might ask
the browser to upload as much as 64MB of gradient data per session,
per device. As with the model data download, in this worst case
scenario, compression would probably not reduce the communication
requirements much.

However, it is important to note that if models failed to download
from time to time on a subset of browsers with limited connectivity
that would be acceptable. The same is true of failed gradient update
uploads (so long as they didn't systematically bias the resulting
model). With this in mind, network usage minimizing optimizations,
such as only uploading a subset of gradients, would be possible in
certain situations.

### On-Device CPU Requirements

We are proposing that machine learning models be evaluated by the
browser. This includes feature extraction, model evaluation,
observation labeling, and the computation of the model gradient. We
are proposing feature extraction and model evaluation be done for
each ad-tech model stored in the browser as a webpage with an
advertisement opportunity loads. This would, of course, need to be
done in such a way as to not impact user-experience. At NextRoll, it
takes roughly one millisecond to perform feature extraction and
evaluate our model. Using this crude estimate, it could take
approximately half of a second to evaluate 500 ad-tech bidding models
on a single CPU.

## Privacy Considerations

Federated learning provides strong privacy protections because nearly
all data created by the browser user is kept on-device. The main
possible source of private data leakage is the model gradient because
this vector eventually makes its way to ad-tech companies. Let's
explore how a malicious client of this system might try to violate
privacy, and how these attacks could be mitigated.

### The Attack Surface

In this proposal ad-tech companies provide their own logic for feature
extraction, and are therefore in control of what hashed features are
created. This could, in theory, allow for sensitive information to be
encoded in the presence or absence of features. An example of such an
attack is presented in [issue #1 of MURRE](https://github.com/AdRoll/privacy/issues/1#issuecomment-729987903).

Similarly, in this proposal ad-tech companies provide their own logic
for gradient computations. This could, in theory, allow for attacks in
which private information is encoded in the value of one or more
gradient components. An example of such an attack is also presented in
[issue #1 of MURRE](https://github.com/AdRoll/privacy/issues/1#issuecomment-730555766).

### Possible Mitigations

It is an open question to what extent attacks, such as those above, can
be prevented, while still allowing non-malicious ad-tech players the
ability to do machine learning.

That said, there are at least mitigations for the attacks above. The
first mitigation is the aggregation of gradients. In theory, the sparse
gradients from browsers could be aggregated until they resulting
batch gradient became dense. This would mitigate attacks based on
programming features, and some attacks based on programming gradient
values.

A second class of mitigation approaches involves introducing small
distortions into the gradients. One such approach is adding mean 0 noise
to gradients. Another approach is gradient clipping, where components
are not allowed to be too large. It would also be possible to drop
random components of some gradients. Yet another option is scaling
the gradients by a random positive number with mean 1. Each of these
distortions probably prevent some classes of attacks based on
programming the gradient values, while still providing useful data
to the ad-tech ecosystem.

Enabling these defenses is the reason for the aggregation service
introduced in the proposal.

## Shortcomings

This proposal has some known shortcomings, and probably some
shortcomings that are to be discovered. We will try to enumerate the
key ones here.

* **Small models** -- In this proposal the browser stores and evaluates
the models of many DSPs bidding on ad opportunities for a particular
browser user. As a result, small models are necessary for the
feasibility of this proposal, and some prediction accuracy reduction
would be inevitable.

* **Bidding logic in the clear** -- My understanding is that there is
currently no way to keep the data and code downloaded by the browser
from prying eyes. This would mean any DSP could see the bidding logic
of any other. This is a shortcoming because this bidding logic could
include intellectual property. Perhaps some legal protections may
apply, but ideally we could come up with a way of obfuscating this
code and data so as to make reverse engineering another DSP's bidding
logic very difficult, at a minimum.

* **Lack of centralized data** -- In this proposal, DSPs do not ever
directly see any user data. This is what makes the proposal strong
from a privacy perspective, and also what makes it weak from other
perspectives. In particular, its not clear how one can do basic
things like backtesting a modeling or code change.

* **Contextual targeting** -- Contextual targeting is not well
supported by this proposal. This is because this proposal does not
include a simple way of identifying and filtering based on ad
context. That said, other types of upper-funnel advertising such
as look-alike targeting is supported by this proposal.

## Relationships with Other Proposals

* **[TURTLEDOVE](https://github.com/WICG/turtledove/blob/master/README.md)** /
**[TERN](https://github.com/WICG/turtledove/blob/master/TERN.md)** --
This proposal is compatible with a TURTLEDOVE framework. The interest
group request would work in more or less the same way. The only
difference is that in addition to interest group information, a tiny
bidding model would also be written into the browser. Another
difference is that the contextual request becomes less important
under this proposal. This is because contextual information would
be processed for bidding within the browser. That said, the
contextual request would still be useful to solicit bids based
purely on ad context. A potential advantage of this proposal over a
TURTLEDOVE-like framework is that model inference is much simpler
since no partial model evaluations are needed.

* **[Aggregate Reporting API](https://github.com/csharrison/aggregate-reporting-api)** --
This proposal suggests an aggregation service for gradients not
unlike the aggregate reporting API. Gradient aggregation could
perhaps be built into the aggregate reporting API. Regardless,
this proposal would likely leverage the aggregate reporting API as
it is currently envisioned for estimating the value of the machine
learning objective function achieved by each model version.

* **[MURRE](https://github.com/AdRoll/privacy/blob/main/MURRE.md)** --
This proposal is inspired by MURRE and similar in a number of ways,
most notably that feature extraction for machine learning happens on
the browser. The main difference is that aggregated gradients, rather
than individual observations are received by ad-tech companies.
Receiving individual observations rather than aggregated gradients
makes optimization easier, but less privacy preserving. Unlike
individual observations, gradients can be summed, making them less
vulnerable to attacks from programmed features. That said, gradient
components are real-valued unlike the binary features in MURRE, which
opens new avenues of attack. However, gradients can be distorted many
ways without losing their utility for optimization so perhaps these
attacks can be prevented.

* **[PARRROT](https://github.com/prebid/identity-gatekeeper/blob/master/proposals/PARRROT.md)** --
This proposal is concerned with the generation of bids, as opposed to
their evaluation within an auction. In particular, this proposal is
agnostic about which party runs the auction.

* **[PELICAN](https://github.com/neustar/pelican)** --
This proposal provides a concrete mechanism by which ad-tech companies
might train statistical multi-touch attribution models.

## References
1. Sculley, D., Golovin, D., Young, M. [Big Learning with Little RAM](http://www.eecs.tufts.edu/~dsculley/papers/round-model.pdf). 2012.
2. Loshchilov, I., Hutter, F. [Decoupled Weight Decay Regularization](https://arxiv.org/pdf/1711.05101.pdf). 2019.