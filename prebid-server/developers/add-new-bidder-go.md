---
layout: page_v2
sidebarType: 5
title: Prebid Server | Developers | Adding a New Bidder

---

# Prebid Server - Adding a New Bidder in Go
{: .no_toc}

* TOC
{:toc }

This document describes how to add a new bid adapter to the Go version of Prebid Server. Our recommendation is to build new adapters in Go because we port them to Java within a couple of months. But if you'd like to build them yourself in both, there are [instructions for building an adapter in PBS-Java](/prebid-server/developers/add-new-bidder-java.html).

**NOTE**: To make everyone's lives easier, Bidders are expected to make net-price bids (e.g. "If this ad wins, what will the publisher make?"), not gross-price bids.
Publishers can correct for gross-price bids by setting [Bid Adjustments](/prebid-server/endpoints/openrtb2/pbs-endpoint-auction.html#bid-adjustments) to account for fees.

## Choose a Bidder Name

This name must be unique. Existing BidderNames can be found [here](https://github.com/prebid/prebid-server/blob/master/openrtb_ext/bidders.go).

Throughout the rest of this document, substitute `{bidder}` with the name you've chosen.

## Define Your Bidder Params

Bidders may define their own APIs for Publishers to pass custom values subject to these guidelines:

- Don't duplicate values already present in the [OpenRTB 2.5 spec](https://www.iab.com/wp-content/uploads/2016/03/OpenRTB-API-Specification-Version-2-5-FINAL.pdf).
- Don't add bidder-specific parameters that already have Prebid conventions: first party data, floors, schain, video params, referrer, COPPA.

Publishers will send values for these parameters in `request.imp[i].ext.{bidder}` of
[the Auction endpoint](/prebid-server/endpoints/openrtb2/pbs-endpoint-auction.html). Prebid Server will preprocess these so that
your bidder will access them at `request.imp[i].ext.bidder`--regardless of what your `{bidder}` name is.

## Implement Your Bidder

Bidder implementations are scattered throughout several files.

- `adapters/{bidder}/{bidder}.go`: contains an implementation of [the Bidder interface](https://github.com/prebid/prebid-server/blob/master/adapters/bidder.go).
- `openrtb_ext/imp_{bidder}.go`: contract classes for your Bidder's params.
- `usersync/usersyncers/{bidder}.go`: A [Usersyncer](https://github.com/prebid/prebid-server/blob/master/usersync/usersync.go) which returns cookie sync info for your bidder.
- `usersync/usersyncers/{bidder}_test.go`: Unit tests for your Usersyncer
- `static/bidder-params/{bidder}.json`: A [draft-4 json-schema](https://spacetelescope.github.io/understanding-json-schema/) which [validates your Bidder's params](https://www.jsonschemavalidator.net/).
- `static/bidder-info/{bidder}.yaml`: contains metadata (e.g. contact email, platform & media type support) about the adapter

Bidder implementations may assume that any params have already been validated against the defined json-schema.

### Bid Response Metadata

In addition to the standard OpenRTB2.5 response fields, Prebid encourages bidders to
provide additional metadata in their bid response:

{% highlight js %}
{
  "seatbid": [{
    "bid": [{
      ...
      "ext": {
        "prebid": {
          "meta": {
            "networkId": NETWORK_ID,
            "networkName": NETWORK_NAME
            "agencyId": AGENCY_ID,
            "agencyName": AGENCY_NAME,
            "advertiserId": ADVERTISER_ID,
            "advertiserName": ADVERTISER_NAME,
            "advertiserDomains": [ARRAY_OF_ADVERTISER_DOMAINS]
            "brandId": BRAND_ID,
            "brandName": BRAND_NAME,
            "primaryCatId": IAB_CATEGORY,
            "secondaryCatIds": [ARRAY_OF_IAB_CATEGORIES],
          }
        }
      }
    }]
  }]
}
{% endhighlight %}

Notes:

- `advertiserDomains` is the same as the OpenRTB 2.5 `bid.adomain` field but replicated here for downstream convenience
- See the [Prebid.js Bidder Adapter](/dev-docs/bidder-adaptor.html) page for details about the requested values for each field.

{: .alert.alert-info :}
Please provide as much information as possible in the meta object. Publishers use this data for tracking down bad creatives and ad blocking. The advertiserDomains field is particularly useful. Some of these fields may become required in a future release.

### BidResponse Requirements

**Note**: In order to be part of the auction, all bids must include:

- An ID
- An ImpID which matches one of the `Imp[i].ID`s from the incoming `BidRequest`
- A positive `Bid.Price`
- A `Bid.CrID` which uniquely identifies the Creative in the bid.

Bids which don't satisfy these standards will be filtered out before Prebid Server responds.

## Long-Form Video Support
If long-form video will be supported ensure the bidder has the following:

{: .table .table-bordered .table-striped }
|Field          |Type                           |Description                       
|----------------|-------------------------------|-----------------------------|
|bid.bidVideo.PrimaryCategory | string | The category for the bid. This should be able to be translated to Primary ad server format|           
|TypedBid.bid.Cat | []string | The category for the bid. Should be an array with length 1 containing the value in IAB format|            
|TypedBid.BidVideo.Duration | int | Ad duration in seconds|
|TypedBid.bid.Price | float | Bid price|

{% capture alertNote %}
`bid.bidVideo.PrimaryCategory` or `TypedBid.bid.Cat` should be specified.
{% endcapture %}

{% include alerts/alert_note.html content=alertNote %}
To learn more about IAB categories, refer site provided by adtagmacros.com: [IAB categories](https://adtagmacros.com/list-of-iab-categories-for-advertisement/)

## Test Your Bidder

### Automated Tests

Bidder tests live in two files:

- `adapters/{bidder}/{bidder}_test.go`: contains unit tests for your Bidder implementation.
- `adapters/{bidder}/params_test.go`: contains unit tests for your Bidder's JSON Schema params.

Since most Bidders communicate through HTTP using JSON bodies, you should
use the [JSON-test utilities](https://github.com/prebid/prebid-server/blob/master/adapters/adapterstest/test_json.go).
This comes with several benefits, which are described in the source code docs.

If your HTTP requests don't use JSON, you'll need to write your tests in the code.
We expect to see at least 90% code coverage on each Bidder.

Bidders should also define an `adapters/{bidder}/{bidder}test/params/race/{mediaType}.json` file for any supported
Media Types (banner, video, audio, or native). These files should contain a JSON object with all the bidder params
(required & optional) which are expected in supporting that video type. This will be used in automated tests which
check for race conditions across Bidders.

### Manual Tests

Build and start your server:

```bash
go build .
./prebid-server
```

Then `POST` an OpenRTB Request to `https://localhost:8000/openrtb2/auction`.

If at least one `request.imp[i].ext.{bidder}` is defined in your Request,
then your bidder should be called.

To test user syncs, [save a UID](../endpoints/setuid.html) using the FamilyName of your Usersyncer.
The next time you use `/openrtb2/auction`, the OpenRTB request sent to your Bidder should have
`BidRequest.User.BuyerUID` with the value you saved.

## Automated Tests

This project uses [TravisCI](https://travis-ci.org/) to make sure that every PR passes automated tests.
To reproduce these tests locally, use:

```
./validate --nofmt --cov
```

### Writing Tests

Tests for `some-file.go` should be placed in the file `some-file_test.go` in the same package.
For more info on how to write tests in Go, see [the Go docs](https://golang.org/pkg/testing/).

### Adapter Tests

If your adapter makes HTTP calls using standard JSON, you should use the
[RunJSONBidderTest](https://github.com/prebid/prebid-server/blob/master/adapters/adapterstest/test_json.go#L50) function.

This will be much more thorough, convenient, maintainable, and reusable than writing standard Go tests
for your adapter.

### Concurrency Tests

Code which creates new goroutines should include tests which thoroughly exercise its concurrent behavior.
The names of functions which test concurrency should start with `TestRace`. For example `TestRaceAuction` or `TestRaceCurrency`.

The `./validate.sh` script will run these using the [Race Detector](https://golang.org/doc/articles/race_detector.html).

## Add your Bidder to the Server

- Add a new [BidderName constant](https://github.com/prebid/prebid-server/blob/master/openrtb_ext/bidders.go) for your `{bidder}`.
- Update the [newAdapterMap function](https://github.com/prebid/prebid-server/blob/master/exchange/adapter_map.go) to make your Bidder available in auctions.
- Update the [newSyncerMap function](https://github.com/prebid/prebid-server/blob/master/usersync/usersync.go) to make your Bidder available for user syncs.

## Contribute

Finally, [Contribute](https://github.com/prebid/prebid-server/blob/master/docs/developers/contributing.md) your Bidder to the project.
