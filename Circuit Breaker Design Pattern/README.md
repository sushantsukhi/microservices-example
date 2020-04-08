# Circuit Breaker (CB) Design Pattern - Netflix Hystrix :

This guide walks you through the process of applying circuit breakers to potentially failing method calls by using the Netflix Hystrix fault tolerance library

Why we need a circuit breaker
	In case we have serviceB down, serviceA should still try to recover from this and try to do one of the followings:
	Custom fallback: 
		Try to get the same data from some other source. If not possible, use its own cache value.
	Fail fast: 
		If serviceA knows that serviceB is down, there is no point waiting for the timeout and consuming its own resources. It should return ASAP “knowing” that serviceB is down.
	Don’t crash: 
		As we saw in this case, serviceA should not have crashed.
	Heal automatic: 
		Periodically check if serviceB is working again.
	Other APIs should work: 
		All other APIs should continue to work.
		
What is circuit breaker design?
	
	There are three status of Circuit Breaker - 
	CLOSED -> When service is UP, when service is failing then it will change to OPEN.
	OPEN -> When service is DOWN.
	HALF-OPEN -> After the end of the period, CB will change the state to HALF-OPEN to check if the system that is being requested is still failing. 
				 If the test still fails, the CB will return to state “OPEN” again, but if successful the CB will change the state to “CLOSED”

	Once serviceA “knows” that serviceB is down, there is no need to make request to serviceB. serviceA should return cached data or timeout error as soon as it can. This is the OPEN state of the circuit.
	Once serviceA “knows” that serviceB is up, we can CLOSED the circuit so that request can be made to serviceB again.
	Periodically make fresh calls to serviceB to see if it is successfully returning the result. This state is HALF-OPEN.

Code : 

BookReadingApplication ----> BookStoreApplication

BookReadingApplication :
	This web service will call BookStoreApplication to get the Book Name by using RestTemplate.
	server.port=8080
	
BookStoreApplication :
	This web service will produce Book Name which will consumed by other web service.
	server.port=8090 


@RestController
@SpringBootApplication
public class CircuitBreakerBookstoreApplication {

	@RequestMapping(value = "/recommended")
	public String readingList() {
		return "Spring in Action (Manning), Cloud Native Java (O'Reilly), Learning Spring Boot (Packt)";
	}
}

@EnableHystrixDashboard
@EnableCircuitBreaker
@RestController
@SpringBootApplication
public class CircuitBreakerReadingApplication {

	@RequestMapping("/to-read")
	public String toRead() {
		return bookService.readingList();
	}
}

@Service
public class BookService {

	@HystrixCommand(fallbackMethod = "reliable")
	public String readingList() {
		URI uri = URI.create("http://localhost:8090/recommended");

		return this.restTemplate.getForObject(uri, String.class);
	}

	public String reliable() {
		return "Cloud Native Java (O'Reilly)";
	}
}

@EnableCircuitBreaker - This enables Circuit Breaker
@EnableHystrixDashboard - This enables Hystrix Dashboard which can be accessed by following URL -
http://localhost:9098/hystrix
In this method we need to enter - http://localhost:9098/actuator/hystrix.stream
One more change need to be done in order to work it. Enter below property in application.properties
management.endpoints.web.exposure.include=hystrix.stream

Whenever we are calling t=other microservice there we have to annotate that method with @HystrixCommand and we have to pass the fallback Method name.
This fallback method will be called in case of failure of that calling microservice( i.e. if that microservice is not available for any reason that we are calling). In fallback method we might return cache data for some time till not microservice recovered itself.

@HystrixCommand(fallbackMethod = "callStudentServiceAndGetData_Fallback")
public String callStudentServiceAndGetData(String schoolname) {
	......
}

private String callStudentServiceAndGetData_Fallback(String schoolname) {
	......
}
