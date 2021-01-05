## Financial Advisor

A simple financial advisor app built using microservice architecture, Spring Boot, Spring Cloud and Docker. 

<br>

#### Functional services

Decomposed into three core microservices. All of them are independently deployable applications organized around certain business domains.

<img width="880" alt="Functional services" src="https://cloud.githubusercontent.com/assets/6069066/13900465/730f2922-ee20-11e5-8df0-e7b51c668847.png">

#### Account service

Contains general input logic and validation: incomes/expenses items, savings and account settings.


| Method | Path                | Description                                                    | User authenticated | Available from UI |
| ------ | ------------------- | -------------------------------------------------------------- | :----------------: | :---------------: |
| GET    | /accounts/{account} | Get specified account data                                     |                    |                  |
| GET    | /accounts/current   | Get current account data                                       |         ×         |        ×        |
| GET    | /accounts/demo      | Get demo account data (pre-filled incomes/expenses items, etc) |                    |        ×        |
| PUT    | /accounts/current   | Save current account data                                      |         ×         |        ×        |
| POST   | /accounts/          | Register new account                                           |                    |        ×        |

#### Statistics service

Performs calculations on major statistics parameters and captures time series for each account. Datapoint contains values normalized to base currency and time period. This data is used to track cash flow dynamics during the account lifetime.


| Method | Path                  | Description                                                  | User authenticated | Available from UI |
| ------ | --------------------- | ------------------------------------------------------------ | :----------------: | :---------------: |
| GET    | /statistics/{account} | Get specified account statistics                             |                    |                  |
| GET    | /statistics/current   | Get current account statistics                               |         ×         |        ×        |
| GET    | /statistics/demo      | Get demo account statistics                                  |                    |        ×        |
| PUT    | /statistics/{account} | Create or update time series datapoint for specified account |                    |                  |

#### Notification service

Stores user contact information and notification settings (reminders, backup frequency etc). Scheduled worker collects required information from other services and sends e-mail messages to subscribed customers.


| Method | Path                            | Description                                | User authenticated | Available from UI |
| ------ | ------------------------------- | ------------------------------------------ | :----------------: | :---------------: |
| GET    | /notifications/settings/current | Get current account notification settings  |         ×         |        ×        |
| PUT    | /notifications/settings/current | Save current account notification settings |         ×         |        ×        |

#### Notes

- Each microservice has its own database, so there is no way to bypass API and access persistence data directly.
- MongoDB is used as a primary database for each of the services.
- All services are talking to each other via the Rest API

## Infrastructure

[Spring cloud](https://spring.io/projects/spring-cloud) provides powerful tools for developers to quickly implement common distributed systems patterns -
<img width="880" alt="Infrastructure services" src="https://cloud.githubusercontent.com/assets/6069066/13906840/365c0d94-eefa-11e5-90ad-9d74804ca412.png">

### Config service

[Spring Cloud Config](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html) is horizontally scalable centralized configuration service for the distributed systems. It uses a pluggable repository layer that currently supports local storage, Git, and Subversion.

In this project, we are going to use `native profile`, which simply loads config files from the local classpath. You can see `shared` directory in [Config service resources](https://github.com/sqshq/PiggyMetrics/tree/master/config/src/main/resources). Now, when Notification-service requests its configuration, Config service responses with `shared/notification-service.yml` and `shared/application.yml` (which is shared between all client applications).

##### Client side usage

Just build Spring Boot application with `spring-cloud-starter-config` dependency, autoconfiguration will do the rest.

Now you don't need any embedded properties in your application. Just provide `bootstrap.yml` with application name and Config service url:

```yml
spring:
  application:
    name: notification-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
```

##### With Spring Cloud Config, you can change application config dynamically.

For example, [EmailService bean](https://github.com/sqshq/PiggyMetrics/blob/master/notification-service/src/main/java/com/piggymetrics/notification/service/EmailServiceImpl.java) is annotated with `@RefreshScope`. That means you can change e-mail text and subject without rebuild and restart the Notification service.

First, change required properties in Config server. Then make a refresh call to the Notification service:
`curl -H "Authorization: Bearer #token#" -XPOST http://127.0.0.1:8000/notifications/refresh`

You could also use Repository [webhooks to automate this process](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html#_push_notifications_and_spring_cloud_bus)

##### Notes

- `@RefreshScope` doesn't work with `@Configuration` classes and doesn't ignores `@Scheduled` methods
- `fail-fast` property means that Spring Boot application will fail startup immediately, if it cannot connect to the Config Service.

### Auth service

Authorization responsibilities are extracted to a separate server, which grants [OAuth2 tokens](https://tools.ietf.org/html/rfc6749) for the backend resource services. Auth Server is used for user authorization as well as for secure machine-to-machine communication inside the perimeter.

In this project, I use [`Password credentials`](https://tools.ietf.org/html/rfc6749#section-4.3) grant type for users authorization (since it's used only by the UI) and [`Client Credentials`](https://tools.ietf.org/html/rfc6749#section-4.4) grant for service-to-service communciation.

Spring Cloud Security provides convenient annotations and autoconfiguration to make this really easy to implement on both server and client side. You can learn more about that in [documentation](http://cloud.spring.io/spring-cloud-security/spring-cloud-security.html).

On the client side, everything works exactly the same as with traditional session-based authorization. You can retrieve `Principal` object from the request, check user roles using the expression-based access control and `@PreAuthorize` annotation.

Each PiggyMetrics client has a scope: `server` for backend services and `ui` - for the browser. We can use `@PreAuthorize` annotation to protect controllers from  an external access:

```java
@PreAuthorize("#oauth2.hasScope('server')")
@RequestMapping(value = "accounts/{name}", method = RequestMethod.GET)
public List<DataPoint> getStatisticsByAccountName(@PathVariable String name) {
	return statisticsService.findByAccountName(name);
}
```
