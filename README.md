# Microservices with Spring Boot and Java

Reference : [Microservices with Spring Boot and Java](http://www.springboottutorial.com/creating-microservices-with-spring-boot-part-1-getting-started)


In this example, we would create two microservices:

Forex Service - Abbreviated as FS
Currency Conversion Service - Abbreviated as CCS
The diagram below shows the communication between CCS and FS. 

Currency Conversion Service -> Forex Service
	  
## 1. Forex Service - 
	Test Forex Microservice
	GET to http://localhost:8000/currency-exchange/from/EUR/to/INR

	Dependencies :
		Web
		DevTools
		Starter JPA
		H2
		
	POM.xml 
		<properties>
			<spring-cloud.version>Hoxton.SR3</spring-cloud.version>
		</properties>
		<groupId>org.springframework.cloud</groupId>
		<dependency>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		
	Classes :
		ExchangeValue - 
			This will be entity class.
			
		ExchangeValueRepository - 
			This will be interface which will extend JpaRepository<ExchangeValue, Long>
				Method - ExchangeValue findByFromAndTo(String from, String to);
				
		ForexController -
			This will be Rest Controller
			
		SpringBootMicroserviceForexServiceApplication - 
			Enable this class for auto registration as Client with eureka server with below annotation
			@EnableDiscoveryClient
		
		application.properties
			spring.application.name=forex-service
			server.port=8000
			spring.jpa.show-sql=true
			spring.h2.console.enabled=true
			eureka.client.service-url.default-zone=http://localhost:8761/eureka
			
		data.sql
			insert into exchange_value(id,currency_from,currency_to,conversion_multiple,port)
			values(10001,'USD','INR',65,0);
			insert into exchange_value(id,currency_from,currency_to,conversion_multiple,port)
			values(10002,'EUR','INR',75,0);
			insert into exchange_value(id,currency_from,currency_to,conversion_multiple,port)
			values(10003,'AUD','INR',25,0);
			
***Please note that you can run many instance of this server by mentioning '-Dserver.port' in VM argument.
For e.g. in above example we are already running one instance on 8000 port so if we want to run another instance on 8001 then
we can create another runtime instance and mention -Dserver.port=8001 it will run on 8001 port and as we have mentioned eureka
server url in application.properties it will automaticaly register itself to eureka server and eureka server will use this instance 
in load balancing. i.e. now there are two forex service instances running on 8000 and 8001 so when currency conversion service we hit first request on 8000 and next request on 8001 simultaneously. This will ensure that we are equally distributing load on forex service.
We can verify this my checking port in the response.***

***If we run another instance on 8002 then it will work in above mentioned manner.***

## 2. Currency Conversion Service-
	Test Currency Converion Microservice
	GET to http://localhost:8100/currency-converter/from/EUR/to/INR/quantity/10000

	Dependencies :
		Web
		DevTools
		Feign
		
	POM.xml 
		<properties>
			<spring-cloud.version>Hoxton.SR3</spring-cloud.version>
		</properties>
		<groupId>org.springframework.cloud</groupId>
		<dependency>
			<artifactId>spring-cloud-starter-openfeign</artifactId>
			<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		
	Classes :
		CurrencyConversionBean - 
			This will be entity class.
		
		CurrencyExchangeServiceProxy - 
			Creating a Feign Proxy. Feign provide a better alternative to RestTemplate to call REST API. 
			We are using RibbonClient to remove hard coding URL from FeignClient as well as distributing the data equally to multiple forex services.
			
			@FeignClient(name="forex-service")
			@RibbonClient(name="forex-service")
			public interface CurrencyExchangeServiceProxy {
			  @GetMapping("/currency-exchange/from/{from}/to/{to}")
			  public CurrencyConversionBean retrieveExchangeValue
				(@PathVariable("from") String from, @PathVariable("to") String to);
			}

		CurrencyConversionController - 
			This class is Rest Controller which is using two ways to get data from forex service.
			1. It will use RestTemplate to call forex service.
		
			@GetMapping("/currency-converter/from/{from}/to/{to}/quantity/{quantity}")
			  public CurrencyConversionBean convertCurrency(@PathVariable String from, @PathVariable String to,
				  @PathVariable BigDecimal quantity) {

				Map<String, String> uriVariables = new HashMap<>();
				uriVariables.put("from", from);
				uriVariables.put("to", to);

				ResponseEntity<CurrencyConversionBean> responseEntity = new RestTemplate().getForEntity(
					"http://localhost:8000/currency-exchange/from/{from}/to/{to}", CurrencyConversionBean.class,
					uriVariables);

				CurrencyConversionBean response = responseEntity.getBody();

				return new CurrencyConversionBean(response.getId(), from, to, response.getConversionMultiple(), quantity,
					quantity.multiply(response.getConversionMultiple()), response.getPort());
			}
			
			2. Using Feign Proxy from the Microservice Controller
			@Autowired
			private CurrencyExchangeServiceProxy proxy;

			@GetMapping("/currency-converter-feign/from/{from}/to/{to}/quantity/{quantity}")
			public CurrencyConversionBean convertCurrencyFeign(@PathVariable String from, @PathVariable String to, @PathVariable BigDecimal quantity) {

				CurrencyConversionBean response = proxy.retrieveExchangeValue(from, to);
				logger.info("{}", response);

				return new CurrencyConversionBean(response.getId(), from, to, response.getConversionMultiple(), 
				quantity, quantity.multiply(response.getConversionMultiple()), response.getPort());
		    }

		SpringBootMicroserviceCurrencyConversionApplication - 
			Enable Feign Clients in this class by annotating this class with below annotation
			@EnableFeignClients("com.in28minutes.springboot.microservice.example.currencyconversion")
			@EnableDiscoveryClient
		
		application.properties
			spring.application.name=currency-conversion-service
			server.port=8100
			eureka.client.service-url.default-zone=http://localhost:8761/eureka

## 3. Eureka Server -
	Test Eureka Server
	GET to http://localhost:8761

	Dependencies :
		Eureka
		DevTools
		
	POM.xml 
		<properties>
			<spring-cloud.version>Hoxton.SR3</spring-cloud.version>
		</properties>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>
	Classes :
			
		SpringBootMicroserviceEurekaNamingServerApplication - 
			Enable this class As Eureka server so that other client can register to this server.
			@EnableEurekaServer
		
		application.properties
			spring.application.name=netflix-eureka-naming-server
			server.port=8761
			eureka.client.register-with-eureka=false -- We are telling this class to not to register self as client as this is server.
			eureka.client.fetch-registry=false
			



#### Important Spring Cloud Modules 

```
Dynamic Scale Up and Down. Using a combination of
 - Naming Server (Eureka)
 - Ribbon (Client Side Load Balancing)
 - Feign (Easier REST Clients)

Visibility and Monitoring with
 - Zipkin Distributed Tracing
 - Netflix API Gateway

Configuration Management with
 - Spring Cloud Config Server

Fault Tolerance with
 - Hystrix
```
