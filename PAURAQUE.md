# PAURAQUE

## Authors
[Jabari Pulliam](https://github.com/jabari-pulliam) (NextRoll)
[Andrew Pascoe](https://github.com/appascoe) (NextRoll)
[Valentino Volonghi](https://github.com/dialtone) (NextRoll)

## Introduction
PAURAQUE (Profile-based API for User Attribute Queries) aims to address the concerns of doing User Attribute Targeting without third-party cookies and cross-site tracking through use of an in-browser and user managed profile database.

### B2B vs. B2C Marketing
B2B differs from traditional B2C marketing in that, instead of looking for individuals meeting certain demographic and behavioral criteria, it is focused on identifying key businesses (Accounts) based on firmographic data such as industry, company size, and location, then advertising to contacts at that business based on their role. Marketers identify sets of key accounts to target based on current or prospective business. This makes behavioral modeling of individual users not a core need for B2B.

### Core B2B Use Cases
1. Target ads to people who work at companies on a Target Accounts List (TAL) who have a particular job function.
2. Report on spend, clicks, impressions, page views, view-through conversions, and click-through conversions with company-level granularity.

### User Attribute Targeting
B2B advertising is based on User Attribute Targeting. Job function, employer, and firmographic data related to the user's employer are all considered attributes of the user visiting a site. The mechanisms used to determine these attributes are a mixture of IP address to domain name resolution (for employer) and linking of third-party cookies to attributes obtained from third-party data sources. This allows User Attribute Targeting to leverage traditional web advertising technology. However, these methods do not protect the user's privacy and are at risk from the deprecation of third-party cookies and the possible implementation of IP blindness.

## Proposed Solution

> Instead, if I’m bothered by random ads, I’ll create a profile to tell advertisers that I’m interested in spy fiction, heavy metal, spicy food, and that I’m not into sports.
>
> That’s **my** profile; my version of myself that I’m comfortable getting ads for.
>
> — John Wilander https://twitter.com/johnwilander/status/1226223367172194304

Our proposed solution for these ABM concerns centers around a few key aspects. At a high level, these are:
1. An in-browser profile for user attributes and interests
2. User control of what data is in their profile
3. User control over which sites can access their profile
4. Attributes shared with a site are only available in the privacy sandbox

### In-Browser Profile
Proposals such as TURTLEDOVE, FLoC, and their derivatives essentially center around the development of an in-browser behavioral profile for each user based on their activity. We propose extending this profile to also contain attributes provided to the browser by the user and which the user may choose to share securely with interested parties. Major browser vendors already have a concept of a user profile that is linked to their Google, Apple, or Firefox accounts and also stores information such as passwords, addresses, and even credit card numbers, which the user may then choose to share with sites using form fills.

Our proposal would standardize this concept and make certain user attributes available to sites through an API. The browser would maintain two objects. The first object would be a user profile with a set of standard fields and a denylist. The second would contain a set of positive interests, negative interests, and a denylist.

Ex.
```
{
  'interests': {
    'positives': ['spy fiction', 'heavy metal', 'spicy foods'],
    'negatives': ['sports'],
    'denied-domains': ['whatever.example']
  },
  'profile': {
    'geo': 'NYC',
    'job_title': 'cfo',
    'company': 'bear stearns',
    'denied-domains': ['badsite.example', 'nothankyou.example']
  }
}
```
### User Control
The user would enter their profile data through a browser settings UI.

There would be three ways of adding interest data for a user.

The first would be allowing the user to explicitly declare their interests in a browser settings UI.

The second would allow sites to request to add interests for users. This would happen through an API call that would result in a UI prompt asking the user if they are interested in a certain category or set of categories related to the page they are viewing. The interest categories would be subject to k-anonymity constraints. Using categories from a standard set may make this threshold more likely to be reached.

The third would allow the User Agent to request to add an interest for a user based on their browsing history. This is similar to the FLoC proposal, but instead of assigning an opaque identifier which represents a cohort, the browser would set an interest category which is transparent to both the user and sites consuming the data.

The user could choose to require being prompted whenever a site or the User Agent wants to add an interest, or they could choose to allow additions without explicit permission each time. Interests and profile data could be removed by the user at any time and the user could add a site to the denylist to block access.

#### Example: Programmatically adding an interest
```js
addInterests(['spy fiction', 'heavy metal', 'sneakers']).then((result) => {
	if (result.isSuccessful) {
		...
	}
});
```

### Sandboxed Access
Access to the user's profile and interests would follow the TURTLEDOVE/FLEDGE model and only be allowed from JavaScript running within the privacy sandbox. This is to prevent any PII or first-party data from being tied to information in the user's profile.

Furthermore, privacy can be maintained by having minimum audience sizes for an ad, which would prevent advertisers and intermediaries from gaining identifying information based on how many times the ad was served. For example, if a campaign was targeting the CEO of Microsoft, inside the sandbox, the bidding logic would be able to see that the user is the CEO of Microsoft, which is obviously a single person. However, with a threshold presumably of some number much greater than one, such a microtargeted campaign would never be able to render an ad. The impact would be needing to build campaigns with a broader audience, such that they target any CEO on an advertiser's Target Account List, which may have thousands of companies.

## Open Questions
### Trustworthiness of Profile Data
Since a user has full control over their profile, there may be a concern over how trustworthy the information in it is. A user could put false values for company or job title. Or, the user could change jobs without updating their profile. It may be valuable for attributes to be able to be verified. Verified attributes may generate a higher bid during the auction.

This could be solved by introducing attribute status, where an attribute could be in the following states:
* Unverified
* Verified
	* Verification Date

A third-party service could be used to provide this verification and issue a certification or [Trust Token](https://web.dev/trust-tokens/) which could be attached to the attribute.

### How will data flow in the auction?
Depending on whether this runs as part of FLEDGE or PARAKEET, or other proposals, it will be important to understand how data will be presented in the auction to determine bid/no-bid decisions.

A few simple options come to mind:

  1. Passing this data straight in the contextual request of a FLEDGE-like spec can be problematic, as it exposes this data to be joined with other contextual request data which could reveal too much. For this purpose it may be better to introduce a new request type besides contextual, like a user attribute request which only contains this data preventing the joining.
  2. In a PARAKEET-like spec the Trusted Server can act as the anonymizer of this data and decouple the user attributes from contextual data to minimize risk of merging the 2 before enabling an auction.
  3. Create an interest-group-like structure for submitting bids given a set of attributes. The clear downside of this option is however is that in order to start submitting bids, the bidder needs to have seen the user before to be able to call a `navigator.joinInterestGroup`-like function.

Other options are available and can be found by studying this problem more in depth, an example of alternative approaches here could be to hash the attributes with an algorithm generating a controlled amount of collisions to guarantee noise, add/remove random attributes and so on.

### Reporting size of targeted attributes
Trusted Servers or Aggregate Reporting APIs should enable the ability for buyers to establish the size of the groups being targeted. This is necessary to establish an optimal budget and early campaign parameters to achieve advertiser goals. In order to achieve this goal there should be a mechanism for the User Agent to send reporting data as well as for standard APIs to access these reports.

### Standard set of interests
Should there be a standard set of interest, e.g. modeled after IAB categories, or should they remain freely settable and purely rely on the market to establish this on its own?
