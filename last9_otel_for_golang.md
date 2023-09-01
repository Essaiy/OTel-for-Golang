1. Open your terminal and create a new directory for your project.

   ```
   mkdir otel-golang
   cd otel-golang
   ```

Create a `docker-compose.yaml` file with the following content.

   ```yaml
   version: '3'
   services:
     otel-collector:
       image: otel/opentelemetry-collector-dev:latest
       ports:
         - 4317:4317
         - 55680:55680
     app:
       build: .
       ports:
         - 8080:8080
   ```

Create a `Dockerfile` with the following content.

   ```Dockerfile
   FROM golang:latest
   WORKDIR /app
   COPY . .
   RUN go build -o main .
   CMD ["./main"]
   ```

Run the following command to start the collector.

   ```
   docker-compose up -d
   ```

Stage 2: Install OpenTelemetry
Install the necessary OpenTelemetry libraries, exporters and trace packages in your Go environment by running the following commands.

```
go get -u go.opentelemetry.io/otel
go get -u go.opentelemetry.io/otel/exporters/otlp
go get -u go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp
go get -u go.opentelemetry.io/otel/trace
```

Stage 3: Instrument the Application
1. In the root directory, create a new file called `main.go` and add the following code.

   ```go
   package main

   import (
       "context"
       "fmt"
       "log"
  
       "go.opentelemetry.io/otel"
       "go.opentelemetry.io/otel/exporters/otlp"
       "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
       "go.opentelemetry.io/otel/trace"
   )
  
   func main() {
       exporter, err := otlp.NewExporter(context.TODO(),
           otlp.WithInsecure(),
           otlp.WithEndpoint("http://otel-collector:4317"),
           otlp.WithHTTPClient(otlptracehttp.NewClient()))
       if err != nil {
           log.Fatalf("Failed to create exporter: %v", err)
       }
       defer exporter.Shutdown(context.Background())
  
       provider := otel.GetTracerProvider()
       tracer := provider.Tracer("example")
  
       trace.RegisterExporter(exporter)
       trace.ApplyConfig(trace.Config{DefaultSampler: trace.AlwaysSample()})
  
       ctx, span := tracer.Start(context.Background(), "sayHello")
       defer span.End()
  
       fmt.Println("Hello, OpenTelemetry!")
   }
   ```

2. Build the application by running the following command in the terminal.

   ```
   docker-compose build
   ```

3. Start application containers using the following command.

   ```
   docker-compose up
   ```

4. Access the application by opening a browser and navigating to http://localhost:8080. You should see the message; "Hello, OpenTelemetry!" on the console.

5. Navigate to http://localhost:55680 to open OpenTelemetry Collector's web UI. From the menu, click on "Services" and then click on the "app" service to see the telemetry data collected from your application.

How to Instrument a Go Application with OpenTelemetry 
You can instrument Go applications with OpenTelemetry via the library-based instrumentation. Let's explore how. 

1. Import the required packages.

      ```go
      import (
          "context"
          "fmt"
          "go.opentelemetry.io/otel"
          "go.opentelemetry.io/otel/exporters/otlp"
          "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
          "go.opentelemetry.io/otel/sdk/resource"
          "go.opentelemetry.io/otel/trace"
      )
      ```

2. Create and configure the exporter.

      ```go
      exporter, err := otlp.NewExporter(context.TODO(),
          otlp.WithInsecure(),
          otlp.WithEndpoint("http://otel-collector:4317"),
          otlp.WithHTTPClient(otlptracehttp.NewClient()),
      )
      if err != nil {
          log.Fatalf("Failed to create exporter: %v", err)
      }
      defer exporter.Shutdown(context.Background())
      ```

3. Initialize the exporter and tracer.

   ```go
   exporter, err := otlp.NewExporter(context.TODO(),
       otlp.WithInsecure(),
       otlp.WithEndpoint("http://otel-collector:4317"),
       otlp.WithHTTPClient(otlptracehttp.NewClient()))
   if err != nil {
       log.Fatalf("Failed to create exporter: %v", err)
   }
   defer exporter.Shutdown(context.Background())
   
   provider := otel.GetTracerProvider()
   tracer := provider.Tracer("example")
   ```

4. Set the global trace provider and register the exporter.

      ```go
      otel.SetTracerProvider(otel.NewTracerProvider(
          otel.TracerProviderOptions{
              Resource: resource.NewWithAttributes(
                  resource.Attributes{
                      "service.name": "my-service",
                  },
              ),
              BatchExporter: exporter,
          },
      ))
      ```

5. Start a span and perform instrumented operations.

      ```go
      ctx, span := otel.Tracer("my-component").Start(context.Background(), "my-operation")
      defer span.End()
  
      // Perform the instrumented operations
      // ...
      ```

With this method, you can collect various telemetry data and metadata types, including metrics (counters, gauges and histograms), logs, attributes, spans and traces, for export as YAML, JSON, CSV, or other types of files. 
Distributed Tracing in Golang 
Distributed tracing enables request tracking across multiple services. Follow these steps to use OpenTelemetry’s context propagation for distributed tracing and span correlation in a Go application.

1. Import the required packages.

   ```go
   import (
       "context"
       "net/http"
       "go.opentelemetry.io/otel"
       "go.opentelemetry.io/otel/exporters/otlp"
       "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
       "go.opentelemetry.io/otel/label"
       "go.opentelemetry.io/otel/propagation"
       "go.opentelemetry.io/otel/sdk/resource"
       "go.opentelemetry.io/otel/trace"
       "go.opentelemetry.io/otel/trace/tracerprovider"
       "go.opentelemetry.io/otel/otelhttp"
   )
   ```

2. Initialize the trace provider and exporter.

   ```go
   func initTracer() error {
       exporter, err := otlp.NewExporter(context.TODO(),
           otlp.WithInsecure(),
           otlp.WithEndpoint("http://otel-collector:4317"),
           otlp.WithHTTPClient(otlptracehttp.NewClient()),
       )
       if err != nil {
           return err
       }
      
       tp := tracerprovider.NewProvider(
           tracerprovider.WithBatcher(exporter),
           tracerprovider.WithResource(resource.NewWithAttributes(label.String("service.name", "my-service"))),
       )
      
       otel.SetTracerProvider(tp)
       otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))
      
       return nil
   }
   ```

3. Instrument the application by creating spans for each operation.

   ```go
   func myHandler(w http.ResponseWriter, r *http.Request) {
       ctx := r.Context()
       tracer := otel.Tracer("my-component")
       ctx, span := tracer.Start(ctx, "my-operation")
       defer span.End()

       // Perform instrumented operations
       // ...

       span.AddEvent("my-event", trace.WithAttributes(label.String("key", "value")))
   }
   ```

4. Wrap your HTTP handler to enable automatic instrumentation with OpenTelemetry.

   ```go
   http.HandleFunc("/path", otelhttp.NewHandler(http.HandlerFunc(myHandler), "handler-name"))
   ```

5. Set up the OpenTelemetry middleware to capture HTTP traces and propagate context.

   ```go
   func main() {
       // Initialize the tracer provider and exporter
       err := initTracer()
       if err != nil {
           log.Fatalf("Failed to initialize tracer: %v", err)
       }

       // Set the OpenTelemetry middleware to capture traces
       srv := &http.Server{
           Addr:    ":8080",
           Handler: otelhttp.NewHandler(http.DefaultServeMux, "http-server"),
       }

       // Start the server
       if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
           log.Fatalf("Failed to listen and serve: %v", err)
       }
   }
   ```

By following these steps, you can trace request flow through different application components to know which services are working and if they are functioning optimally. If issues are identified, spans can provide adequate context for their remediation.

Visualizing your Telemetry Data
After collecting your Go telemetry, you must visualize them in an observability platform, in this case, Prometheus. Follow these steps to export data from OTel to Prometheus.

1. Modify the `docker-compose.yml` file to add the Prometheus service.

      ```yaml
      version: '3'
      
      services:
        myapp:
          build:
            context: .
            dockerfile: Dockerfile
          ports:
            - '8080:8080'
      
        jaeger:
          image: jaegertracing/all-in-one:latest
          ports:
            - '16686:16686'
      
        prometheus:
          image: prom/prometheus
          ports:
            - '9090:9090'
          volumes:
            - ./prometheus.yml:/etc/prometheus/prometheus.yml
      ```

2. Create a `prometheus.yml` file in your project directory and define the configuration.

      ```
      touch prometheus.yml
      code prometheus.yml
      ```

3. Add the following code to the `prometheus.yml` file to configure Prometheus to scrape metrics from your Go application.

      ```yaml
      global:
        scrape_interval: 10s
      
      scrape_configs:
        - job_name: 'myapp'
          static_configs:
            - targets: ['myapp:8080']
      ```

4. Configure your Go application to expose metrics using the OpenTelemetry Prometheus exporter and import the necessary packages.

      ```go
      import (
          "go.opentelemetry.io/otel/exporters/metric/prometheus"
          "go.opentelemetry.io/otel/metric"
          "go.opentelemetry.io/otel"
          "go.opentelemetry.io/otel/label"
          "go.opentelemetry.io/otel/sdk/metric/controller/push"
      )
      ```

5. Create a new Prometheus exporter and set it as the OpenTelemetry metric exporter

      ```go
      // Create a new Prometheus exporter
      promExporter, err := prometheus.NewExporter(prometheus.Options{})

      // Set the Prometheus exporter as the metric exporter
      if err == nil {
          pusher := push.New(
              promExporter,
              push.WithPeriod(1*time.Second),
          )
          controller := pusher.Controller()
          controller.Start()
          defer controller.Stop()

          metric.SetMeterProvider(pusher.Provider())
      }
      ```

6. Instrument your code with metrics using the OpenTelemetry API.

      ```go
      meter := otel.GetMeterProvider().Meter("myapp")

      counter := metric.Must(meter).NewInt64Counter("requests_total")
      counter.Add(ctx, 1, label.String("path", "/api/foo"))
      ```

7. Access the Prometheus UI by navigating to `http://localhost:9090`. Here, you can define custom queries and create dashboards to visualize your application's metrics.
