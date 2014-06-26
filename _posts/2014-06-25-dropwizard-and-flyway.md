---
layout: post
title: "Dropwizard and Flyway"
modified: 2014-06-25 23:50:08 -0500
category: software
tags: [dropwizard,flyway,rest,db,migrations,java,frameworks,sql]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

In my current project we are developing a bunch of microservices using [Dropwizard](dropwizard.github.io/dropwizard/), a light weight Java framework who provides you a quick way of creating RESTful services by combining some libraries like Jersey and Jackson.
<br/><br/>
One of the reason why we chose it is because it gives you a lot of stuff out of the box (like [Metrics](http://dropwizard.readthedocs.org/en/latest/manual/core.html#metrics) and an embedded Jetty) but all the libraries are easily changeable if you want to use something different. In our case, we needed to manage the DB migrations but didn't want to use Liquidbase, the library that came wih Dropwizard. So we looked around and picked [Flyway](http://flywaydb.org), a lighter alternative. Migration scripts in Flyway are written in pure SQL so they are easy to understand and maintain.
<br/><br/>
Flyway gives you three ways of configure your migrations: [Command Line Tool](http://flywaydb.org/documentation/commandline/), [API](http://flywaydb.org/documentation/api/) that can be called from Java and a plugin for a few build tools like [Maven](http://flywaydb.org/documentation/maven/) and [Gradle](http://flywaydb.org/documentation/gradle/). Initially for our project we opted to use the plugin for Gradle so we create our tasks like that:
<br/>

        flyway {
            def yamlConfig = new Yaml().load(new File('application-config.yml').newReader())
            def dbConfig = yamlConfig.database

            user = dbConfig.user
            password = dbConfig.password
            url = dbConfig.url
        }

        // Configure the run task to start the Dropwizard service
        run {
            dependsOn flywayMigrate
            args 'server', 'servicio-usuario.yml'
        }
        test {
            dependsOn flywayMigrate
            ....
        }

By making the task run and test dependant of flyway we make sure the DB is in the correct stage every time we execute the application or run the tests. And although this is a great use fo flyway we soon understood that our app needed something different: we obviously need our production environment to run the migrations but we won't have gradle in there so, what else can we do?
<br/><br/>
This is when we decided to use the flyway API and call it from the code when the application starts up. I have a [gist](https://gist.github.com/mariagomez/a0e1011bfb8b0cca0c9d) that show how you can do it following some of Dropwizard practices. Basically, Dropwizard has this notion of [bundles](http://dropwizard.readthedocs.org/en/latest/manual/core.html#bundles) that can be attached to the app on start up. That is how you wire up your ORM and your IoC container. So for Flyway we created a custom bundle that connects to the database using the configuration details and run the migrations.
<br/>

        public class DBMigrationsBundle implements ConfiguredBundle<HelloWorldConfiguration>{
            @Override
            public void run(HelloWorldConfiguration configuration, Environment environment) throws Exception {
                Flyway flyway = new Flyway();
                flyway.setDataSource(configuration.getDataSourceFactory().getUrl(),
                                    configuration.getDataSourceFactory().getUser(),
                                    configuration.getDataSourceFactory().getPassword());
                flyway.migrate();
            }

            @Override
            public void initialize(Bootstrap<?> bootstrap) {

            }
        }

We are now sure that whatever environment we deploy the app, it will run the migrations.
