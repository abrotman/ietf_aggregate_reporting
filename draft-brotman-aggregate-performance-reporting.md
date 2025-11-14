%%%
Title = "Aggregate Performanc Reporting"
abbrev = "APR"
category = "std"
docName = "draft-brotman-aggregate-performance-reporting-00"
ipr = "trust200902"
area = "Applications"
keyword = [""]

date = 2025-11-15T00:00:00Z

[seriesInfo]
name = "Internet-Draft"
value = "draft-brotman-aggregate-performance-reporting-03"
stream = "IETF"
status = "standard"

[[author]]
initials="A."
surname="Brotman"
fullname="Alex Brotman"
organization="Comcast, Inc"
 [author.address]
 email="alex_brotman@comcast.com"

[[author]]
initials="T."
surname="Corbett"
fullname="Tom Corbett"
Organization="Iterable"
  [author.address]
  email="tom.corbett@iterable.com"

[[author]]
initials="E."
surname="Gustafsson"
fullname="Emil Gustafsson"
Organization="Google"
  [author.address]
  email="emgu@google.com"

%%%

.# Abstract

Definition of an aggregate performance report format for email messaging, the
means to discover target destinations, and a specified delivery method.

{mainmatter}

# Introduction

In the Email industry, there is a great amount of emphasis put upon various 
types of reputation.  Mailbox Providers (MBP) typically use these reputation 
systems to govern placement, permitted volume, or other characteristics 
related to delivery.  In many cases, the entities sending the messages have 
little insight into these reputation systems that are directly impacted by 
their actions. That lack of usable information directly impacts their 
ability to take future corrective actions.

A report which contains some elements of reputation and metrics associated 
with reputation change could allow those sending entities to have more 
insight into how their actions impact some of these reputation systems.

Proposed below will be a document format, as well as methods for report 
destination discovery and delivery methods.


# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
[@?RFC2119].

# Background

# Glossary

DKIM - DomainKeys Identified Mail
MBP - Mailbox Provider

# Destination Discovery

In order to discover where the reports will be delivered, the report generator 
will perform a DNS lookup against the domain used with the valid DKIM 
signatures.  The lookup will also leverage the selector.  An example signature 
might be:

DKIM-Signature: v=1;a=rsa-sha256;c=relaxed/relaxed;s=sel1;d=foo.example.org;...

And the resulting TXT record would be:

sel1._aprf._domainkey.foo.example.org 

There is the option to use a wildcard [RFC1034] to the left of the '_aprf' 
label.  This would use the same record for all selectors, unless specifically 
stated in the DNS system. 

# Record Attributes

v:   This MUST always exist, and the value must always be "APRFv1".  Failure 
     to do so should be treated as an invalid record, and the record MUST 
     be ignored.

rua: This is the destination address for the report data.  The value must 
     be prefaced with a "mailto:", and then include a valid 5321 address 
     as the destination.  If there is more than one destination, they 
     should be separated with a comma, and each should have its own 
     "mailto:".  Absence of this attribute suggests the signing entity has 
     no interest in receiving reports, and the record MUST be ignored.

sdi: An optional attribute which helps segment the data. The contents are 
     a header and separator character, separated by a comma.  Defined in 
     the section below.

# Signer-Defined Identifiers (SDI)

There is an attribute ('sdi') by which the DKIM domain holder can share a 
header name which can help segment the data contained within the report.  
The attribute is defined in two parts, separated by a comma (",").  The 
parts MUST be the RFC5322 Header Name, and then a single character 
separator.  The separator MUST be a printable ASCII [RFC20] character, 
and MUST NOT be ";", "=", or ",".

The header field MUST be DKIM signed by the related signature.

The information contained within is expected to be increasing in granularity 
as it moves to the right, with a maximum of four fields.  There MUST be a 
limit of four parts within the header.

Example:

sel1._aprf._domainkey.example.org TXT "v=ARPFv1; … ; sdi=Signer-Info,^"

Where the header name is "Signer-Info", and the separator is "^".

 An example header:

Signer-Info:SenderCommonName^BrandName^RegionalDistinction^CampaignName

Use of this by the report generator is optional.  The report generator MAY 
ignore this attribute.

## Note about Usage of SDI

It's quite possible that two signing entities could attempt to use the same 
header field.  This is not explicitly forbidden, however, if each entity 
uses a different separator string, the results will likely be undesirable.  
It is very much suggested that signers choose a header name that is likely 
unique to their entity.

Additionally, a signer should take caution when creating segment names.  These 
segment names may create a data leakage.  Those names could appear in reports.

# DNS Record Samples

v=APRFv1;rua:mailto:reports@example.org;

v=APRFv1;rua:mailto:reports@example.org,mailto:reports2@example2.net;

v=APRFv1;rua:mailto:reports@example.org;sdi=MsgInfo,^;

If one of the destinations does not align with the sending RFC5322.From domain, 
there should be destination validation, discussed below.

# Report Format & Contents

The report format MUST be valid JSON.

## Report Time Period

The report MUST contain data for one UTC day.  It MUST begin at 0000UTC, and 
MUST end at 2359.59UTC on that day.

## Document Contents

### Header

The “header” portion of the report will include data about the entity creating 
the report.  The fields will be:

source:        The common name of the reporting entity.  If a company is 
               reporting on behalf of a MBP, that MBP name should be in 
               this field.

dkim_domain:   The domain which is being reported on.  This should be from 
               the d= field of the message.  If the report is going to roll 
               up data to the higher level domain, the field should show 
               this as an asterisk (e.g., "*.example.org").

dkim_selector: The selector which is being reported on.  If this is an 
               aggregate report that is rolling up data, the reporting 
               entity should make this value an asterisk ("*").

report_start:  The time where the data within the report begins, noted 
               in the epoch format.

report_end:    The time where the data within the report ends, noted in the 
               epoch format.

contact_info:  An email address that may be used to contact the report 
               generator with questions.  This field is optional.

sdi_used:      Demonstrate which SDI was used to create the report 
               data.  The string MUST be in the same format retrieved from 
               DNS. If the report generator is not using the declared 'sdi', 
               this value MUST be "N/A".  If the value is not in the record, 
               or is not valid, the value here MUST be "N/F".

extra_info:    Optional field.  This could be used by the report generator 
               to provide additional information for the report recipient.  
               This could include information such as how to better interpret 
               the reports, contact information, information about data 
               points that may influence reputation.

### Body

The second portion is called the "Body", and will include the data.  Within 
the body, there are a list of segments. Each segment contains a 
"classification", and "engagement" part in addition to a sender defined 
identifier, if applicable.  The "classification" section is meant to disclose 
placement information, while "engagement" is meant to disclose what 
happens after the message is classified, and include "positive", "negative", 
and "neutral" data points. 

Sample default segment (no signer defined identifier):

…
{
"classification": { ... },
"engagement": { ... }
},
...

Sample segment with signer defined identifier:

...
{
"segment": "ExampleCustomerID",
"classification": { ... },
"engagement": { ... }
},
...

# Classification

This segment is meant to disclose the opinion of the MBP that received the 
message.  This could include values such as: inbox, unwanted, forwarded, 
promotional, or some other placement information. These are scalar values, 
and expected to be created as "buckets".  Below 1000, these buckets should 
be buckets of 100.  Above 1000, the bucket should be at 1000 increments.

These metrics should pertain to the reporting period, and measure the number 
of messages in each category that were received during that time. 

Sample segment:

...
"classification":  {
"inbox": 10000,
"unwanted": 500
},
...


The report generator MAY provide information about these categories in the 
"extra_info" header.

# Engagement

This segment is to disclose data around user engagement, and how that may 
impact the reputation.

These metrics should pertain to the reporting period, and measure the number 
of engagement actions in each category, regardless of when the messages were 
received. The values are meant to be bucketed scalar values, similar to 
the metrics in the prior section.

Sample segment:

...
"engagement": {
"positive": 100,
"neutral": 500,
"negative": 50
},
...

The report generator MAY provide information about these categories in the 
"extra_info" header.

# Report Delivery

The reports must be attached to the messages as a standard attachment. 

When using the date below, the date used should be the ending date of the 
report.  For a single UTC day report, the ending date should be that date.

The attachment name MUST be in the form:

<yyyymmdd>_<DKIM domain>_<DKIM selector>_<source>

The suffix for the file MUST be ".json".

The subject MUST be in the form:

ARPF: <yyyymmdd>_<DKIM domain>_<DKIM selector>_<source>

The Content-Type for the message MUST be "multipart/report".  The 
Content-Type for the report attachment MUST be "application/json".  If 
there is a plain-text portion of the report, the Content-Type MUST 
be "text/plain".


Reports will be delivered via SMTP to the destination address.


NOTE: TBD Compression?
NOTE: If a later version shows up with the same date period, does it 
overwrite, discard?  Should there be a flag showing it to be an update?

# Destination Validation

When the report destination and DKIM domain are not aligned, there is a 
method by which the report generator SHOULD validate the report destination 
before attempting delivery.

<selector>.<DKIM domain>._aprf.<report destination domain>

And the value MUST be "v=APRFv1;" or "v=APRFv1"

The <selector> or <DKIM domain> may be a wildcard entry.  This would allow 
all portions to the left to go to that destination.


# Security Considerations

## Data Leakage

### Personal Data

At a sufficient low volume, it may be possible to expose some information 
that would otherwise be "lost in the noise" about individuals.  One 
suggestion may be that the MBP that is supplying the reports set a threshold 
for a number of messages or distinct recipients in order to better obfuscate 
that information.

### Report Data

Report generators may not wish to send reports to all of the destinations 
requested.  The decision on destinations could be related to 5322.From 
alignment, domain reputation, or other considerations.  A report generator 
is not required to send reports to every entity requesting these reports.

# IANA Considerations

# Appendix

## Definition of "Performance"

This report is meant to allow the MBP/report generator to share information 
about metrics relating to performance and reputation.  The metrics that 
are noted above are positive/neutral/negative.  As one may have noticed, 
there is no formal definition of these metrics or what they may 
mean.  Each MBP/report generator may have different actions that belong 
in each of these categories.  

Some examples could be:

positive: open or click engagement, False positive ("not-spam") report
neutral: filing message into a folder, forwarding
negative: marking as spam, deletion without opening, unsubscribe

These could be considered "secret sauce", so it is likely not advantageous 
for the MBP to disclose precisely what each category consists of.

A report generator MAY choose to divulge some or all of this information 
via the "extra_info" field in the report header.


{backmatter}
