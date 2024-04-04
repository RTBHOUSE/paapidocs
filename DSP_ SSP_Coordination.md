## PA API - DSP &lt;> SSP Coordination

The Protected Audience API (PA API) is a new API that allows advertisers to target their ads to specific audiences based on their interests. 

To use the Protected Audience API, advertisers will need to work with a DSP (Demand Side Platform) and an SSP (Supply Side Platform). The document aims to explain and streamline preferred integration between a DSP and SSP.

The DSP will be responsible for bidding on ad inventory that is eligible for the Protected Audience API, and the SSP will be responsible for delivering that ad inventory to the DSP.

There are a few key coordination areas that need to be addressed in order to successfully integrate the Protected Audience API. These areas include:



- [Bid Request flow](#bid-request-flow)
  - [Early experimentation](#early-experiment---early-adopters)
  - [OpenRTB protocol proposal](#proposed-standard-path)
- [Creative audit](#creative-audit)
  - [DSP’s renderURL methodology](#rtb-house-example-renderurls)
- [PA on-device bidding data](#pa-on-device-bidding-data-ssp---dsp)
- [PA on-device scoring data](#pa-on-device-scoring-data-dsp---ssp)
- [PA win reporting](#pa-api-win-reporting)
- [Post-auction reporting](#post-auction-reporting)

By working together, DSPs and SSPs can ensure that the Protected Audience API is implemented in a way that benefits both parties. 


## Bid Request flow

The document describes two ways of request-response flow:



1. [Early experiment - early adopters](#early-experiment---early-adopters)
2. [Proposed standard path](#proposed-standard-path)

For the time being both are supported. With the expected near emergence of a standard - the latter is preferred.


### Early experiment - early adopters


#### Bid Request

SSP is expected to send a bid request with a “ae” feature signaling PA API availability inside an extension to the imp object:


```
{
    "id": "12345678912345678911T32",
    "imp": [{
        "id": "1",
        "banner": {
            "w": 300,
            "h": 250
        },
        "tagid": "123",
        "ext": {
            "ae": 1
        }
    }],
(...)}
```


The field conveys two values:



* 0 or no value - **PA API is not available**
* 1: PA API is available - yet is not guaranteed to execute, pending the buyer’s participation and final decision by the publisher

In the future it is expected to grow, also including signaling for Bidding & Auction Services which is not yet widely available & supported.


#### Bid Response

Bidders will respond to a PA API-eligible impression with:



* Standard RTB ad markup
* Standard ad markup with an extension for the PA API
* Only PA API-specific extensions

Bidders may also choose not to bid at all.

An example of PA API-specific markup:


```
{
    "id": "12345678912345678911T32",
    "ext": {
        "igbid": [
            {
                "impid": "1",
                "igbuyer": [
                    {
                        "igdomain": "https://f.creativecdn.com",
                        "buyersignal": {
                            (buyer-specific signals in JSON)
                        }
                    }
                ]
            }
        ]
    },
    "bidid": "12345678912345678911T32",
    "seatbid": [
// Bidder could also return a standard RTB response to participate in contextual auction
        (...)
    ]
}
```


The impid reflects the PA API eligible impression from the bid request. The contents of buyersignal are opaque to the seller and should be passed "as is" to the buyer's on-device bidding function. 

The field name itself can be configured to suit an SSP's needs. For example, [Google Ads spec suggests](https://github.com/google/ads-privacy/tree/master/proposals/fledge-rtb) naming it "buyerdata".


#### On-device bidding: 



1. Ads returned by the BiddingFunction do not have any specific ad metadata
2. RTB House’s preferred bidding currency is USD


### Proposed standard path

Please note, the process is ongoing and could introduce changes, especially nearing the end of 2023.


#### Bid Request

SSP is expected to send a bid request with a “ae” feature signaling PA API availability inside an extension to the imp object:


```
OpenRTB Core BidRequest.imp = {
   // Optional InterestGroupAuction.Request object
   // Provided if the Imp supports an interest group auction mechanism.
   // Absence of this object is equivalent to prior Imp.ext.ae=0 signal.
   // Defined in Interest Group Auctions specification, see section #.## 
   "ig": { 


   // The IG auction environment support for this impression. 
   // For inventory that supports on-device orchestrated interest group
   // bidding, this will be set to ON_DEVICE_INTEREST_GROUP_AUCTION (1). Note that
   // this only indicates that the interest group auction is supported, not
   // that it is guaranteed to execute. If no buyer chooses to participate in
   // the interest group auction, then the interest group auction will be
   // skipped and the winner of the contextual auction, if any, will be
   // served instead.
   // DEFAULT: 1

   "ae": 1,

   // Indicates whether interest group bidding is allowed for the bidder.
   // Depending on account settings and other factors, a bidder might be
   // disallowed from submitting interest group bids, even though an interest
   // group auction may ultimately decide the winning ad.
   // The seller sets this. Example, the publisher intends to enable IG, but the seller (SSP)
   // has not onboarded this buyer for IG auctions. This value should not be filled out by the publisher.
   // Default value of this field is boolean given in text form, but numeric boolean values (0, 1) are also supported.
   // DEFAULT=true

   "biddable": true,
   "ext": {...}  
   }
}
```

Optionally we support also "ae" and "biddable" fields given in alternative sections of imp object. Like in imp -> ext -> igs
With is currently standard from  [official IAB specification](https://github.com/InteractiveAdvertisingBureau/openrtb/blob/main/extensions/community_extensions/Protected%20Audience%20Support.md).


Example below:

```
OpenRTB Core BidRequest.imp = {
	...
	"ext": {
		"igs": {
			"ae": 1,
			"biddable": true
		}
	}  
}
```

Another supported version of sending "ae" feature is creating "ae" field directly in imp -> ext.


Example below:

```
OpenRTB Core BidRequest.imp = {
	...
	"ext": {
		...
		"ae": 1,
		...
	}  
}
```

#### Bid Response

**Please note, the standard is taking shape. Fields could change names over time if consensus is reached.**

Bidders will respond to a PA API-eligible impression with:



* Standard RTB ad markup
* Standard ad markup with an extension for the PA API
* Only PA API-specific extensions

Bidders may also choose not to bid at all.

An example of PA API-specific markup:


```
OpenRTB BidResponse = {

   // Optional 
   // one or more InterestGroupAuction.Response objects
   // Provided if the buyer is signaling participation in potential interest group auction(s)
   // Defined in Interest Group Auctions specification, see section #.## 
   "igi": [ 
   {
       // ID of ad slot represented by the corresponding Imp object in the bid
	// request to link information to be used in the interest group auction to
	// the specific ad slot.
	"impid": "1",

	// array of buyer-level information to use in the interest group auction.
	"igb": [{   	 
        	// Required
        	// Origin of the interest group buyer to participate in the
        	// in-browser auction.
        	// See https://developer.mozilla.org/en-US/docs/Glossary/Origin
        	"origin": "https://ads.dsp.com",
        	// Indicates the currency in which interest group bids will be placed.
        	// Value must be a three digit ISO 4217 alpha currency code (e.g. "USD").
        	"cur": "USD",

        	// Optional
        	// Buyer-specific signals ultimately provided to the buyer's
        	// `generateBid()` function as the `perBuyerSignals` argument.
        	// If specified, seller will add to its auction config
        	// `perBuyerSignals` attribute map, keyed by the interest group buyer
              // origin.
        	// Value may be any valid JSON serializable value.
        	"pbs" = ...,

        	// Optional
        	// Buyer priority signals.
        	// If specified, seller will add to its auction config
        	// `perBuyerPrioritySignals` attribute map, keyed by the interest group buyer origin.
        	"ps": {...},
            "ext": {...}  
  	}]
}]
}
```



#### On-device bidding: 



1. Ads returned by the BiddingFunction do not have any specific ad metadata
2. Preferred bidding currency is USD


## Creative Audit

The Protected Audience API (PA API) is a new method for managing creative flow and providing publisher controls. It introduces a number of changes, including:



* A new, more stable identifier for creative, called renderUrl
* [Ads Composed of Multiple Pieces](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#34-ads-composed-of-multiple-pieces) extension that provides better and richer insight into what is shown on creative

To effectively provide publisher controls, SSPs will need to adapt their creative audit mechanisms to use the new mechanisms. 

During the scoring process in the `scoreAd()` function, SSPs have access to the `renderUrl` identifying a specific creative, as well as componentAds, if provided by the DSP. SSPs also have access to their real-time data from the Trusted Seller Server, which can help inform decisions on specific ads that are submitted by buyers. Based on the `renderUrl`, SSPs are able to determine offline what the creative is about: the products, marketed brand, landing page, and all other significant features. If the ad is not suitable or undesirable in any other way, SSPs can decide to nullify its score and block it from winning the auction.


### RTB House example renderURLs:



1. `https://f.creativecdn.com/creatives?id=gM9yOzfVBXDw0GdeZ1Cv&c=8i8iBPavpChmuCqGQL8t&s=rtbhfledge`
2. `https://f.creativecdn.com/creatives?id=sDFqT3WQQE2WmdeNZxtx&c=HS25kzTy3wDbs8oa43xq&s=rtbhfledge`
3. `https://f.creativecdn.com/creatives?id=0xlS2c1rA3uWiiYL4nhc&c=2Rf830zQiV8UCzdKcDfb&s=rtbhfledge`
4. `https://f.creativecdn.com/creatives?id=1011nfKZY8gUXqcfjdNR&c=dEU0qtF76ATzIppLqNl7&s=rtbhfledge`
5. `https://f.creativecdn.com/creatives?id=d0GSOS1ZgpuI3Ctuf7Ho&c=QxFpeqaC9sJ3YXix43FB&s=rtbhfledge`


## PA on-device bidding data: SSP -> DSP

RTB House currently does not have expectations to receive specific, nonstandard data from SSPs


## PA on-device scoring data: DSP -> SSP

RTB House currently prefers unified, single currency - USD. We plan on following emerging standard, where the contextual response will inform both the Seller and Buyer’s bidding function of the currency - see [Bid Response sample](#bid-response-1)

As a result of a bidding function RTB House also returns an ad object with informations specific to the returned bid, eg:
```
{
    "bid": 1,
    "render": "https://f.creativecdn.com/creatives?id=BcZKHkT4MAlkP8W1GbNn&c=kQKZEgHd2MX0GUG9VHAS&s=rtbhfledge",
    "ad": {
        "adomain": ["example.com"],
        "cid": "kQKZEgHd2MX0GUG9VHAS",
        "crid": "BcZKHkT4MAlkP8W1GbNn_kQKZEgHd2MX0GUG9VHAS",
        "w": 300,
        "h": 250
    },
    "adComponents": ["https://f.creativecdn.com/creatives?id=OdjHedJd8uDClBSou5rp&c=kQKZEgHd2MX0GUG9VHAS&_oi=2258342499549171292&s=rtbhfledge",  ...],
    "bidCurrency": "USD",
    (...)
}
```


## PA API win reporting

Currently, RTB House does not have any specific requirements on data receivable from SSPs’ reportResult() function.

For financial reconciliation the standard process of data comparison is expected to be observed. 


## Post auction reporting

According to the PA API specification, some events, such as clicks, views, and other similar metrics, must be delegated to an SSP by the buyer. RTB House is open to discussing the transfer of necessary information to SSPs on a case-by-case basis, as there is no established standard yet.


### Click-event reporting

There are currently three distinct click-related events in Chrome. 

The most recent information confirms that all three events will be supported in the near future ([#130](https://github.com/WICG/fenced-frame/pull/130)). For the widest possible support, we suggest using all of them. 

The buyer must register events for themselves as well as allow delegation for the direct seller:


```
// report FFAR
window.fence && window.fence.setReportEventDataForAutomaticBeacons && window.fence.setReportEventDataForAutomaticBeacons({
    eventType: 'reserved.top_navigation',
    destination: ['buyer', 'direct-seller']
});
window.fence && window.fence.setReportEventDataForAutomaticBeacons && window.fence.setReportEventDataForAutomaticBeacons({
    eventType: 'reserved.top_navigation_start',
    destination: ['buyer', 'direct-seller']
});
window.fence && window.fence.setReportEventDataForAutomaticBeacons && window.fence.setReportEventDataForAutomaticBeacons({
    eventType: 'reserved.top_navigation_commit',
    destination: ['buyer', 'direct-seller']
});
```


Seller needs to register the event in it’s reportResult() function:


```
// reportResult()
registerAdBeacon({
    "reserved.top_navigation": "seller_endpoint?with_some_params"
}, {
    "reserved.top_navigation_start": "seller_endpoint?with_some_params"
}, {
    "reserved.top_navigation_commit": "seller_endpoint?with_some_params"
})
```



### Other events reporting

The FFAR mechanism allows for the registration and monitoring of various custom events, even before Fenced Frames are enforced and required after 2026. One such example is a creative render, which is an event that cannot be observed by the Seller on its own without cooperation from a buyer.

For example, a buyer and seller could agree to mutually measure ad renders. In order to achieve this, the buyer would need to register the event in the creative and delegate it to the direct-seller. This would allow both parties to track the number of times an ad has been rendered, which can be used to measure and diagnose potential issues with ad renders.

Similarly to click events - buyer needs to register an event:


```
// report FFAR from buyer's creative
window.fence && window.fence.reportEvent && window.fence.reportEvent({
// "render" value of eventType is only an example for discussion purposes only
    eventType: 'render', 
    destination: ['buyer', 'direct-seller']
});
```


Seller needs to also register the same event’s name in reportResult() code in order to be able to receive the report:


```
// reportResult()
registerAdBeacon({
    "render": "seller_endpoint?with_some_params"
})
```



## References



1. Sample of RTB House Creative renderUrls
  - [https://f.creativecdn.com/creatives?id=gM9yOzfVBXDw0GdeZ1Cv&c=8i8iBPavpChmuCqGQL8t&s=rtbhfledge](https://f.creativecdn.com/creatives?id=gM9yOzfVBXDw0GdeZ1Cv&c=8i8iBPavpChmuCqGQL8t&s=rtbhfledge)
  - [https://f.creativecdn.com/creatives?id=sDFqT3WQQE2WmdeNZxtx&c=HS25kzTy3wDbs8oa43xq&s=rtbhfledge](https://f.creativecdn.com/creatives?id=sDFqT3WQQE2WmdeNZxtx&c=HS25kzTy3wDbs8oa43xq&s=rtbhfledge)
  - [https://f.creativecdn.com/creatives?id=0xlS2c1rA3uWiiYL4nhc&c=2Rf830zQiV8UCzdKcDfb&s=rtbhfledge](https://f.creativecdn.com/creatives?id=0xlS2c1rA3uWiiYL4nhc&c=2Rf830zQiV8UCzdKcDfb&s=rtbhfledge)
  - [https://f.creativecdn.com/creatives?id=1011nfKZY8gUXqcfjdNR&c=dEU0qtF76ATzIppLqNl7&s=rtbhfledge](https://f.creativecdn.com/creatives?id=1011nfKZY8gUXqcfjdNR&c=dEU0qtF76ATzIppLqNl7&s=rtbhfledge)
  - [https://f.creativecdn.com/creatives?id=d0GSOS1ZgpuI3Ctuf7Ho&c=QxFpeqaC9sJ3YXix43FB&s=rtbhfledge](https://f.creativecdn.com/creatives?id=d0GSOS1ZgpuI3Ctuf7Ho&c=QxFpeqaC9sJ3YXix43FB&s=rtbhfledge)
2. [Prebid FLEDGE for GPT module](https://docs.prebid.org/dev-docs/modules/fledgeForGpt.html)
3. [Google Ads proposed FLEDGE specification](https://github.com/google/ads-privacy/tree/master/proposals/fledge-rtb)
4. [Chrome’s ScoreAd documentation](https://developer.chrome.com/docs/privacy-sandbox/protected-audience-api/ad-auction/#scoread)
5. [Fenced Frames Ads Reporting mechanism](https://github.com/WICG/turtledove/blob/main/Fenced_Frames_Ads_Reporting.md)
