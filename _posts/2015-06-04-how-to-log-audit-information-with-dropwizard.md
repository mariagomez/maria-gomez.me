---
layout: post
title: "How to log audit information with Dropwizard"
modified: 2015-06-04 07:30:37 -0400
category: [software]
tags: [dropwizard,audit,java,microservices]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

For those who don’t know, [Dropwizard](dropwizard.github.io/dropwizard/) is Java framework that allows you to build light-weight RESTful webservices. It comes with a few functionalities already configured and ready to use like an embedded Jetty container and the [Jersey](https://jersey.java.net/) framework for building webservices. It also allows you to add more functionality as you need by importing or creating <i>[bundles](https://dropwizard.github.io/dropwizard/manual/core.html#bundles)</i>. This framework is currently very popular among the Java community and it’s being used as an alternative of the much heavier enterprise options. The [Thoughtworks Tech Radar](http://www.thoughtworks.com/radar/languages-and-frameworks/dropwizard) shows it on the Adopt ring, meaning we promote and recommend its use. In our project we are using Dropwizard to build a set of microservices that are consumed by a single page app.
<br/><br/>
Although it has a really active community and it’s improving pretty fast, the framework is not mature yet and therefore there are some functionalities or requirements that are not available out of the box. One of them is the [audit trail](http://en.wikipedia.org/wiki/Audit_trail), that is, keeping a record of the transactions done by the users in the system that is not intrusive for them and respect their sensitive information. When we talk about webservices, we can try a few approaches:

  * Log at the persistence layer. When there is a change in any table or entity you can trigger a procedure that will record the change. Hibernate has something called [Envers](http://envers.jboss.org/) that at first seems like an easy way to do this but we could not make it work as we were not using J2EE and the library is only available as part of the enterprise package . Another reason not to go down this path is that there doesn’t seem to be a way of separating the audit tables from the normal tables so the database can grow really quickly and there may be some performance issues related to that.
  <br/><br/>
  * Log using DB hooks or interceptors that get called when the DB is about to get modified.
  <br/><br/>
  * Log at the resource level. When a request came to the server, log the content of it as well as the additional information you need. Jersey allows you to easily register MethodDispatcher that will decorate your resources with new functionality. I did some research in the DW mailing list and found this [gist](https://gist.github.com/ryankennedy/6688601), which gave the confirmation and inspiration we needed to follow this path.

### Creating the audit bundle in DW

Creating a dispatcher is easy enough, our looks like this:

{% highlight java linenos %}
package me.mariagomez.dropwizard.audit;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sun.jersey.api.core.HttpContext;
import com.sun.jersey.spi.dispatch.RequestDispatcher;
import io.dropwizard.jackson.Jackson;
import me.mariagomez.dropwizard.audit.providers.PrincipalProvider;
import org.joda.time.DateTime;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import static me.mariagomez.dropwizard.audit.Constants.X_REMOTE_ADDR;

public class AuditRequestDispatcher implements RequestDispatcher {

  private static final ObjectMapper MAPPER = Jackson.newObjectMapper();
  private static final Logger LOGGER = LoggerFactory.getLogger(AuditRequestDispatcher.class);

  private RequestDispatcher dispatcher;
  private AuditWriter auditWriter;
  private PrincipalProvider principalProvider;

  public AuditRequestDispatcher(RequestDispatcher dispatcher, AuditWriter auditWriter,
                            PrincipalProvider principalProvider) {
    this.dispatcher = dispatcher;
    this.auditWriter = auditWriter;
    this.principalProvider = principalProvider;
  }

  @Override
  public void dispatch(Object resource, HttpContext context) {
    dispatcher.dispatch(resource, context);

    int responseCode = context.getResponse().getStatus();
    if (responseCode < 200 || responseCode > 299) {
        return;
    }
    String method = context.getRequest().getMethod();
    String path = context.getRequest().getPath();
    String remoteAddress = context.getRequest().getRequestHeaders().getFirst(X_REMOTE_ADDR);
    String username = principalProvider.getUsername();
    DateTime date = DateTime.now();

    String entity = null;
    try {
        entity = MAPPER.writeValueAsString(context.getResponse().getEntity());
    } catch (JsonProcessingException e) {
        LOGGER.error("Error while parsing the entity. \n Message: " + e.getMessage());
    }

    AuditInfo auditInfo = new AuditInfo(method, responseCode, date, entity, path, remoteAddress, username);
    auditWriter.write(auditInfo);
  }
}
{% endhighlight %}


As you can see, we let the dispatcher’s flow continue and we insert our logic after the request has been processed. This is because we only want to audit the successful requests. We also have a PrincipalProvider injected as a dependency, this gives us the info about the user making the request.
<br/><br/>
Next we attach this dispatcher to the resource by using a Provider:

{% highlight java linenos %}
package me.mariagomez.dropwizard.audit;

import com.sun.jersey.api.model.AbstractResourceMethod;
import com.sun.jersey.spi.container.ResourceMethodDispatchProvider;
import com.sun.jersey.spi.dispatch.RequestDispatcher;
import me.mariagomez.dropwizard.audit.providers.PrincipalProvider;

import java.util.Arrays;
import java.util.List;

public class AuditMethodDispatchProvider implements ResourceMethodDispatchProvider {

    private static final List<String> AUDITABLE_METHODS = Arrays.asList("POST", "PUT", "DELETE");
    private ResourceMethodDispatchProvider provider;
    private AuditWriter auditWriter;
    private PrincipalProvider principalProvider;

    public AuditMethodDispatchProvider(ResourceMethodDispatchProvider provider, AuditWriter auditWriter,
                                       PrincipalProvider principalProvider) {
        this.provider = provider;
        this.auditWriter = auditWriter;
        this.principalProvider = principalProvider;
    }

    @Override
    public RequestDispatcher create(AbstractResourceMethod abstractResourceMethod) {
        RequestDispatcher requestDispatcher = provider.create(abstractResourceMethod);
        if (isMethodAuditable(abstractResourceMethod.getHttpMethod())){
            return new AuditRequestDispatcher(requestDispatcher, auditWriter, principalProvider);
        }
        return requestDispatcher;
    }

    private boolean isMethodAuditable(String method) {
        return AUDITABLE_METHODS.contains(method);
    }

}

{% endhighlight %}

In our case, we wanted the dispatcher to be attached to all resources with POST, PUT and DELETE verbs, but the [gist](https://gist.github.com/ryankennedy/6688601) I mentioned before shows how to do this by using an annotation instead.
<br/><br/>
Once we have all this in place, we just need to create the bundle so we can use this functionality within our application. Bundles in Dropwizard can have access to the configuration files so we took advantage of that to allow different writers (initially we only used the normal logs but you can link to a writer that send the data somewhere else).

{% highlight java linenos %}
package me.mariagomez.dropwizard.audit;

import io.dropwizard.Configuration;
import io.dropwizard.ConfiguredBundle;
import io.dropwizard.jersey.setup.JerseyEnvironment;
import io.dropwizard.setup.Bootstrap;
import io.dropwizard.setup.Environment;
import me.mariagomez.dropwizard.audit.filters.RemoteAddressFilter;
import me.mariagomez.dropwizard.audit.providers.PrincipalProvider;

public class AuditBundle<T extends Configuration> implements ConfiguredBundle<T> {

    private AuditWriter auditWriter;
    private PrincipalProvider principalProvider;

    public AuditBundle(AuditWriter auditWriter, PrincipalProvider principalProvider) {
        this.auditWriter = auditWriter;
        this.principalProvider = principalProvider;
    }

    @Override
    public void run(T configuration, Environment environment) {
        JerseyEnvironment jersey = environment.jersey();
        jersey.register(new AuditMethodDispatchAdapter(auditWriter, principalProvider));
        jersey.getResourceConfig().getContainerRequestFilters().add(new RemoteAddressFilter());
    }

    @Override
    public void initialize(Bootstrap<?> bootstrap) {
        // Do nothing
    }
}
{% endhighlight %}

Finally, you need to register the bundle in your application:

{% highlight java linenos %}
package me.mariagomez.dropwizard.audit.example;

import io.dropwizard.Application;
import io.dropwizard.setup.Bootstrap;
import io.dropwizard.setup.Environment;
import me.mariagomez.dropwizard.audit.AuditBundle;

public class ExampleApplication extends Application<ExampleConfiguration> {

    @Override
    public void initialize(Bootstrap<ExampleConfiguration> bootstrap) {
        AuditBundle<ExampleConfiguration> bundle = new AuditBundle<>(new PrincipalProviderImpl());
        bootstrap.addBundle(bundle);
    }

    @Override
    public void run(ExampleConfiguration configuration, Environment environment) throws Exception {
        environment.jersey().register(ExampleResource.class);
    }
}
{% endhighlight %}


And you should be good to go!
<br/><br/>

PS: All the code plus an example is in [this project on github](https://github.com/mariagomez/dropwizard-audit). Contribution are more than welcome.
