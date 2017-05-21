log4stash
=====================

log4stash is a [log4net](http://logging.apache.org/log4net/) appender to log messages to the [ElasticSearch](http://www.elasticsearch.org) document database. ElasticSearch offers robust full-text search engine and analyzation so that errors and messages can be indexed quickly and searched easily.

log4stash provide few logging filters similar to the filters on [logstash](http://logstash.net).

The origin of log4stash is [@jptoto](https://github.com/jptoto)'s [log4net.ElasticSearch](https://github.com/jptoto/log4net.ElasticSearch) repository.

### Features:
* Supports .NET 3.5+
* Easy installation and setup via [Nuget](https://nuget.org/packages/log4stash/)
* Ability to analyze the log event before sending it to elasticsearch using built-in filters and custom filters similar to [logstash](http://logstash.net/docs/1.4.2/).

### Breaking Changes:
* __Upgrading to 2.0.4__: BasicAuthUsername and BasicAuthPassword moved under AuthenticationMethod, see [config example][config-example] for more information.
* __Upgrading to 2.0.0__: The namespace has been changed from _log4net.ElasticSearch_ to _log4stash_ 
* __Upgrading to 1.0.0__: The definition of IElasticAppenderFilter has been changed, PrepareEvent has only one parameter and PrepareConfiguration's parameter type has changed to IElasticsearchClient.

#### Versions notes:
* :green_book: log4stash 2.0.4 has new filters see [filters section][filters-section] for more information. Added support for AWS Version 4 authorization header thanks to [@Marcelo Palladino][mfpalladino]. Support configuring multiple elastic nodes. Add document id source so you can decide your own doc id (instead of getting generated one from the elastic). Add Timeout option for requests. Many thanks to [@eran gil][erangil] for the efforts and the pull requests.

* :green_book: log4stash 2.0.0 has new filter `Json`, you can add it if you have json string in you log event and you want it to be converted to an object and not be passed as string to the elastic. Thanks to [@Ignas Vel�a][ignasv] for this filter.

* :green_book: log4stash 1.1.0 has new feature `SerializeObjects`, if true (the default) it serializes the exception object and message object into json object and add them to Elastic. You can see them under "MessageObject" and "ExceptionObject" keys.  - [Related commit](https://github.com/urielha/log4stash/commit/560676de9b074be70e00f93566c543a846ba5c8e)

### Filters:
* **Add** - add new key and value to the event.
* **Remove** - remove key from the event.
* **Rename** - rename key to another name.
* **Kv** - analyze value (default is to analyze the 'Message' value) and export key-value pairs using regex (similar to logstash's kv filter).
* **Grok** - analyze value (default is 'Message') using custom regex and saved patterns (similar to logstash's grok filter).
* **ConvertToArray** - split raw string to an array by given seperators. 
* :new: **Json** - convert json string to an object (so it will be parsed as object in elasticsearch).
* :new: **Convert** - Available convertors: `ToString`, `ToLower`, `ToUpper`, `ToInt` and `ToArray`. See [config example][config-example] for more information. 
* :new: **Xml** - Parse xml into an object.

#### Custom filter:
To add your own filters you just need to implement the interface IElasticAppenderFilter on your assembly and configure it on the log4net configuration file.

<!-- ### Usage:
Please see the [DOCUMENTATION](https://github.com/urielha/log4stash/wiki/0-Documentation) Wiki page to begin logging errors to ElasticSearch! -->

### Issues:
I do my best to reply to issues or questions ASAP. Please use the [ISSUES](https://github.com/urielha/log4stash/issues) page to submit questions or errors.

### Configuration Examples:

Almost all the parameters are optional, to see the default values check the [c'tor](https://github.com/urielha/log4stash/blob/master/src/log4stash/ElasticSearchAppender.cs#L52) of the appender and the c'tor of every filter. 
You can also set any public property in the appender/filter which didn't appear in the example.

##### Simple configuration:
```xml
<appender name="ElasticSearchAppender" type="log4stash.ElasticSearchAppender, log4stash">
    <Server>localhost</Server>
    <Port>9200</Port>
    <ElasticFilters>
      <!-- example of using filter with default parameters -->
      <kv /> 
    </ElasticFilters>
</appender>
```

##### (Almost) Full configuration:
```xml
<appender name="ElasticSearchAppender" type="log4stash.ElasticSearchAppender, log4stash">
    <Server>localhost</Server>
    <Port>9200</Port>
    <IndexName>log_test_%{+yyyy-MM-dd}</IndexName>
    <IndexType>LogEvent</IndexType>
    <Bulksize>2000</Bulksize>
    <BulkIdleTimeout>10000</BulkIdleTimeout>
    <IndexAsync>False</IndexAsync>
    <DocumentIdSource>IdSource</DocumentIdSource> <!-- obsolete -->

    <!-- optional: elasticsearch timeout for the request, default = 10000 -->
    <ElasticSearchTimeout>10000</ElasticSearchTimeout>
  
    <!--You can add parameters to the request to control the parameters sent to ElasticSearch.
    for example, as you can see here, you can add a routing specification to the appender.
    The Key is the key to be added to the request, and the value is the parameter's name in the log event properties.-->
    <IndexOperationParams>
      <Parameter>
        <Key>_routing</Key>
        <Value>%{RoutingSource}</Value>
      </Parameter>
      <Parameter>
        <Key>key</Key>
        <Value>value</Value>
      </Parameter>
    </IndexOperationParams>

    <!-- for more information read about log4net.Core.FixFlags -->
    <FixedFields>Partial</FixedFields>
    
    <Template>
      <Name>templateName</Name>
      <FileName>path2template.json</FileName>
    </Template>

    <!--Only one credential type can used at once-->
    <!--Here we list all possible types-->
    <AuthenticationMethod>
        <!--For basic authentication purposes-->
        <Basic>
            <Username>Username</Username>
            <Password>Password</Password>
        </Basic>
        <!--For AWS ElasticSearch service-->
        <Aws>
            <Aws4SignerSecretKey>Secret</Aws4SignerSecretKey>
            <Aws4SignerAccessKey>AccessKey</Aws4SignerAccessKey>
            <Aws4SignerRegion>Region</Aws4SignerRegion>
        </Aws>
    </AuthenticationMethod>
    
    <!-- all filters goes in ElasticFilters tag -->
    <ElasticFilters>
      <Add>
        <Key>@type</Key>
        <Value>Special</Value>
      </Add>

      <!-- using the @type value from the previous filter -->
      <Add>
        <Key>SmartValue</Key>
        <Value>the type is %{@type}</Value>
      </Add>

      <Remove>
        <Key>@type</Key>
      </Remove>

      <!-- you can load custom filters like I do here -->
      <Filter type="log4stash.Filters.RenameKeyFilter, log4stash">
        <Key>SmartValue</Key>
        <RenameTo>SmartValue2</RenameTo>
      </Filter>
    
      <!-- converts a json object to fields in the document -->
      <Json>
        <SourceKey>JsonRaw</SourceKey>
        <FlattenJson>false</FlattenJson>
    <!-- the separator property is only relevant when setting the FlattenJson property to 'true' -->
    <Separator>_</Separator> 
      </Json>

      <!-- converts an xml object to fields in the document -->
      <Xml>
        <SourceKey>XmlRaw</SourceKey>
        <FlattenXml>false</FlattenXml>
      </Xml>
      
      <!-- kv and grok filters similar to logstash's filters -->
      <Kv>
        <SourceKey>Message</SourceKey>
        <ValueSplit>:=</ValueSplit>
        <FieldSplit> ,</FieldSplit>
      </kv>

      <Grok>
        <SourceKey>Message</SourceKey>
        <Pattern>the message is %{WORD:Message} and guid %{UUID:the_guid}</Pattern>
        <Overwrite>true</Overwrite>
      </Grok>

      <!-- Convert string like: "1,2, 45 9" into array of numbers [1,2,45,9] -->
      <ConvertToArray>
        <SourceKey>someIds</SourceKey>
        <!-- The separators (space and comma) -->
        <Seperators>, </Seperators> 
      </ConvertToArray>

      <Convert>
        <!-- convert given key to string -->
        <ToString>shouldBeString</ToString>

        <!-- same as ConvertToArray. Just for convenience -->
        <ToArray>
           <SourceKey>anotherIds</SourceKey>
        </ToArray>
      </Convert>
    </ElasticFilters>
</appender>
```

Note that the filters got called by the order they appeared in the config (as shown in the example).

### Templates:
To get to know the [ElasticSearch templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html) follow the link.

Sample template could be found in: [log-index-spec.json](https://github.com/urielha/log4stash/blob/master/scripts/log-index-spec.json). And more complex template with dynamic mappings can be found in the tests template: [template.json](https://github.com/urielha/log4stash/blob/master/src/log4stash.Tests/template.json)

You can follow the link to read more about [dynamic mappings](https://www.elastic.co/guide/en/elasticsearch/reference/current/default-mapping.html).

### License:
[MIT License](https://github.com/urielha/log4stash/blob/master/LICENSE)

### Thanks:
Thanks to [@jptoto](https://github.com/jptoto) for the idea and the first working ElasticAppender.
Many thanks to [@mpdreamz](https://github.com/Mpdreamz) and the team for their great work on the NEST library!
The inspiration to the filters and style had taken from [elasticsearch/logstash](https://github.com/elasticsearch/logstash) project.

### Build status:

| Status | Provider |
| ------ | -------- |
| [![Build status][TravisImg]][TravisLink] | Mono CI provided by [travis-ci][] |
| [![Build Status][AppVeyorImg]][AppVeyorLink] | Windows CI provided by [AppVeyor][] (without tests for now) |

[TravisImg]:https://travis-ci.org/urielha/log4stash.svg?branch=master
[TravisLink]:https://travis-ci.org/urielha/log4stash
[AppVeyorImg]:https://ci.appveyor.com/api/projects/status/byp4s7vl8cuhyae0
[AppVeyorLink]:https://ci.appveyor.com/project/urielha/log4stash

[travis-ci]:https://travis-ci.org/
[AppVeyor]:http://www.appveyor.com/

[config-example]:https://github.com/urielha/log4stash#almost-full-configuration
[filters-section]:https://github.com/urielha/log4stash#filters

[ignasv]:https://github.com/ignasv
[erangil]:https://github.com/erangil2
[mfpalladino]:https://github.com/mfpalladino