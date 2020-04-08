# Bulkhead Design Pattern using resilience4j

// Create bulkhead config once
BulkheadConfig config = BulkheadConfig.custom()
									  .maxConcurrentCalls(150)
									  .maxWaitTIme(100)
									  .build();
									  
Bulhead bulkheadForPayment = Bulkhead.of("payments", config);

// docorate microservice call with bulkhead
Supplier<String> pay = Bulkhead.decorateSupplier(bulkheadForPayment, this::makePayment);

// Execute the call
Try<String> result = Try.ofSupplier(pay);
if (result.isSuccess()) {
	result.get();
} else {
	return "default-response";
}
