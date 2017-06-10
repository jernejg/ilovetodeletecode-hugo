+++
Tags = ["Development","golang"]
Description = ""
date = "2017-06-01T22:35:15+01:00"
title = "Running Apache Solr 6.5 on an Azure App Service instance"
draft = false
+++

This post describes a daring way to attempt to deploy a single Solr node to an Azure App Service. I say daring because I could not find a copy paste solution on Stack Overflow. I saw some light at the end of the tunnel after reading the [Upload a custom Java web app to Azure](https://docs.microsoft.com/en-us/azure/app-service-web/web-sites-java-custom-upload) article on Azure documentation which mentions Jetty and since Solr is running in a Jetty Servlet container by default I guessed I should at least give it a try. 

## Why Azure App Services?

I have a small amount of data to index and I will never need more than one Solr node running, so why not. A pre-configured Linux VM would be an overkill. I haven't tried Azure Search yet, but with Solr I can retrieve all the stats data for my filters and search results in a single request, not to mention its powerful natural language processing capabilities.

Let's get to work...

## Azure Portal's configuration UI
You only need to enable Java (at least version 1.8 for Solr > 6.x) and Jetty as a Web Container.

{{< figure link="/img/Post5_azure_portal_configuration.png" src="/img/Post5_azure_portal_configuration.png">}}

## Web.config httpPlatform configuration
Extract contents of [solr-6.5.1.zip](http://www-eu.apache.org/dist/lucene/solr/6.5.1/) to <b>%HOME%\site\wwwroot</b> folder of your App Service and add a Web.config with the following settings:

{{< highlight xml "style=native" >}}
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

<code>HTTP_PLATFORM_PORT</code> is the environment variable that holds the port number Azure has dynamically assigned to the Java process. We use its value as a parameter to the <b>-p</b> switch so that Solr does not start on the default port which is <b>8983</b>.

Having <code>stdoutLogEnabled</code> enabled causes the <b>stdout</b> and <b>stderr</b> for the process specified in the <b>processPath</b> setting to be redirected to location defined in <b>stdoutLogFile</b>.

The default value of <b>stdoutLogFile</b> should be <b> %HOME%\LogFiles\ </b> but in my case the logs were actually written to <b>%HOME%\site\wwwroot\ </b>

Although there seem to be some problems during the startup, Solr starts up successfully and you can access the Admin panel at &lt;yoursite&gt;.azurewebsites.net

{{< highlight sv "style=native" >}}
Access is denied.
Waiting up to 30 to see Solr running on port 16055
INFO  - 2017-06-09 12:44:49.394; org.apache.http.impl.client.DefaultRequestDirector; I/O exception (java.net.SocketException) caught when connecting to {}->http://localhost:16055: Permission denied: connect
INFO  - 2017-06-09 12:44:49.521; org.apache.http.impl.client.DefaultRequestDirector; Retrying connect to {}->http://localhost:16055
INFO  - 2017-06-09 12:44:49.659; org.apache.http.impl.client.DefaultRequestDirector; I/O exception (java.net.SocketException) caught when connecting to {}->http://localhost:16055: Permission denied: connect
INFO  - 2017-06-09 12:44:49.659; org.apache.http.impl.client.DefaultRequestDirector; Retrying connect to {}->http://localhost:16055
Started Solr server on port 16055. Happy searching!
{{< /highlight >}}

That's it! And oh yeah, don't forget to enable Authentication (Azure App Service / Solr Authentication Plugins) or else everyone will have access to your Admin panel and Solr API.
