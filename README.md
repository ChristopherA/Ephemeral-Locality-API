# Ephemeral Locality Service API version 0.4

Version 0.4 - 2008-08-07

Copyright © 2008 by Christopher Allen \<ChristopherA@LifeWithAlacrity.com\> Licensed CC-BY

## Abstract

The Ephemeral Locality Service API enables a user to store short-lived messages with associated location data and to retrieve messages of nearby users. It accomplishes this while maintaining a user-determined level of anonymity.

The API uses no authentication and only stores messages for a predetermined period of time. These features support user privacy and keep the API lightweight enough for use on location-aware mobile devices.

It is in these areas of privacy and simplicity that the API differs from other locality APIs. The Ephemeral Locality Service API is ideal for use during ephemeral events where a more heavyweight, permissions-based API would be overkill.

An example use case is a simple audience polling application for a single conference presentation. Each participating audience member anonymously publishes their agreement or disagreement with the speaker, and then retrieves similar messages stored by their peers based on current proximity.

The format of the API's requests and responses are defined by open standards, so implementing a client is straight-forward.

## Table of Contents

[1.](#conventions) Conventions and Terminology

[2.](#application_notes) Application Notes

[2.1.](#pseudo_anonymity) Pseudo-Anonymity

[2.2.](#lightweight_api) Lightweight API

[3.](#id_generation) ID Generation

[3.1.](#appid_generation) `AppID` Generation

[3.2.](#privateuserid_generation)`PrivateUserID` Generation

[3.3.](#uniqueuserid_generation)`UniqueUserID` Generation

[4. ](#request_throttling)Request Throttling

[4.1. ](#example_throttling_response)Example Throttling Response

[4.2. ](#rationale_for_throttling)Rationale for Throttling

[5. ](#functions)Functions

[5.1.](#messages_new)**`/messages/new`**

[5.2.](#messages) **`/messages`**

[5.3.](#apps_appid)**`/apps/AppID`**

[5.4.](#user_privateuserid)**`/users/PrivateUserID`**

[6.](#security_considerations) Security Considerations

[6.1. ](#eavesdropping_attacks)Eavesdropping Attacks

[6.2.](#man_in_the_middle_attacks) Man-in-the-Middle Attacks

[6.3.](#service_compromise) Service Compromise

[7. ](#references) References

[8. ](#api_change_log)API Change Log

[9.](#api_change_log) Author’s Addresses

## 1. Conventions and Terminology

In this document, API function URLs are shown in a **`bold monospaced font`**. HTTP parameters, values, headers, and bodies are shown in a `monospaced font`. Non-literals are **`underlined`** to distinguish them from the surrounding text.

The ephemeral locality service defined by this API is referred to as simply "the service", while any particular client application is referred to as "the client". A particular user accessing the service via the client is referred to as "a user". A unique identifier for a particular client application is known as an `AppID`. A unique private identifier for a particular user is known as a `PrivateUserID`, while the corresponding unique public identifier for a particular user is known as a `UniqueUserID`.

## 2. Application Notes

This API is intended for use as lightweight, pseudo-anonymous ephemeral locality storage and retrieval service. These specific characteristics offer a number of benefits to client applications as described below.

### 2.1. Pseudo-Anonymity

The publication of personal location information has potential privacy implications. Being able to track a user's location, especially over the course of several days, might allow someone to discern the user's activities, affiliations, place of work, place of residence, times that they are usually away from home, and so on.

And yet there are many circumstances in which a user might want to make their locality information available to other users for a certain length of time. For instance, a user at a conference might want to publish their location information for a short period of time so that other nearby conference attendees can find and meet up with that user. However, the user may not want to publish their location data to conference attendees once the conference has ended.

The API described in this document supports just such a use case and other similar use cases, allowing a user to publish their location data while maintaining the level of anonymity desired through a number of features designed to support this.

This first way in which anonymity is supported is the API's lack of registration or authentication requirements. Each client is responsible for providing its own self-generated `PrivateUserID` hash. A user desiring a very high level of anonymity can generate a new `PrivateUserID` for each location message they store in the service, while a user less concerned about anonymity can use the same `PrivateUserID` for a defined period of time before generating a new `PrivateUserID`.

Using the same `PrivateUserID` for a period of time allows basic continuity of identity for as long as the user desires, which can be useful for certain use cases such as the aforementioned conference scenario.

When publishing a location message, the user provides their private `PrivateUserID` to the service. However, when anyone retrieves that message, they only receive a non-reversible hash of the `PrivateUserID` called a `UniqueUserID`. And because `PrivateUserID`s are generated on the client and not registered with any authority nor associated with any personally identifiable information via the API, there is less of a risk for the locations or trends of a particular `UniqueUserID` from being correlated to an actual individual.

The ephemeral nature of the service's stored messages also enhances anonymity. Messages are purged from the service after a period of time, so even those users who continue using the same `PrivateUserID` (and thus `UniqueUserID`) over the course of several days are somewhat protected. A user who retrieves stored messages from the service will only receive those messages stored relatively recently, making it that much more difficult to spot trends in a `UniqueUserID` 's locations and thereby pierce the veil of anonymity.

The API is designed without any functions allowing clients to retrieve service logs or client IP addresses. This improves the anonymity of a particular client or `UniqueUserID`, as other clients are only able to access the location information that the user intentionally publishes, and not other information such as the identity or location associated with a particular IP address.

There are no API functions for retrieving all messages published by a particular `PrivateUserID` or `UniqueUserID`. Once a particular user moves out of another user's search radius, their location messages will no longer be returned to that searching user. This feature makes it more difficult to track a particular user over a long period of time.

Furthermore, as the API supports the specification of a radius of uncertainty for any published location data, the user can choose to report their location with varying degrees of uncertainty based on how strongly they want to protect their specific location.

One user could report a small radius of uncertainty limited only by their particular client hardware, allowing others to know their specific location to within a few meters. Another user could intentionally report a large radius of uncertainty, thereby only making their general area known.

The service also employs request rate limiting, described in the **[Request Throttling](#request_throttling)** section. This limits the ability of a malicious user to "troll" for in-use `UniqueUserID`s in an attempt to track down particular users or gain access to secret `PrivateUserID`s. It also prevents a malicious user from effectively scanning through a large geographical area for messages from particular `UniqueUserID`s, as there are limits on both the maximum size of the requested radius for any particular location search, and limits on the number of requests that a client may make in a given period of time.

### 2.2. Lightweight API

The API is streamlined to easily store and retrieve locality information without any other extraneous functionality, such as authentication or permissions features. All API requests are simple HTTP GET and POST calls, while API responses are in the form of simple JSON data structures. Libraries for making HTTP requests and parsing JSON responses are available for most popular programming languages, and the API is designed to be usable from mobile devices with limited memory, processing, and bandwidth resources.

Additionally, the API does not have any specific requirements on the free-form messages that it stores, other than size limits. This allows the API to be used from a number of different applications without enforcing cumbersome requirements on the published message data. And because the message storage and retrieval is partitioned by `AppID` , several unrelated client applications can use the service simultaneously for distinct purposes without interfering with one another's stored messages.

## 3. ID Generation

Each ID used in the API is simply a Base-32 SHA1 URN, defined in **[[RFC 3548]](#references)** and **[[RFC 2141]](#references)**. Using a hash for IDs is significant in that it allows the client to generate IDs on its own. It also enables the client to construct IDs from any sort of constituent data that it deems useful.

### 3.1.`AppID` Generation

Since `AppIDs` are generally known and shared among multiple users, the data that is used to generate the hash does not have to be a secret, although it should be unique.

Once an `AppID` has been generated for a particular application, it should be used consistently whenever referring to that application via the API. There is no need to generate a new `AppID` except for a new application.

A reasonable way to generate an `AppID` is to create a hash comprised of the following data:

- a unique device ID, MAC address, or generated local ID

- the current date and time

Generating an `AppID` in this manner allows different users to generate `AppID`s independently without fear of collisions. Moreover, a given user can generate multiple `AppID`s for different applications over a period of time without generating the same `AppID` twice.

Note that the client may use the **`/apps/AppID`** function to check for `AppID` hash uniqueness.

### 3.2. `PrivateUserID` Generation

Unlike an `AppID` , a `PrivateUserID` should be kept secret between the user's client and the service if the user wants to prevent others from publishing messages with the same `PrivateUserID`.

If a user is interested in a very high degree of anonymity, they could publish each message with a different `PrivateUserID` simply by generating their `PrivateUserID` hashes from random numbers. This would make it very difficult to correlate different location messages with a particular user.

On the other hand, if a user desires a degree of continuity of identity among their published messages while still maintaining some anonymity, they could publish each message with a `PrivateUserID` hash comprised of the following constituent data:

- a secret key generated by and stored on the client

- a unique device ID, MAC address, or generated local ID

- the current date (without the time)

Such a `PrivateUserID` generation scheme causes a new `PrivateUserID` to be generated each day. This allows other users to see that a common user is associated with multiple published messages throughout the course of the day. But as soon as the next day rolls around, the user starts using a new `PrivateUserID` due to the date component included within the hash. Therefore, tracking the user's location data over the course of several days is difficult, thereby protecting the user's anonymity.

Note that the client may use the **`/users/PrivateUserID`** function to check for `PrivateUserID` hash uniqueness.

### 3.3. `UniqueUserID` Generation

Because a `PrivateUserID` is secret and known only to the user it represents and the service itself, other users must be prevented from using the API to find out the `PrivateUserID` associated with a message. However, there is still a need for users to be able to determine whether a common user is associated with multiple published messages.

The solution used by the API is to reveal a `UniqueUserID` instead of a `PrivateUserID` when returning information about a particular message. A `UniqueUserID` is simply a hash of the following data:

- the `PrivateUserID` associated with the message

- a secret salted key known only by the service

Returning this hash instead of the `PrivateUserID` allows the client to determine whether two messages were published by a common `PrivateUserID` without actually seeing the secret `PrivateUserID`itself. Thus, because the `UniqueUserID` is a non-reverisble transformation of the `PrivateUserID` , a user cannot impersonate another user by publishing a message with a `PrivateUserID` not their own.

If for some reason a user wants to make their secret `PrivateUserID` no longer secret, they can include it within the contents of a published message itself. However, for most use cases this should not be necessary, and the `PrivateUserID` should be kept secret to ensure that only one user can publish messages associated with that `PrivateUserID` .

A user can determine their own `UniqueUserID` by calling the **`/users/PrivateUserID`** function before storing any messages. Also see the section on the **`/messages`** function return parameters and specifically the `UniqueUserID` parameter.

## 4. Request Throttling

The service may perform throttling or rate limiting on client calls to each of the API functions, as listed in the description for each. Throttling can occur in the following cases:

- Too many requests by a given client IP address in a time period

- Too many requests by a given `AppID` in a time period

- Too many requests by a given `PrivateUserID` in a time period (for those API functions that accept a `PrivateUserID` input parameter)

- Excessive service load

Each function is throttled individually, meaning that in general a call to one function does not affect the throttling performed on calls to any other function. The one exception is that any particular function call can contribute to excessive service load, thereby causing load-based throttling on other functions.

When the service receives a request that causes it to instate throttling based on one of the above cases, it immediately returns an HTTP 503 Service Unavailable error, as described in **[[RFC 2616]](#references)** Section 10.5.4. It does so without returning any parameters for the called function or storing any messages. Along with the HTTP 503 error, the service returns a "Retry-After" header indicating an HTTP date after which the client may retry the request, as described in **[[RFC 2616]](#references)** Section 14.37. If the client retries the request before that time, another HTTP 503 error will be returned.

In addition to returning an HTTP error when throttling is instated, the service may also further rate limit requests from the client's IP or `PrivateUserID` by instating more restrictive rate limits on the client's IP address or `PrivateUserID` (where applicable). After a period of time, the service automatically lifts these more restrictive rate limits, returning the IP and `PrivateUserID` to the default throttling time period. See the description for each API function for more information.

Note that service hosting providers may enforce their own quota-based rate limiting, separate from the service-based throttling described here.

### 4.1. Example Throttling Response

```
HTTP 1.1 503 Service Unavailable
Retry-After: Sat, 20 Dec 2008 23:59:59 GMT
```


## 4.2. Rationale for Throttling

The service performs throttling for a few general reasons:

- Throttling attempts to counter malicious denial of service attacks and other attempts to use up the service's storage space, memory, bandwidth, processing, or other resources.

- Throttling attempts to prevent a sudden non-malicious surge in the service's usage from making the service unavailable for everyone.
- Throttling attempts to prevent a malicious client from "trolling" for a user's location or other information by making repeated requests at an excessive rate. See the Application Notes section for more information.

The service throttles `PrivateUserID` and not just the IP address to handle the case of a client issuing subsequent function calls with different IP addresses via the use of a proxy or NAT router. Note however that distinct `PrivateUserIDs` sharing a given IP address may encounter IP-based throttling based on the requests of a single client.

A malicious client may make a high rate of requests with a given `AppID` in an attempt to cause the service to throttle requests from that `AppID` , thereby preventing all other clients from making requests with that `AppID` . In order to combat this type of attack, throttling on the basis of IP and `PrivateUserID` is performed with much lower rate limits so that the malicious client's IP or `PrivateUserID` is throttled before `AppID` throttling kicks in.

`AppID` throttling is performed to prevent a popular `AppID` from causing excessive service load and preventing access for unrelated `AppID`s.

## 5. Functions

The following web-accessible functions comprise the service's public API. Each function URL is intended to be a called via an HTTP GET or POST request, as described in **[[RFC 2616]](#references)**. Parameter names are case insensitive, and all parameters are required except where indicated.

The character set for all requests and returned values is UTF-8, thereby allowing messages in a variety of languages. Note that all parameter values are sent as UTF-8 string representations of the indicated types. This means that, for instance, an integer value is represented as a string of digits and not binary data.

### 5.1. /messages/new

#### Description

This function stores a new message from the client along with associated location and time stamp data, returning a value to indicate success. The message is ephemeral and only stored for a set duration.

#### Usage

This function is invoked as an HTTP POST with parameters sent as form-encoded values.

#### Input Parameters

**`AppID`: Base-32 SHA1 URN, as described in [[RFC 3548]](#references) and [[RFC 2141]](#references)**

A unique client-generated hash identifying the application that is calling this function. The hash should be of the form: form: `urn:sha1:**hashvalue**`. Example: `urn:sha1:YNCKHTQCWBTRNJIV4WNAE52SJUQCZO5C`

See the section on ID Generation for more information. The client may use the separate **`/apps/AppID`** function to check for `AppID` hash uniqueness before calling **`/messages/new`**.

**`PrivateUserID` : Base-32 SHA1 URN**

A unique client-generated hash identifying the user that is calling this function. The hash should be of the form: form: `urn:sha1:**hashvalue**`. Example: `urn:sha1:PLSTHIPQGSSZTS5FJUPAKUZWUGYQYPFB`

See the section on ID Generation for more information. The client may use the separate **`/users/PrivateUserID`** function to check for `PrivateUserID` hash uniqueness before calling **`/messages/new`**.

**`Time`: internet time stamp in UTC, as described in [[RFC 3339]](#references)**

The UTC date and time to associate with the stored message, usually the client's current date and time. The time stamp should be of the form: `yyyy-mm-dd-Thh-mm-ssZ`. Example: `2008-12-19T16:39:57Z`

**`Latitude`: degrees as double precision number, as described in [[WGS 84]](#references)**

The latitude of the location to associate with the stored message, usually the client's current position as determined by GPS or other sources. The latitude should be positive for latitudes north of the equator and negative for latitudes south of the equator. Example: `47.60638`

**`Longitude`: degrees as double precision number**

The longitude of the location to associate with the stored message, usually the client's current position as determined by GPS or other sources. The longitude should be positive for longitudes east of the prime meridian and negative for longitudes west of the prime meridian. Example: -`122.33083`

**`Altitude`: meters as a double precision number**

The altitude above sea level to associate with the stored message, usually the client's current position as determined by GPS or other sources. Example: `44.43`

This value is optional and defaults to an undefined value, meaning that the altitude is unknown.

**`Accuracy`: meters as a positive double precision number**

The radius of uncertainty around the location defined by the given Latitude and Longitude values. Accuracy does not take Altitude into account. Example: `27`

This value is optional and defaults to 0.

**`Message`: string, 1 to 140 characters**

The user-provided free-form string to store, such as a description of what the user is currently doing. Example: `My Party Is Here at 8pm`

**`MajorVersion`: string, 1 to 64 characters**

The major version of the API in use by the client. Ignored in the initial version of the API, but can be used in future versions to signal incompatible version differences for backwards compatibility.

This value is optional and defaults to an undefined value.

**`MinorVersion`: string, 1 to 64 characters**

The minor version of the API in use by the client. Ignored in the initial version of the API, but can be used in future versions to signal compatible version differences for backwards compatibility.

This value is optional and defaults to an undefined value.

#### Return Parameters

To indicate the result of storing the message, this function returns a JSON object (dictionary) within the contents of the HTTP response body. JSON is described in **[[RFC 4627]](#references)**. Note that the parameters within the JSON object may occur in any order.

**`Result`: JSON string, 0 to 64 characters**

A string indicating success or failure of storing the message. Possible values are OK and `Error.`

**`InvalidParameter`: JSON string, 0 to 64 characters**

The name of the input parameter that is invalid, if any. InvalidParameter is only included if the `Result` value is `Error` and there was an invalid input parameter. If there are multiple invalid parameters, then only one of them is returned.

**`ErrorMessage`: JSON string, 0 to 140 characters**

A human-readable description of the error that occurred when storing the message, if any. `ErrorMessage` is only included if the `Result` value is `Error.` If there were multiple errors, then only one of them is returned.

#### Throttling

The service may perform rate limiting on calls to this function based on client IP address, `AppID` , and/or `PrivateUserID` . When throttling occurs, more restrictive thresholds may temporarily be put into place for the client's IP or `PrivateUserID` .

See the **[Request Throttling](#request_throttling)** section for more information.

#### Availability

Available in Ephemeral Locality Service API version 0.1 and later.

#### Example Request

```
POST /messages/new HTTP/1.1 Host: example.com User-Agent: EphemeralLocalityClient/1.0
Content-Type: application/x-www-form-urlencoded; charset=utf-8 Content-Length: 258 AppID =urn:sha1:YNCKHTQCWBTRNJIV4WNAE52SJUQCZO5C&
PrivateUserID=urn:sha1:PLSTHIPQGSSZTS5FJUPAKUZWUGYQYPFB&
Time=2008-12-19T16:39:57Z&Latitude=47.60638&Longitude=-122.33083&Altitude=44.43&Accuracy=27
&Message=My+Party+Is+Here+at+8pm&MajorVersion=1&MinorVersion=0
```

#### Example Response

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 19 { "Result": "OK" }
```

#### Example `Error` Response**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 107
{ "Result": "Error", "ErrorMessage": "The data store is unavailable, so the message could not be saved." }
```

#### Example `Error` Response for Invalid Input**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 105 { "Result": "Error", "InvalidParameter": "AppID", "ErrorMessage":
"The AppID is a required parameter." }
```

### 5.2. /messages

**Description**

This function returns a list of previously stored messages that match the given search criteria, including those messages stored by the given user and any other users. It allows a client to find messages associated with nearby locations.

#### Usage

This function is invoked as an HTTP GET with parameters sent as query values, as defined in **[[RFC 3986]](#references)**.

#### Input Parameters

**`AppID` : Base-32 SHA1 URN, as described in [[RFC 3548]](#references) and [[RFC 2141]](#references)**

A unique client-generated hash identifying the application that is calling this function. Only those messages with the same `AppID` value are returned. The hash should be of the form: form: `urn:sha1:**hashvalue**`. Example: `urn:sha1:YNCKHTQCWBTRNJIV4WNAE52SJUQCZO5C`

**`PrivateUserID` : Base-32 SHA1 URN**

A unique client-generated hash identifying the user that is calling this function. The hash should be of the form: form: `urn:sha1:**hashvalue**`. Example: `urn:sha1:PLSTHIPQGSSZTS5FJUPAKUZWUGYQYPFB`

**`Latitude`: degrees as double precision number, as described in [[WGS 84]](#references)**

The latitude of the location where the message search is centered, usually the client's current position as determined by GPS or other sources. The latitude should be positive for latitudes north of the equator and negative for latitudes south of the equator. Example: `47.60638`

**`Longitude`: degrees as double precision number**

The longitude of the location where the message search is centered, usually the client's current position as determined by GPS or other sources. The longitude should be positive for longitudes east of the prime meridian and negative for longitudes west of the prime meridian. Example: `-122.33083`

**`Altitude`: meters as a double precision number**

The altitude above sea level of the location where the message search is centered, usually the client's current position as determined by GPS or other sources. Example: `44.43`

This value is optional and defaults to an undefined value, meaning that the altitude is unknown.

**`Accuracy`: meters as a positive double precision number, from 0 to 10000**

The radius of uncertainty around the location defined by the given `Latitude` and `Longitude` values. `Accuracy` does not take `Altitude` into account. Example: `27`

This value is optional and defaults to 0.

**`DesiredAccuracy`: meters as a positive double precision number**

The maximum `Accuracy` of any message that is to be returned. In other words, if the `DesiredAccuracy` is 100, then those messages with an `Accuracy` greater than 100 will not be returned.

This value is optional and defaults to an undefined value, meaning that messages of any `Accuracy` are to be returned.

**`DesiredRange`: meters as a positive double precision number, from 0 to 10000**

The maximum distance from the given `Location` of any message that is to be returned. In other words, if the `DesiredRange` is 200, then those messages with a `Location` greater than 200 meters away from the given `Location` will not be returned. `DesiredRange` does not take `Altitude` into account in this distance calculation. It does however take the given `Accuracy` into account by effectively increasing the `DesiredRange` value.

**`DesiredTimeBegin`: internet time stamp in UTC, as described in [[RFC 3339]](#references)**

The earliest `Time` of any message that is to be returned. In other words, if the `DesiredTimeBegin` is 2008-12-19T16:39:57Z, then those messages with a `Time` earlier than 4:39:57 p.m. on December 19, 2008 UTC will not be returned.

This value is optional and defaults to an undefined value, meaning that begin `Time` filtering is not performed on returned messages. However, a `DesiredTimeEnd` parameter may still be provided.

**`DesiredTimeEnd`: internet time stamp in UTC**

The latest `Time` of any message that is to be returned. In other words, if the `DesiredTimeBegin` is 2008-12-19T18:39:57Z, then those messages with a `Time` later than 6:39:57 p.m. on December 19, 2008 UTC will not be returned.

If both `DesiredTimeBegin` and `DesiredTimeEnd` are given, and `DesiredTimeEnd` occurs chronologically before `DesiredTimeBegin`, an error is returned.

This value is optional and defaults to an undefined value, meaning that end `Time` filtering is not performed on returned messages. However, a `DesiredTimeStart` parameter may still be provided.

**`Format`**: **string, 1 to 64 characters**

A string indicating the requested format of the returned parameters. In this version of the API, the only possible value is Json, meaning that the returned parameters will be formatted as a JSON object. JSON is described in **[[RFC 4627]](#references)**. Future versions of the API could support additional formats such as RSS.

This value is optional and defaults to Json.

**`MajorVersion`: string, 1 to 64 characters**

The major version of the API in use by the client. Ignored in the initial version of the API, but can be used in future versions to signal incompatible version differences for backwards compatibility.

This value is optional and defaults to an undefined value.

**`MinorVersion`: string, 1 to 64 characters**

The minor version of the API in use by the client. Ignored in the initial version of the API, but can be used in future versions to signal compatible version differences for backwards compatibility.

This value is optional and defaults to an undefined value.

## Return Parameters

This function returns a JSON object (dictionary) within the contents of the HTTP response body. JSON is described in **[[RFC 4627]](#references)**. This object includes a list of those messages, if any, that match the given search criteria. Note that the parameters within the JSON object and all contained objects may occur in any order.

`Result`: **JSON string, 0 to 64 characters**

A string indicating success or failure of retrieving messages. Possible values are `OK` and `Error.`

**`InvalidParameter`: JSON string, 0 to 64 characters**

The name of the input parameter that is invalid, if any. `InvalidParameter` is only included if the `Result` value is `Error` and there was an invalid input parameter. If there are multiple invalid parameters, then only one of them is returned.

**`ErrorMessage`: JSON string, 0 to 140 characters**

A human-readable description of the error that occurred when retrieving messages, if any. `ErrorMessage` is only included if the `Result` value is `Error.` If there were multiple errors, then only one of them is returned.

**`Messages`: JSON array of objects**

A list of objects describing those messages that match the criteria specified in the given input parameters. Each element of the list contains the following parameters:

**`UniqueUserID`: JSON string containing a Base-32 SHA1 URN**

Unique identifier of the publisher of the message. This ID is comprised of a hash of the following: The message's `PrivateUserID` plus a salted secret key known only to the service. A user can determine what their own `UniqueUserID` is by calling the **`/users/PrivateUserID`** function before storing any messages. See the section on ID Generation for more information.

**`Message`: JSON string, 0 to 140 characters**

The text of a message matching the search criteria.

**`Time`: JSON string containing an internet time stamp in UTC**

The UTC date and time associated with the message.

**`Latitude`: JSON number containing degrees as double precision number**

Latitude associated with the message.

**`Longitude`: JSON number containing degrees as double precision number**

Longitude associated with the message.

**`Altitude`: JSON number containing meters as double precision number**

Altitude above sea level associated with the message. If the `Altitude` parameter is omitted, that means it is undefined for the message.

The `Messages` value is only included when there is no error. If there are no messages matching the given search criteria, then the `Messages` list is simply empty.

**`TimeBegin`: internet time stamp in UTC**

The earliest possible time of any returned message, or `DesiredTimeBegin`, whichever is later. In other words, the `TimeBegin` value indicates the earliest time at which messages stored on the service can begin, taking into account the automatic purging of ephemeral messages. If, for example, the service stores messages for a maximum of 28 hours before they are purged, then the `TimeBegin` value will be at most 28 hours before the current time.

The `TimeBegin` value is only included when there is no error. It is returned regardless of whether the `Messages` list is empty.

#### Throttling

The service may perform rate limiting on calls to this function based on client IP address, `AppID` , and/or `PrivateUserID` . When throttling occurs, more restrictive thresholds may temporarily be put into place for the client's IP or `PrivateUserID` .

See the **[Request Throttling](#request_throttling)** section for more information.

#### Availability

Available in Ephemeral Locality Service API version 0.1 and later.

#### Example Request

```
GET /messages?AppID=urn:sha1:YNCKHTQCWBTRNJIV4WNAE52SJUQCZO5C&
PrivateUserID=urn:sha1:PLSTHIPQGSSZTS5FJUPAKUZWUGYQYPFB&
Latitude=47.60638&Longitude=-122.33083&Altitude=44.43&Accuracy=27&
DesiredAccuracy=100&DesiredRange=200
&DesiredTimeBegin=2008-12-19T16:39:57Z
&DesiredTimeEnd=2008-12-19T18:39:57Z&
Format=Json&MajorVersion=1&MinorVersion=0
HTTP/1.1 Host: example.com User-Agent: EphemeralLocalityClient/1.0
```

#### Example Response with Matches

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 489 { "Result": "OK", "Messages": [ { "UniqueUserID": "urn:sha1:LUPFJGZ5BTYUFSPTWUSASZQIYGPKQHPS",
"Message": "My Party Is Here at 8pm", "Time": "2008-12-19T16:39:57Z", "Latitude": 47.60638, "Longitude": -122.33083,
"Altitude": 44.43 }, { "UniqueUserID": "urn:sha1:WIZPKASQFYUGYSSJQTGBLPPPS5UHUZFT", "Message":
"Looking for a party between 7 and 10pm", "Time": "2008-12-19T17:06:21Z", "Latitude": 47.60612,
"Longitude": -122.33022, "Altitude": 61.72 } ], "TimeBegin": "2008-12-19T16:39:57Z" }
```

#### Example Response with No Matches

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 72 { "Result": "OK", "Messages": [], "TimeBegin": "2008-12-19T16:39:57Z" }
```

#### Example `Error` Response**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 111 { "Result": "Error", "ErrorMessage": "The data store is unavailable, so the messages could not be searched." }
```

**Example `Error` Response for Invalid Input**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 105
{ "Result": "Error", "InvalidParameter": "AppID", "ErrorMessage": "The AppID is a required parameter." }
```

### 5.3. /apps/AppID

#### Description

This function allows the client to check the uniqueness of a newly generated `AppID` . The idea is that when the client creates a new `AppID` , it can use this function to ensure that the service has not already seen that `AppID` associated with any previous messages, thereby preventing `AppID` collisions.

Note that there is an unlikely but real "time of check / time of use" race condition with this function. Conceivably, after one client calls`/apps/**AppID**`to check the uniqueness of a particular `AppID` , another client could then start using that `AppID` before the first client uses it.

Implementation note: For purposes of supporting this API function, it may make sense for the service to track `AppID`s longer than the ephemeral duration of messages.

#### Usage

This function is invoked as an HTTP GET with its `AppID` parameter as part of the URL and other parameters in query form, as defined in **[[RFC 3986]](#references)**.

#### Input Parameters

**`AppID` : Base-32 SHA1 URN, as described in [[RFC 3548]](#references) and [[RFC 2141]](#references)**

A client-generated hash identifying the application that is calling this function. This is the value that is to be tested for uniqueness. The hash should be of the form: form: `urn:sha1:**hashvalue**`. Example: `urn:sha1:YNCKHTQCWBTRNJIV4WNAE52SJUQCZO5C`

**`MajorVersion`: string, 1 to 64 characters**

The major version of the API in use by the client. Ignored in the initial version of the API, but can be used in future versions to signal incompatible version differences for backwards compatibility.

This value is optional and defaults to an undefined value.

**`MinorVersion`: string, 1 to 64 characters**

The minor version of the API in use by the client. Ignored in the initial version of the API, but can be used in future versions to signal compatible version differences for backwards compatibility.

This value is optional and defaults to an undefined value.

#### Return Parameters

This function returns a JSON object (dictionary) within the contents of the HTTP response body. JSON is described in **[[RFC 4627]](#references)**. Note that the parameters within the JSON object may occur in any order.

**`Result`: JSON string, 0 to 64 characters**

A string indicating success or failure of the function. Possible values are OK and `Error.`

**`InvalidParameter`: JSON string, 0 to 64 characters**

The name of the input parameter that is invalid, if any. `InvalidParameter` is only included if the `Result` value is `Error` and there was an invalid input parameter. If there are multiple invalid parameters, then only one of them is returned.

**`ErrorMessage`: JSON string, 0 to 140 characters**

A human-readable description of the error that occurred, if any. `ErrorMessage` is only included if the `Result` value is `Error`. If there were multiple errors, then only one of them is returned.

**`Exists`: JSON string, 0 to 64 characters**

A string indicating whether the service has already seen a message with the given `AppID`. Possible values are `True` and `False`. The `Exists` value is only included when there is no error.

#### Throttling

The service may perform rate limiting on calls to this function based on client IP address and/or `AppID` . When throttling occurs, more restrictive thresholds may temporarily be put into place for the client's IP.

See the **[Request Throttling](#request_throttling)** section for more information.

#### Availability

Available in Ephemeral Locality Service API version 0.1 and later.

#### Example Request

```
GET /apps/urn:sha1:YNCKHTQCWBTRNJIV4WNAE52SJUQCZO5C&MajorVersion=1&MinorVersion=0 HTTP/1.1
```

`Host: `[example.com](http://example.com/)

```
User-Agent: EphemeralLocalityClient/1.0
```

#### Example Response with Pre-Existing `AppID`**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 37
{ "Result": "OK", "Exists": "True" }
```

#### Example Response with Unique `AppID`**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 38
{ "Result": "OK", "Exists": "False" }
```

#### Example `Error` Response**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 107 { "Result": "Error", "ErrorMessage": "The data store is unavailable, so uniqueness could not be tested." }
````

### 5.4. /users/PrivateUserID

#### Description

This function allows the client to check the uniqueness of a newly generated `PrivateUserID` . The idea is that when the client creates a new `PrivateUserID`, it can use this function to ensure that the service has not already seen that `PrivateUserID` associated with any currently stored messages, thereby preventing `PrivateUserID` collisions. This function also allows a user to determine the `UniqueUserID` that corresponds to their own `PrivateUserID` .

Note that there is an unlikely but real "time of check / time of use" race condition with this function. Conceivably, after one client calls **`/users/PrivateUserID`** to check the uniqueness of a particular `PrivateUserID` , another client could then start using that `PrivateUserID` before the first client uses it.

#### Usage

This function is invoked as an HTTP GET with its `PrivateUserID` parameter as part of the URL and other parameters in query form, as defined in **[[RFC 3986]](#references)**.

#### Input Parameters

**`PrivateUserID` : Base-32 SHA1 URN, as described in [[RFC 3548]](#references) and [[RFC 2141]](#references)**

A client-generated hash identifying the user that is calling this function. This is the value that is to be tested for uniqueness. The hash should be of the form: form: `urn:sha1:**hashvalue**`. Example: `urn:sha1:PLSTHIPQGSSZTS5FJUPAKUZWUGYQYPFB`

**`AppID`: Base-32 SHA1 URN**

A client-generated hash identifying the application that is calling this function. The hash should be of the form: form: `urn:sha1:**hashvalue**`. Example: `urn:sha1:YNCKHTQCWBTRNJIV4WNAE52SJUQCZO5C`

**`MajorVersion`: string, 1 to 64 characters**

The major version of the API in use by the client. Ignored in the initial version of the API, but can be used in future versions to signal incompatible version differences for backwards compatibility.

This value is optional and defaults to an undefined value.

**`MinorVersion`: string, 1 to 64 characters**

The minor version of the API in use by the client. Ignored in the initial version of the API, but can be used in future versions to signal compatible version differences for backwards compatibility.

This value is optional and defaults to an undefined value.

#### Return Parameters

This function returns a JSON object (dictionary) within the contents of the HTTP response body. JSON is described in **[[RFC 4627]](#references)**. Note that the parameters within the JSON object may occur in any order.

**`Result`: JSON string, 0 to 64 characters**

A string indicating success or failure of the function. Possible values are `OK` and `Error`.

**`UniqueUserID`: Base-32 SHA1 URN**

Unique identifier of the publisher of the message. This ID is comprised of a hash of the following: The `PrivateUserID` input parameter plus a salted secret key known only to the service. See the section on ID Generation for more information. The `UniqueUserID` value is only included when there is no error and the value of the Exists return parameter is `False`.

**`InvalidParameter`: JSON string, 0 to 64 characters**

The name of the input parameter that is invalid, if any. `InvalidParameter` is only included if the `Result` value is `Error` and there was an invalid input parameter. If there are multiple invalid parameters, then only one of them is returned.

**`ErrorMessage`: JSON string, 0 to 140 characters**

A human-readable description of the error that occurred, if any. `ErrorMessage` is only included if the `Result` value is `Error.` If there were multiple errors, then only one of them is returned.

**`Exists`: JSON string, 0 to 64 characters**

A string indicating whether the service has a currently stored message with the given `PrivateUserID` . Possible values are `True` and `False`. The `Exists` value is only included when there is no error.

#### Throttling

The service may perform rate limiting on calls to this function based on client IP address, `AppID` , and/or PrivateUserID` . When throttling occurs, more restrictive thresholds may temporarily be put into place for the client's IP or `PrivateUserID` .

See the **[Request Throttling](#request_throttling)** section for more information.

#### Availability

Available in Ephemeral Locality Service API version 0.1 and later.

#### Example Request

```
GET /apps/urn:sha1:PLSTHIPQGSSZTS5FJUPAKUZWUGYQYPFB&AppID=urn:sha1:
YNCKHTQCWBTRNJIV4WNAE52SJUQCZO5C&MajorVersion=1&MinorVersion=0 HTTP/1.1 Host: example.com User-Agent: EphemeralLocalityClient/1.0
```

#### Example Response with Pre-Existing `PrivateUserID`**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 37
{ "Result": "OK", "Exists": "True" }
```

#### Example Response with Unique `PrivateUserID`**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 100
{ "Result": "OK", "UniqueUserID": "urn:sha1:LUPFJGZ5BTYUFSPTWUSASZQIYGPKQHPS", "Exists": "False" }
```

#### Example `Error` Response**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 107
{ "Result": "Error", "ErrorMessage": "The data store is unavailable, so uniqueness could not be tested." }
```

## 6. Security Considerations

### 6.1. Eavesdropping Attacks

While the API is not vulnerable to eavesdropping of authentication credentials, as it doesn't include any authentication, the `PrivateUserID` parameter is a secret shared between the client and the service. Thus, a successful eavesdropping attack could allow an attacker to determine the client's `PrivateUserID` and re-use it in their own messages, thereby publishing messages as that user.

This attack can be prevented by using SSL for the connection between the client and the service.

### 6.2. Man-in-the-Middle Attacks

If DNS resolution or the transport layer is compromised, an attacker could masquerade as the service itself, receiving requests from clients and potentially forwarding them on to the actual service. This would allow the attacker unfettered access to all requests and responses, including the secret `PrivateUserID` parameter of each client. The attacker would also be able to return false or inaccurate responses to API requests, such as changing the location data associated with a message or inserting fake messages altogether.

These sorts of attacks can all be prevented by using SSL with a trusted authority for the connection between the client and the service.

#### 6.3. Service Compromise

If the service itself is compromised by an attacker, then in addition to the considerations mentioned in the previous section, the attacker would also have direct access to all stored messages without having to go through the API. This would bypass some of the API's safeguards intended to protect anonymity, such as request rate limiting and the inability to directly track a particular `UniqueUserID` . However, due to the ephemeral nature of the stored messages, any such attack would only allow the retrieval of messages that have not yet expired.

The chance of such an attack succeeding can be lessened by locking down the server or servers running the service and engaging in security best practices.

#### 7. References

- Josefsson, S., Ed., “**[The Base16, Base32, and Base64 Data Encodings](http://www.ietf.org/rfc/rfc3548.txt)**,” RFC 3548

- Moats, R., “**[URN Syntax](http://www.ietf.org/rfc/rfc2141.txt)**,” RFC 2141

- Fielding, R., Gettys, J., Mogul, J., Frystyk, H., Masinter, L., Leach, P., and T. Berners-Lee, “**[Hypertext Transfer Protocol -- HTTP/1.1](http://www.ietf.org/rfc/rfc2616.txt)**,” RFC 2616

- Newman, C. and Klyne, G., “**[Date and Time on the Internet: Timestamps](http://www.ietf.org/rfc/rfc3339.txt)**,” RFC 3339

- DoD, 1997, “**[World Geodetic System 1984: Its Definition and Relationships with Local Geodetic Systems – Third Edition](http://earth-info.nga.mil/GandG/publications/tr8350.2/tr8350_2.html)**,” Technical Report TR8350.2

- Berners-Lee, T., “**[Uniform Resource Identifiers (URI): Generic Syntax](http://www.ietf.org/rfc/rfc3986.txt)**,” RFC 3986

- Crockford, D., “**[JavaScript Object Notation (JSON)](http://www.ietf.org/rfc/rfc4627.txt)**,” RFC 4627

## 8. API Change Log

0.1: Initial draft

0.2: Application notes section, new **`/messages`** parameters, limits on search radius, and details on throttling

0.3: ID Generation section, new **`/messages`** `UniqueUserID` return parameter to replace `PrivateUserID`

0.4: `AppID` generation clarifications, `PrivateUserID` and `UniqueUserID` clarifications, new **`/users/PrivateUserID`** `UniqueUserID` return parameter, UserID parameter renamed to `PrivateUserID`, PublisherID parameter renamed to `UniqueUserID` , throttling thresholds removed, table of contents, references section, security considerations section

## 9. Authors' Addresses

- Christopher Allen
   Email: ChristopherA@LifeWithAlacrity.com

- Dan Helman
  Email: [dan@luminotes.com](mailto:dan@luminotes.com)
