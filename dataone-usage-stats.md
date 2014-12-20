---
layout: page
title: DataONE Usage statistics for MDC
permalink: /dataone-usage-stats/
---

The MDC project will utilize the [DataONE](http://www.dataone.org) federation of data repositories as a source for data usage statistics.  DataONE federates a large collection of data sets across a couple of dozen data repositories and provides a common metadata index across these.  In addition, DataONE federates usage statistics for all of these repositories, making it easy to calculate total downloads for any object across the whole federation.  These design notes provide some background in how to use the various DataONE APIs to access DataONE content and statistics.

The PLOS Lagotto system needs particular types of access to DataONE:

* Basic metadata for each dataset: identifier, identifier_type, title, publication_date (year or year-month is fine), member node
* optional additional metadata that describes the relationship between works, e.g. isNewVersion (I’m using the DataCite relation type categories for this)
* a way to query a DataONE API to give me this information by member node, publication_date and/or update_date. The latter is needed so that I can run a daily cron job that automatically imports new/updated works.

There are several ways to get at all of this information.  DataONE exposes three main sources of information on data sets (SystemMetadata, ScienceMetadata, and ResourceMaps), each of which can be retrieved with an API call if you know the identifier of a data set.  Here are the REST calls needed to get information for an example data set from the KNB member node.

## Example data set from the KNB 

* __Bennington College, and Huron Mt. Wildlife Foundation and Woods K.__ 2014. *Long-term tree demography in old-growth forests of Huron Mts, MI from permanent-plot censuses, 1962-2009* (doi:10.5063/F1PC3085)

* To get SystemMetadata:

{% highlight sh %}
https://cn.dataone.org/cn/v1/meta/doi:10.5063/F1PC3085
{% endhighlight %}

* To get ScienceMetadata:

{% highlight sh %}
https://cn.dataone.org/cn/v1/object/doi:10.5063/F1PC3085
{% endhighlight %}

* To get the ResourceMap (which describes the components of a data package): 

{% highlight sh %}
https://cn.dataone.org/cn/v1/object/urn:uuid:5a721c8b-94b1-4c33-9ab2-d99f963a4215
{% endhighlight %}

DataONE also indexes most the metadata provided in these individual documents in a SOLR index, so you can issue a SOLR query to get most of the info you want for a single document:

{% highlight sh %} https://cn.dataone.org/cn/v1/query/solr/?fl=id,title,formatId,formatType,documents,resourceMap,obsoletes,obsoletedBy,datePublished,authoritativeMN,replicaMN,dateUploaded,dateModified&q=id:doi\:10.5063/F1PC3085
{% endhighlight %}

And here's the result:

{% highlight xml %}
<response>
  <lst name="responseHeader">
    <int name="status">0</int>
    <int name="QTime">6</int>
    <lst name="params">
      <str name="fl">id,title,formatId,formatType,documents,resourceMap,obsoletes,obsoletedBy,datePublished,authoritativeMN,replicaMN,dateUploaded,dateModified\</str>
      <str name="q">id:doi\:10.5063/F1PC3085</str>
    </lst>
  </lst>
  <result name="response" numFound="1" start="0">
    <doc>
      <str name="authoritativeMN">urn:node:KNB</str>
      <date name="datePublished">2014-09-22T00:00:00Z</date>
      <date name="dateUploaded">2014-09-22T17:55:59.848Z</date>
      <arr name="documents">
        <str>knb.498.1</str>
      </arr>
      <str name="formatId">eml://ecoinformatics.org/eml-2.1.1</str>
      <str name="formatType">METADATA</str>
      <str name="id">doi:10.5063/F1PC3085</str>
      <str name="obsoletes">knb.499.1</str>
      <arr name="replicaMN">
        <str>urn:node:KNB</str>
        <str>urn:node:CN</str>
        <str>urn:node:mnUCSB1</str>
        <str>urn:node:mnUNM1</str>
        <str>urn:node:TFRI</str>
      </arr>
      <arr name="resourceMap">
        <str>urn:uuid:5a721c8b-94b1-4c33-9ab2-d99f963a4215</str>
      </arr>
      <str name="title">Long-term tree demography in old-growth forests of Huron Mts, MI from permanent-plot censuses, 1962-2009</str>
    </doc>
  </result>
</response>
{% endhighlight %}

Of course, because its SOLR, you can also ask for the same info for a range of objects.  For example, the following query gives some metadata for all objects that have had a SystemMetadata modification in the last week.  SystemMetadata gets modified anytime an important event happens to an object, such as an access control change, or if it is obsoleted by a new version, or a new replica is created.

{% highlight sh %} https://cn.dataone.org/cn/v1/query/solr/?fl=id,title,formatId,formatType,documents,resourceMap,obsoletes,obsoletedBy,datePublished,authoritativeMN,replicaMN,dateUploaded,dateModified&q=dateModified:[NOW-7DAYS/DAY%20TO%20NOW]
{% endhighlight %}

You could add more filter criteria, such as restrict it to only documents on one node (e.g., `datasource:urn\:node\:DRYAD`).

## Getting download statistics for each object:

We also record download stats (and download size) in another SOLR index that I mentioned in previous posts, and which are described in [DataONE Usage Statistics](http://jenkins-1.dataone.org/jenkins/job/API%20Documentation%20-%20trunk/ws/api-documentation/build/html/design/UsageStatistics.html).  For the KNB data set above, you could get monthly download stats using:

{% highlight sh %} https://cn.dataone.org/cn/v1/query/logsolr/select?q=pid:doi\:10.5063/F1PC3085&fq=event:read&facet=true&facet.range=dateLogged&facet.range.start=2014-01-01T01:01:01Z&facet.range.end=2014-12-31T24:59:59Z&facet.range.gap=%2B1MONTH
{% endhighlight %}

Of course, it's likely you'll want download statistics for each object that has been accessed on a given day.  As an example, just facet by the object identifier and query the for log events from the past week (not including today):

{% highlight sh %} https://cn.dataone.org/cn/v1/query/logsolr/select?q=event:read+dateLogged:[NOW-7DAYS/DAY%20TO%20NOW]&facet=true&facet.field=pid&facet.mincount=1&facet.limit=-1
{% endhighlight %}

Lagotto will more likely want to query for just a single day to get finer-grained counts, and update its index daily.  Note that this query returns a count for each pid in the time period, and that one will need to step through multiple pages of results by incrementing facet.offset (possibly in pages of 1000), or get all of the results in one page by setting facet.limit=-1.

## Getting download statistics for each Member Node

For the KNB node, you could see monthly download statistics using:

{% highlight sh %}
https://cn.dataone.org/cn/v1/query/logsolr/select?q=nodeId:urn\:node\:KNB&fq=event:read&facet=true&facet.range=dateLogged&facet.range.start=2000-01-01T01:01:01Z&facet.range.end=2014-12-31T24:59:59Z&facet.range.gap=%2B1MONTH
{% endhighlight %}

And the same stats for Dryad:

{% highlight sh %}
https://cn.dataone.org/cn/v1/query/logsolr/select?q=nodeId:urn\:node\:DRYAD&fq=event:read&facet=true&facet.range=dateLogged&facet.range.start=2000-01-01T01:01:01Z&facet.range.end=2014-12-31T24:59:59Z&facet.range.gap=%2B1MONTH
{% endhighlight %}

## Getting information about Member Nodes

DataONE returns a Member Node `nodeId` in many data structures, such as `urn:node:KNB`. This is the permanent, unique, and unchanging identifier for the node, but is not particularly human readable.  DataONE also provides a node registry that provides additional metadata about Member Nodes that Lagotto might want to use in user interface displays, such as the node name.  The node registry can be accessed at:

{% highlight sh %}
https://cn.dataone.org/cn/v1/node
{% endhighlight %}

and produces output like:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="/cn/xslt/dataone.types.v1.xsl"?>
<d1:nodeList xmlns:d1="http://ns.dataone.org/service/types/v1">
  <node replicate="true" synchronize="true" type="mn" state="up">
    <identifier>urn:node:KNB</identifier>
    <name>Knowledge Network for Biocomplexity</name>
    <description>The Knowledge Network for Biocomplexity (KNB) is a national network intended to facilitate ecological and environmental research on biocomplexity.</description>
    <baseURL>https://knb.ecoinformatics.org/knb/d1/mn</baseURL>
    <services>
      <service name="MNRead" version="v1" available="true"/>
      <service name="MNCore" version="v1" available="true"/>
      <service name="MNAuthorization" version="v1" available="true"/>
      <service name="MNStorage" version="v1" available="true"/>
      <service name="MNReplication" version="v1" available="true"/>
    </services>
    <synchronization>
      <schedule hour="*" mday="*" min="0/3" mon="*" sec="10" wday="?" year="*"/>
      <lastHarvested>2014-12-19T17:45:04.271+00:00</lastHarvested>
      <lastCompleteHarvest>1900-01-01T00:00:00.000+00:00</lastCompleteHarvest>
    </synchronization>
    <subject>CN=urn:node:KNB,DC=dataone,DC=org</subject>
    <contactSubject>CN=ben leinfelder A756,O=Google,C=US,DC=cilogon,DC=org</contactSubject>
  </node>
  <node replicate="false" synchronize="true" type="mn" state="up">
    <identifier>urn:node:ESA</identifier>
    <name>ESA Data Registry</name>
    <description>The data sets registered here are associated with articles published in the journals of the Ecological Society of America.</description>
    <baseURL>https://data.esa.org/esa/d1/mn</baseURL>
    <services>
      <service name="MNRead" version="v1" available="true"/>
      <service name="MNCore" version="v1" available="true"/>
      <service name="MNAuthorization" version="v1" available="true"/>
      <service name="MNStorage" version="v1" available="true"/>
      <service name="MNReplication" version="v1" available="true"/>
    </services>
    <synchronization>
      <schedule hour="*" mday="*" min="0/3" mon="*" sec="10" wday="?" year="*"/>
      <lastHarvested>2014-03-30T03:50:10.138+00:00</lastHarvested>
      <lastCompleteHarvest>1900-01-01T00:00:00.000+00:00</lastCompleteHarvest>
    </synchronization>
    <subject>CN=urn:node:ESA,DC=dataone,DC=org</subject>
    <contactSubject>CN=ben leinfelder A756,O=Google,C=US,DC=cilogon,DC=org</contactSubject>
  </node>
  <node replicate="false" synchronize="true" type="mn" state="up">
    <identifier>urn:node:USGSCSAS</identifier>
    <name>USGS Core Sciences Clearinghouse</name>
    <description>US Geological Survey Core Science Metadata Clearinghouse archives metadata records describing datasets largely focused on wildlife biology, ecology, environmental science, temperature, geospatial data layers documenting land cover and stewardship (ownership and management), and more.</description>
    <baseURL>http://mercury-ops2.ornl.gov/clearinghouse/mn</baseURL>
    <services>
      <service name="MNCore" version="v1" available="true"/>
      <service name="MNRead" version="v1" available="true"/>
      <service name="MNAuthorization" version="v1" available="false"/>
      <service name="MNStorage" version="v1" available="false"/>
      <service name="MNReplication" version="v1" available="false"/>
    </services>
    <synchronization>
      <schedule hour="*" mday="*" min="0,5,10,15,20,25,30,35,40,45,50,55" mon="*" sec="0" wday="?" year="*"/>
      <lastHarvested>2014-01-31T15:29:30.000+00:00</lastHarvested>
      <lastCompleteHarvest>1900-01-01T00:00:00.000+00:00</lastCompleteHarvest>
    </synchronization>
    <subject>CN=Robert Nahf A579,O=Google,C=US,DC=cilogon,DC=org</subject>
    <subject>CN=csas,DC=cilogon,DC=org</subject>
    <subject>CN=Giriprakash Palanisamy T864,O=Google,C=US,DC=cilogon,DC=org</subject>
    <contactSubject>CN=Robert Nahf A579,O=Google,C=US,DC=cilogon,DC=org</contactSubject>
  </node>
  <node replicate="false" synchronize="true" type="mn" state="up">
    <identifier>urn:node:CDL</identifier>
    <name>Merritt Repository</name>
    <description>A cross-domain repository for data and metadata provided by the California Digital Library at the University of California</description>
    <baseURL>https://merritt.cdlib.org:8084/knb/d1/mn</baseURL>
    <services>
      <service name="MNCore" version="v1" available="true"/>
      <service name="MNRead" version="v1" available="true"/>
      <service name="MNAuthorization" version="v1" available="false"/>
      <service name="MNStorage" version="v1" available="true"/>
      <service name="MNReplication" version="v1" available="false"/>
    </services>
    <synchronization>
      <schedule hour="23" mday="*" min="00" mon="*" sec="00" wday="?" year="*"/>
      <lastHarvested>2014-12-11T23:12:03.884+00:00</lastHarvested>
      <lastCompleteHarvest>1900-01-01T00:00:00.000+00:00</lastCompleteHarvest>
    </synchronization>
    <subject>CN=urn:node:CDL,DC=dataone,DC=org</subject>
    <contactSubject>CN=Mark Reyes A5620,O=University of California - Office of the President,C=US,DC=cilogon,DC=org</contactSubject>
  </node>
  ...
</d1:nodeList>
{% endhighlight %}


