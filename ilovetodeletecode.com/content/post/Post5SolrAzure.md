+++
Tags = ["Development","golang"]
Description = ""
date = "2017-06-01T22:35:15+01:00"
title = "Running Apache Solr 6.5 on a Azure App Service instance"
+++

This post describes a daring way to attempt to deploy a single Solr node to an Azure App Service. I say daring because I could not find a copy paste solution on Stack Overflow. I saw some light at the end of the tunnel after reading the [Upload a custom Java web app to Azure](https://docs.microsoft.com/en-us/azure/app-service-web/web-sites-java-custom-upload) article on Azure documentation which mentions Jetty and since Solr is running in a Jetty Servlet container by default I guessed I should at least give it a try.

## Why Azure App Services?

I have a small ammount of data to index and I will never need more than one Solr node running, so why not. A pre configured Linux VM would be a overkill. I haven't tried Azure Search yet, but with Solr I can retrieve all the data for my filters and search results in a single http request, not to mention its powerful natural language processing capabilities. And offcourse I wanted to put those MSDN subscription Azure credits to good use.

Lets get to work...

## Azure Portal's configuration UI
You only need to enable Java (at least version 1.8 for Solr > 6.x) and Jetty as a Web Container.

{{< figure link="/img/Post5_azure_portal_configuration.png" src="/img/Post5_azure_portal_configuration.png">}}

## Web.config httpPlatform configuration
Extract contents of [solr-6.5.1.zip](http://www-eu.apache.org/dist/lucene/solr/6.5.1/) and deploy via FTP to your wwwroot folder and add a Web.config with the following settings.

{{< highlight xml "style=paraiso-dark" >}}
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>    
    <handlers>
      <add  name="httppPlatformHandler" 
            path="*" 
            verb="*" 
            modules="httpPlatformHandler" 
            resourceType="Unspecified" />
    </handlers>
    <httpPlatform processPath="%HOME%\site\wwwroot\bin\solr.cmd" 
        arguments="start -p %HTTP_PLATFORM_PORT%"
        startupTimeLimit="20"
        startupRetryCount="10"
        stdoutLogEnabled="true">
    </httpPlatform>
  </system.webServer>
</configuration>
{{< /highlight >}}

As soon as you provide the Web.config file, the stdout log files should start appearing.
